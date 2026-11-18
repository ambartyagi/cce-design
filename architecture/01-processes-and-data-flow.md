# 01. Processes and data flow

## What runs

One image, several entrypoints (ADR-0001). All state lives in Postgres; the processes keep none.

| Process | Started by | Talks to | Does |
|---|---|---|---|
| `postgres` (+pgvector) | compose | — | All data. Row-level security enforces tenancy (ADR-0010) |
| `migrate` | compose, on every start | postgres (admin role) | Applies Alembic migrations, then exits (ADR-0006) |
| `api` | compose | postgres (app role, RLS), answer model, decision provider, embeddings provider | HTTP API, `/ask`, `/demo` chat UI |
| `worker` | compose | postgres (admin role), embeddings provider | Runs queued jobs: `embed`, and `dependents_fanout` (after a shared rule changes, flag companies' own answers about it) |
| Operator commands | `docker compose run --rm api python -m cce.<x>` | postgres directly | Readers (connectors), sync runs, certify, questions, tenants, packs, corrections, embed backfill |
| `caddy` | prod compose | api | TLS and reverse proxy (ADR-0004) |
| Outside the stack | — | — | Answer model (hosted or Ollama/vLLM), Jev decision provider (optional), embeddings server (Ollama) |

Operator commands do **not** go through the HTTP API. They open their own database session,
but they call the same functions as the API (`store.certify_proposal`, `store.create_object`),
so the same checks apply and the same `change_log` rows are written.

```mermaid
flowchart TB
  subgraph Outside
    EWB[EazeWork backend<br/>integrator key]
    ADM[Company admin / HR<br/>admin key]
    OP[Operator<br/>shell on the server]
    LLM[(Answer model<br/>hosted or local)]
    JEV[(Decision provider<br/>Jev or heuristic)]
    EMB[(Embeddings server<br/>Ollama)]
  end

  subgraph Stack["Docker stack"]
    CADDY[caddy]
    API[api]
    WORKER[worker]
    MIG[migrate]
    CLI["operator commands<br/>(python -m cce.*)"]
    PG[(postgres + pgvector)]
  end

  EWB -- HTTPS + X-CCE-Tenant --> CADDY --> API
  ADM -- HTTPS --> CADDY
  OP --> CLI
  API -- app role, RLS --> PG
  CLI -- app role per tenant --> PG
  WORKER -- admin role --> PG
  MIG -- admin role --> PG
  API -- HTTP --> LLM
  API -- HTTP --> JEV
  API -- query embedding --> EMB
  WORKER -- document embeddings --> EMB
```

## How processes hand work to each other

There is no message broker. Processes communicate through Postgres:

| From → to | Mechanism | Written in the same transaction as the data? |
|---|---|---|
| api / commands → worker | A row in `jobs`; the worker claims it with `SELECT … FOR UPDATE SKIP LOCKED` (ADR-0002) | **Yes.** `create_object` inserts the object and its `embed` job together, so a job can never refer to a row that was rolled back |
| any writer → audit | A row in `change_log` (append-only) | Yes |
| reader → certifier | A row in `proposals` with `status = pending` | Yes |
| certifier → answering | `knowledge_objects.status = active`; the next `/ask` sees it | Yes, nothing is cached except the platform tenant list (30 s) |
| `/ask` → admin | A row in `decision_traces`, and in `knowledge_gaps` when refused | Yes |

The worker polls every 2 seconds when idle. A failed job is retried after 30 seconds, up to 3
attempts, then marked `failed` with the error kept.

## Flow 1: a configuration export becomes knowledge

Each import is a **sync run** (ADR-0022, [04 Change paths](04-change-paths.md) B).

```mermaid
sequenceDiagram
  autonumber
  actor Op as Operator
  participant R as eazework_export reader
  participant DB as postgres
  actor HR as HR (Questions for HR)
  participant W as worker

  Op->>R: ./cce.sh ingest Company data/exports/…
  R->>DB: open a sync run (file fingerprint, reader version)
  alt same file and reader as the last applied run
    R->>DB: run applied: no change
  else
    R->>R: read widget .xlsx files, render each row<br/>as a plain-language rule (no model)
    R->>DB: proposals by source key: create / supersede,<br/>removals and new ambiguities as questions
    alt first import, reader changed, unusually large
      R->>DB: run held
      Op->>DB: cce.sync apply (or discard)
    end
    DB->>DB: certify new and changed rules: new version,<br/>old one superseded, edges, change_log, embed job
    DB->>DB: HR answers about a changed rule: paused,<br/>re-confirm question raised
  end
  HR->>DB: answer questions: keep / retire / new wording
  W->>DB: claim embed job, store vector + model name
```

Tables written: `source_snapshots`, `proposals`, `knowledge_objects`, `dependency_edges`, `change_log`, `jobs`.
The employee profile report is never read by this reader (ADR-0017).

## Flow 2: an employee asks a question

```mermaid
sequenceDiagram
  autonumber
  actor U as Employee (via EazeWork)
  participant API as api /ask
  participant C as compiler
  participant D as decision gates
  participant M as answer model
  participant DB as postgres

  U->>API: question, history, employee code or profile (transient)
  API->>M: rewrite a follow-up into a standalone question (only when needed)
  API->>C: compile context
  C->>DB: hybrid search (words + vectors of the current model)
  C->>DB: entity matches, scope, one-hop expansion
  C->>C: drop rules whose eligibility excludes this asker<br/>order: overrides/clarifications, own before platform,<br/>human before ingested, newer version first
  C-->>API: context blocks + attribution (id, version)
  API->>D: what kind of question? (knowledge / personal data / speculative)
  API->>D: does the context cover it? (grounded probability)
  alt not covered or not a knowledge question
    API->>DB: decision_trace (refused) + knowledge_gap
  else covered
    API->>M: answer from the blocks (fast model, or careful model<br/>when an override or history is involved)
    API->>DB: decision_trace (answer, model, objects + versions)
  end
  API-->>U: answer, citations, trace id
```

The asker's profile is used to filter rules and is **not stored** and **not sent to the model**
(ADR-0018).

## Flow 3: a person changes knowledge

| Who | How | Result |
|---|---|---|
| HR answers a question for HR | `POST /questions/{id}/answer`, `make answer` | A certified `Clarification` that `interprets` the rules the question was about |
| Admin corrects an answer: rule wrong | `POST /traces/{id}/correction` cause `rule` | A `Clarification` proposal, certified at once unless `CCE_CORRECTIONS_REQUIRE_APPROVAL=true` |
| Admin corrects an answer: answer misread | same, cause `misread` | Knowledge unchanged; a test case for model and prompt changes |
| Admin corrects an answer: HR data wrong | same, cause `data` | Knowledge unchanged; an item in Questions for HR; the fix belongs in EazeWork |
| Anyone with a key | `POST /proposals` then `/certify` | Any create / supersede / deprecate, risk auto-scored |

## Flow 4: EazeWork help-center content

The same as flow 1, with the `documents` reader and the **platform tenant** (ADR-0015). Source
keys are `helpcenter:<slug>[#section]`; unchanged articles (same content hash) are skipped.
Help-center runs are always held for a person; a folder is a full release, so missing
articles become questions. When a shared article changes, the worker flags companies' own
overrides of it in their Questions for HR.
Every company reads platform objects; no company can change them, only interpret them with a
`TenantOverride`.

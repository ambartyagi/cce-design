# 0016. Local embeddings (qwen3-embedding:0.6b, 1024 dims) and hybrid retrieval

Date: 2026-10-09
Status: **Accepted — implemented** (migration 0006); calibrated 2026-10-10 on the laptop (LO 0.47 / HI 0.55 / min 0.43), golden set 26/26 with embeddings

## Context

Retrieval and the grounded-or-refuse gate were purely lexical (stems, title weighting).
That works when the asker uses the policy's words and fails when they don't:
H1-02 ("Do I need a doctor's note…?" vs a rule that says "medical certificate")
retrieves the right object but the word-overlap gate refuses to answer.
A help center of ~100 chunks written in product vocabulary makes the gap wider.

Constraints: runs on the 48 GB M4 Pro next to a 27B chat model; no customer text
leaves the machine by default; CI has no model.

## Decision

1. **Model:** `qwen3-embedding:0.6b` served by the existing Ollama (`/api/embed`).
   Small (~0.6 GB), multilingual (Hindi/Hinglish questions are realistic), strong on
   MTEB for its size, instruction-aware queries. OpenAI-compatible endpoints are
   supported by the same interface for a later comparison; not the default.
2. **Column is `vector(1024)`** (migration 0006 nulls old vectors). Larger Matryoshka
   outputs are truncated and renormalised; smaller models are zero-padded. Each row
   records `embedding_model = "<provider>:<model>@1024"` (ADR-0009); search only
   compares vectors from the current model, so a model switch is `make embed-backfill`,
   never a silent mix.
3. **Hybrid retrieval:** lexical and vector rankings fused by Reciprocal Rank Fusion
   (k=60), separately for the policy and how-to budgets. Vector hits below
   `CCE_EMBEDDINGS_MIN_SIMILARITY` are dropped. Exact scan, no ANN index: below ~50k
   rows it is faster and exact; revisit then.
4. **Semantic grounding:** the heuristic grounded gate takes
   `max(word_coverage, clamp((max_sim − LO)/(HI − LO)))`. It can only *rescue* an
   answer the words missed, never sink one the words support. Typed (Jev) providers
   are unchanged.
5. **Never on the critical path:** if the embedder is down, `query_vector` returns
   None and the request runs lexical-only; `/ask` reports `retrieval: lexical|hybrid`,
   `embedding_model`, `max_similarity` and the grounding breakdown.
6. **CI stays lexical** (`CCE_EMBEDDINGS_PROVIDER=none`). A deterministic hashing
   provider exists as a test double only.
7. **Thresholds are data, not guesses:** `make calibrate` measures question↔object
   similarity on the golden set and help-center probes and suggests
   `MIN_SIMILARITY`, `GROUNDED_SIMILARITY_LO/HI`. Defaults (0.30 / 0.35 / 0.60) are
   placeholders until it has run on the real model.

## Consequences

- One more moving part locally (an embedding model in Ollama), none in CI.
- Writes enqueue an `embed` job; the harness embeds synchronously before a run.
- Evaluation now has to be read per retrieval mode; `latest.json` records the model.
- The config-first HRMS decision moves to ADR-0017.

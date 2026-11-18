# Evals

> *"A certified correction must never regress."* — Design Review G3
>
> This directory is where that invariant becomes a checkable thing rather than an intention.

## What's here

The fixtures moved into the HRMS domain pack on 2026-10-08, when the pack loader landed (P1-3):

| File | What it is |
|---|---|
| [`domains/hrms/evals/golden-hrms-v1.yaml`](../../domains/hrms/evals/golden-hrms-v1.yaml) | 26 golden cases for the HRMS `policy_qa` agent (UC-H1/H2) |
| [`domains/hrms/seeds/seed-hrms-v1.yaml`](../../domains/hrms/seeds/seed-hrms-v1.yaml) | The knowledge corpus the questions are answered against — 5 shared statutory objects, 2 tenants, ~20 objects |

Both are **drafts pending review** (P1-1c on the private task board). This directory
keeps only this README.

## ⚠️ The fixture data is illustrative

The statutory rows in the seed corpus (EPF rates, gratuity formulas, maternity entitlements,
UAE end-of-service) are written to be *plausible*, so that the eval exercises real precedence
and effective-dating machinery. **They have not been verified against current law.** They are
test fixtures, not an authority, and the two tenants are invented. Verify before any of this
stands in for a real domain — and when it does, the verification itself becomes an ADR.

## Why a seed corpus ships with the questions

Golden questions without a corpus can't be scored, and a corpus assembled after the questions
gets unconsciously fitted to them. Writing them together, with the corpus designed around
specific *failure modes*, is what makes the set diagnostic instead of decorative. Every fixture
in the seed file exists because some case depends on it:

- **two tenants** — so H1-01 and H1-04 are the same question with different right answers
- **two entities inside one tenant** — so scoping has to be finer than tenancy
- **a superseded version pair** (`notice-managers`, `wfh`) — so time travel and
  "don't retrieve superseded versions by default" are both testable
- **a tenant override of a shared statutory rule** (`pf-above-ceiling`) — so precedence is
  tested, not assumed
- **two grade-scoped rules that both mention buy-out, with different approvers** — so a
  retrieval miss produces a *confidently wrong* answer rather than an obviously empty one
- **a counter-intuitive definition clause** (entity transfer breaks continuous service) — so
  we can tell "the model read the context" from "the model guessed and got lucky"
- **a confidential precedent** — so precedents are shown as precedents, and sensitivity has
  something to bite on
- **three topics with no covering object** — so grounded-or-refuse has to actually refuse

## Scoring: two numbers, never one

When a case fails you need to know *which half* broke:

- **retrieval score** — did every `expect_retrieval` object make it into the compiled context?
- **answer score** — did the answer contain every `expect_answer_contains` and none of the
  `expect_answer_excludes`?

Retrieval fail + answer pass is not a win. It means the model answered from priors and happened
to be right — which is precisely the thing this platform exists to replace. Flag it loudly.

## The regression case

`H1-24` is the exit gate for Phase 1 in a single case. The harness:

1. asks how much internet reimbursement can be claimed → asserts **1,500**
2. files a `supersede` proposal raising it to 2,000, and certifies it
3. re-asks → asserts **2,000**, and asserts 1,500 no longer appears
4. re-asks **3 more times**, and on every run of the suite from here to the end of the project

That last step is the whole point. A correction that holds once is a feature; a correction that
holds forever, automatically checked, is the thesis.

## Adding cases

**Cases are only ever added.** The rule (P1-6e): every time a certified correction is made in
any environment, a case gets appended asserting the corrected answer. Bump the file's `version`
when the schema changes, never to prune cases.

Keep the `coverage` map at the bottom of the golden file honest — if a new category grows past
~4 cases while `refuse` still has 3, the set is drifting toward what's easy to write rather than
what's likely to break.

## Running it

```bash
make eval                                        # inside the compose stack
python scripts/eval_harness.py --pack domains/hrms --only H1-24 -v   # one case, verbose
```

Status 2026-10-08: **25/26 pass, 1 expected failure** (`H1-02`, marked `xfail` in the
golden file: the heuristic decision provider can't map "doctor's note" to "medical
certificate" — it needs a semantic provider). With the `echo` LLM provider the
`expect_answer_excludes` checks are reported but not scored, because echo returns the
compiled context verbatim; they become live the day a synthesizing model is routed.

The bare-model baseline arm (`P1-7e`) also waits for a real model: with `echo`, "no
context" is an empty answer, not a comparison.

The harness is wired into `make demo` and the CI workflow as **one** check, not two — per
IMPLEMENTATION_NOTES §2 (private repo), a separately-remembered check is a check
that stops getting run.

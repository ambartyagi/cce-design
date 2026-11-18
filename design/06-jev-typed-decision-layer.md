# Design Addendum — Typed Decision Models (Jev) in the Governance Layer

**Version:** 0.1
**Date:** 2026-10-05 (status updated 2026-10-07)
**Status:** Accepted and implemented (ADR-0014; `cce.decide`, migration 0003,
2026-10-08). A first implementation on 2026-10-05 was lost with an unpushed
sandbox; the rebuild sits on top of the entity layer (0002).
**Prompted by:** Ambar asking whether new products (specifically Jev, from TypeSafe AI) change the CCE design.

---

## 1. What Jev Is

Jev is a model released by TypeSafe AI (mid-September 2026), positioned as a
**"System One" model** — fast, narrow, parallel decision-making — as a
companion to LLMs ("System Two" — sequential, general-purpose text
generation), not a replacement for them.

Instead of generating free text, Jev takes unstructured context plus a set of
**typed questions** and returns structured, calibrated answers in one pass:

- **Choice** — select among predefined options → probability per option
- **Score** — rate against ordered levels → continuous score + distribution
- **Noul** — yes/no → calibrated probability that a statement is true

It is trained via reinforcement learning for calibration (honest probability
estimates), not human preference — the opposite optimization target from a
chat model. TypeSafe's claim is that invalid/undefined output categories are
*mathematically impossible* (schema-bounded), not just empirically rare, and
that stated confidence correlates with actual accuracy. Early enterprise
adopters report large cost reductions (10–20x in LangChain's writeup) versus
using a general LLM for the same narrow classification/routing task, since
there's no token-by-token generation for a decision that's really just "which
of these N things is this."

## 2. Where It Touches CCE — This Is Not a Tangential Fit

The CCE design has several mechanisms that were specified as "a function of
signals" or "an LLM judgment call" because nothing better existed at design
time. Re-reading the design series with Jev in mind, most of them are exactly
the shape of problem Jev is built for:

| CCE mechanism | Where it's specified | Plan before Jev | Jev fit | Status |
|---|---|---|---|---|
| Risk classification for proposals | Design Doc open question #5; Review §G9 | Hand-rolled scoring function | **Score** over low/med/high with a real probability | **Implemented** (`/proposals`, `risk_probability`, `risk_source`) |
| Grounded-or-refuse gate | Reporting doc §2.2, §4 | LLM self-reports grounding — unreliable | **Noul** pre-check gating generation | **Implemented** (`/ask`, `grounded_probability`, `refused`) |
| Auto-materialize decision | Review §G9; Gathr posture | Risk-rubric threshold | Threshold on stored `risk_probability` | Phase 2 |
| Conflict detection | Ontology doc, Stage 2 | Scope/time overlap query → human | **Noul** precision filter after SQL recall | Phase 2 — SQL recall gets real inputs from 0002's scope edges |
| Ingestion classification (type, sensitivity) | Use Cases doc UC-H6; Review §G5 | Unspecified | **Choice** → populates `KnowledgeObject.confidence` | Phase 2 |
| UC-H0 generated-statement confidence | Design doc 07 §2.4 | — | **Noul** per generated policy statement, sorts the certification batch | Phase 1, after 0003 |

The common thread: every one of these is a **narrow, schema-bounded decision**
made far more often than the system generates a user-facing answer. A
proposal gets risk-scored once; an answer gets grounded-checked on every
`/ask`. That call-volume asymmetry is exactly where Jev's cost/latency
advantage matters most for a single-operator Mac Mini deployment.

## 3. The Architecture Change (now in code)

`cce.decide` is the **second capability class**, parallel to generation:

| Capability | Interface | Provider |
|---|---|---|
| Generation (existing) | `complete()` / `answer(question, blocks) → text` | echo / Anthropic / exo |
| Embeddings (existing, provider pending) | `embed(texts) → vectors` | none yet |
| **Typed decisioning (new)** | `decide(context, question) → Decision(value, probability, source)` | **heuristic (default) / Jev** |

Routing rule: anything that is a governance *gate* calls `decide()`, never
`complete()`. This keeps "numbers from queries, words from models" (the
Reporting doc's rule for report figures) consistent with a sibling rule for
governance: **gates from calibrated typed decisions, narrative from
generation.**

`decide_safely()` wraps every gate call — a gate must never hard-fail the
request it's gating; on any error it returns `None` and the caller proceeds
ungated, exactly as Phase 0 behaved. Every decision's probability and source
are recorded on the row it gated (`proposals`, `decision_traces`), so the
attribution contract extends to the new provider type with no new tables.

## 4. What Does Not Change

The core thesis, the two-layer knowledge model (ADR-0013), the proposal →
certify → materialize loop, RLS tenancy, and the domain-pack configuration
model are all unaffected. Jev is a **governance-layer refinement** — it makes
several previously hand-waved mechanisms concrete and auditable instead of
changing what the platform is for. A genuinely new category of model showed
up and the design absorbed it as a provider-abstraction slot rather than
needing a rearchitecture — which is the entire point of having isolated the
LLM behind `cce.llm` in Phase 0.

## 5. Caveats

- The `HeuristicDecisionProvider`'s probabilities are word-overlap arithmetic
  with the right interface — **not calibrated**. They make the gates
  functional, not trustworthy. Swap to Jev before relying on thresholds.
- `JevDecisionProvider`'s endpoint/payload shape follows TypeSafe's published
  integration guide as of 2026-10 (`model`, `state`, `questions{type,
  instructions}`; responses with `noul` / `score`+`confidence` /
  `probabilities`). Verify against current docs when wiring a real key.
- Jev access (hosted vs. self-hosted) and its cost at gate call volume are
  undecided — see STATUS.md.

---

## Sources

- [TypeSafe AI Releases Jev: a Decision-Only Model That Returns Typed Probabilities Instead of Text — InfoQ](https://www.infoq.com/news/2026/10/typesafe-ai-jev-released/)
- [What Is Jev AI? TypeSafe's System One Model, and Where It Fits in an Agent Harness — width.ai](https://www.width.ai/post/what-is-jev-ai-typesafe)
- [Jev is rapidly rising to challenge the LLM for enterprise AI supremacy — The Register](https://www.theregister.com/ai-and-ml/2026/09/28/jev-is-rapidly-rising-to-challenge-the-llm-for-enterprise-ai-supremacy/5299260)
- [How to Use Jev: A practical guide to TypeSafe's System One model — LangChain blog](https://www.langchain.com/blog/building-a-harness-with-jev)

*Companion to the CCE design series.*

# 0014. Typed decision models (Jev) for governance gates

Date: 2026-10-05
Status: **Accepted — implemented 2026-10-08** as `cce.decide` + migration 0003.
(A first implementation on 2026-10-05 was lost with an unpushed sandbox; this is
the rebuild.) `decide()`, `HeuristicDecisionProvider`, `JevDecisionProvider`;
risk auto-scoring on `/proposals`, grounded-or-refuse on `/ask`; 6 gate checks
in the walking skeleton. See `design/06-jev-typed-decision-layer.md` for the
research this is based on.

## Context

Several governance mechanisms in the design series were specified as "a
function of signals" or "an LLM judgment call" because nothing better existed
at design time:

- The risk rubric for proposal certification (original Design Doc, open
  question #5; Design Review §G9: `risk = f(impact tags, scope, dependent
  count, provenance class)`), left as a hand-rolled heuristic.
- The grounded-or-refuse gate (Reporting & Agent Archetypes doc §2.2, §4),
  left as the generation LLM self-reporting whether it has enough grounding —
  a notoriously unreliable thing to ask a text generator to judge about
  itself.

TypeSafe AI's Jev (released September 2026) is purpose-built for exactly this
shape of problem: a narrow, schema-bounded decision (Choice / Score / Noul)
returned as a genuine calibrated probability, not a string.

## Decision

Extend the `cce.llm` provider abstraction with a second capability class,
`decide()`, parallel to the existing generation path:

```
decide(context: str, question: {type, instructions, ...}) -> Decision(value, probability, source)
```

Routing rule: **gates call `decide()`, answers call the generation
provider.** Narrative from generation, decisions from calibrated typed
decisions — the same discipline as "numbers from queries, words from models"
for reports, extended to a third case.

Two providers ship:

- **`HeuristicDecisionProvider`** (default, `CCE_DECISION_PROVIDER=heuristic`)
  — zero-dependency word-overlap/signal-count placeholder, same philosophy as
  the `echo` generation provider: it exists so every gate has something to
  call with no external setup, not as a claim of calibration.
- **`JevDecisionProvider`** (`CCE_DECISION_PROVIDER=jev` + `CCE_JEV_API_KEY`)
  — real calibrated decisions once TypeSafe access is set up.

Both return the same `Decision` shape so callers never branch on provider.
`decide_safely()` wraps every call: **a governance gate must never hard-fail
the request it's gating** — any error returns `None`, and callers treat
`None` as "ungated, proceed exactly as before this ADR existed."

Two gates are wired:

1. **Risk auto-scoring** (`POST /proposals`): when `risk` isn't given
   explicitly, a Score question scores rationale + change content against
   low/med/high. Recorded as `risk`, `risk_probability`, `risk_source`
   (`explicit` | `heuristic` | `jev`).
2. **Grounded-or-refuse** (`POST /ask`): a Noul question asks whether the
   compiled context covers the user's question, *before* the generation
   provider is called. Below `CCE_GROUNDED_REFUSE_THRESHOLD` (default 0.35),
   the request short-circuits to a refusal. Recorded as
   `grounded_probability`, `refused` on `decision_traces`.

## Alternatives considered

- **Leave both gates as hand-rolled heuristics indefinitely** — rejected:
  always a placeholder; Jev is the first purpose-built replacement.
- **Let the generation LLM self-report confidence** — rejected: the exact
  failure mode Jev's calibration framing targets, and strictly more expensive
  (pay for generation before knowing it should have been a refusal).
- **Wait for Jev to mature before building the abstraction** — rejected: the
  `decide()` interface and heuristic fallback are valuable on their own
  (auditable, typed, same attribution contract), as `echo` was for generation.

## Consequences

**Easier:** both gates return a recorded probability instead of an implicit
guess; swapping in Jev is a config change; the call-volume asymmetry (gates
fire far more often than generation) is where Jev's cost advantage lands.

**Deferred:** the heuristic's probabilities are not calibrated — don't trust
thresholds on them. Conflict detection (Ontology doc Stage 2) and ingestion
classification (UC-H6) are further `decide()` sites, Phase 2.

**Unchanged:** two-layer knowledge model (ADR-0013), proposal loop, RLS
tenancy (ADR-0010), domain packs (ADR-0012). A governance-layer refinement,
not an architecture change — the payoff of isolating the LLM behind `cce.llm`.

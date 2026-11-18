# 0017. Configuration first: an HRMS tenant's knowledge starts from its EazeWork setup

Date: 2026-10-10
Status: **Accepted — implemented** (`cce.connectors.eazework_export`; CI: `scripts/check_eazework.py`)

## Context

Doc 07 §3 left open whether a new HRMS company's knowledge should start from its policy
documents or from its product configuration. The first real export (design doc 08,
"Widget": 11 `.xlsx` widget reports) settled the evidence question: the configuration is
structured, complete for the rules employees ask about most (leave, attendance, late
coming, WFH, shifts, holidays, notice/probation by policy), and identical in shape across
EazeWork customers. Policy documents are prose, vary per company, and are often out of date
relative to what the product actually enforces.

## Decision

1. **Configuration is the first source.** One reader per EazeWork widget turns rows into
   knowledge proposals rendered deterministically in plain language (no model in the loop):
   `PolicyRule` per active leave type, per attendance section and per late-coming policy
   (with its rule table and deduction priority); `Definition` per shift, weekly-off pattern
   and holiday calendar; notice and probation policies inferred from the employee mapping
   when no master is exported. Source keys `eazework:<area>:<name>` make a re-export a diff.
2. **Everything goes through certification** like any other connector. Draft/Closed leave
   types are not loaded.
3. **Ambiguities become questions for HR**, not silent guesses: undefined or unused masters,
   missing approvers, copied eligibility, allowance vs rule thresholds, uncovered or
   overlapping late-coming situations (found by probing the rule table), formula values not
   in the export, policy names with no master. Each is a `ConfigurationGap` proposal whose
   title is the question (`make questions`, `make questions-md`, `GET /questions`).
   **Answering** one (`make answer`, `POST /questions/{id}/answer`) certifies a
   `Clarification` object (provenance human, ranked with overrides) that `interprets` the
   rules it was about. Employee questions nobody could answer (knowledge gaps) share the
   same inbox. Bulk approval never touches questions.
4. **Documents remain the second source** (`connectors/documents.py`) for what
   configuration cannot hold: reasons, process notes, exceptions.
5. **The employee profile report is never read by the connector.** Mapping reports are read
   only for policy names and counts.

## Consequences

- A company is answerable within minutes of an export, before anyone writes a document.
- Knowledge is only as right as the configuration; the questions list is how wrong or
  ambiguous configuration surfaces (17 questions for Widget, several high-severity).
- Widget layout changes in EazeWork need parser changes; CI builds a synthetic export in
  the same layout to catch regressions.

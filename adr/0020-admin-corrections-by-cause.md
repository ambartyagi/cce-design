# 0020. Admin corrections of answers, recorded by cause

Date: 2026-10-10
Status: **Accepted — implemented** (migration 0008; `cce.corrections`; demo UI "Correct this answer")

## Context

An admin who spots a wrong answer needs a way to fix it. "Wrong" has three different
causes, and each needs a different fix. Treating all of them as "edit the knowledge"
would hide model mistakes behind knowledge patches and would let an admin paper over
wrong HR data.

## Decision

`POST /traces/{trace_id}/correction` (company **admin** keys only; integrator keys acting
for employees get 403). The admin picks the cause:

| Cause | What happens |
|---|---|
| **rule** — the rule is wrong or missing | A `Clarification` proposal ("Question / Correct answer") that `interprets` the rules the answer used, for *everyone*, *the same people as one of those rules* (inherits its eligibility), or a custom expression. Certified at once by default; with `CCE_CORRECTIONS_REQUIRE_APPROVAL=true` it waits for a second admin (`make queue`). Clarifications rank first. |
| **misread** — the rule is right, the answer misread it | Knowledge is unchanged. Saved as a test case (question, employee code, expected answer, must-mention terms); `python -m cce.corrections export` writes question-set YAML for `make ask-set`. This is the regression set for model/prompt changes. |
| **data** — the asker's HR details look wrong | Knowledge is unchanged. Appears in Questions for HR; answering it records HR's reply and closes it. The fix belongs in EazeWork. |

Every correction is an `answer_feedback` row tied to the answer's decision trace (who,
when, what was said, what should have been said, which rules were cited). The employee
*code* is kept so a case can be replayed; no profile values are stored.

Clarifications no longer route to the careful model: they are direct Q/A the fast
model follows; the careful route stays for `TenantOverride` of a different rule.

## Consequences

- Demo default (immediate certification) is not a production default: turn on approval
  before a real company uses it.
- Misread corrections are the most valuable data for choosing between models.

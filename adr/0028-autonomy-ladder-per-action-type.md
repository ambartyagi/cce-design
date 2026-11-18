# 0028. Bot autonomy is a per-action-type level, earned by record and lost automatically

Date: 2026-10-16
Status: **Proposed** (design: [design/10-bot-operating-model.md](../design/10-bot-operating-model.md) §5)

## Context

"Minimal human support" is only safe if bots take on more only as they prove themselves, and
lose it the moment they fail. The design review (G9) found that bulk approval trains certifiers
to rubber-stamp; bots generating proposals would make that far worse. Some actions cannot be
undone and must never be automated regardless of record.

## Decision

1. **Autonomy is a level per action type, not per bot:** L0 observe (shadow), L1 propose with a
   human certifying, L2 act behind a calibrated `decide()` gate with undo and a digest, L3 act and
   notify on anomaly. The level is a setting (doc 09 §3) with history and platform locks.
2. **Only reversible actions may reach L2 or L3.** Irreversible actions are capped at L1; those
   that reach every company or commit the company to a person need two principals.
3. **Probation:** every new bot and every new action type starts at L0 for at least two weeks.
4. **Promotion is earned**, by the spec's declared record (e.g. 50 consecutive approvals without
   edit) or by a certified `OperatingRule` generalised from consistent decisions. A promotion is a
   proposal a human certifies.
5. **Demotion is automatic and immediate** on the first rollback of one of the action's runs, a
   client flag, a budget breach, or an approval-rate collapse. Levels are re-earned, never
   re-granted by hand.
6. **Budgets are part of the spec**: human minutes and tokens per tenant-month. Exceeding either
   is a demotion trigger.
7. **Circuit breaker** per bot, per tenant and global, as a setting, so a bot can be sent to L0
   without a deploy.
8. **The governing metric is human minutes per company per month**, which must fall month over
   month; it is computed from decision records, not self-reported.

## Alternatives considered

- **Autonomy per bot.** Rejected: a bot that is reliable at wording-only diffs and unreliable at
  removals would be held at the level of its worst action.
- **Autonomy granted by configuration up front.** Rejected: no record, no calibration, and the
  first failure happens at full autonomy.
- **Demotion by human review only.** Rejected: the failure may repeat many times before the
  weekly review.

## Consequences

- Promotion/demotion logic lives in the Ops instance's weekly review job and in a trigger on
  bot-run rollback; both write settings changes with provenance.
- Clients and the platform can cap levels, so a partner-hosted deployment can run every
  in-product bot at L1 if it chooses.
- Shadow mode requires each action type to declare what "agreement with the human" means in its
  eval set.

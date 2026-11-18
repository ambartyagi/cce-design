# 0018. Rule eligibility, and the asker's profile as transient, consented input

Date: 2026-10-10
Status: **Accepted — implemented** (migration 0007; `cce.eligibility`, `cce.profile`)

## Context

EazeWork rules say who they apply to as expressions (`Job Duration >= 80 Day AND Gender =
Female`, `(Work Location <> Delhi HO) AND …`) or by mapping each employee to a named
policy (notice "Sr Mgmt", shift "Evening Shift"). The same question — "I was 20 minutes
late, what's deducted?" — has a different answer per employee. Scope exclusion (ADR-0015)
only compares entities of one type and can't express this.

Deciding which rules apply needs the asker's HR attributes, which are personal data. The
project rule is `pii: none` in knowledge.

## Decision

1. **Eligibility is stored on the rule** (`knowledge_objects.eligibility`, text) and
   evaluated at compile time against the asker's profile. Evaluation is three-valued; only
   a definite *false* excludes a rule (reported as `excluded: eligibility`), unknown keeps
   it with an "applies to: …" label. Mapped policies are expressed the same way
   (`Shift = Evening Shift`, `Notice Period Policy = Sr Mgmt`).
2. **"Doesn't apply to you" is an answer, not a gap.** When nothing applicable covers the
   question but excluded rules do, `/ask` says which rules exist and who they are for, and
   logs no knowledge gap.
3. **The profile is transient and whitelisted.** It arrives per request — inline
   (`profile`) or fetched by employee code from **EazeWork's consent hook**
   (`CCE_PROFILE_PROVIDER=http`, `employee` + `consent_token`; the hook refuses without the
   employee's consent). Only `cce.profile.WHITELIST` attributes are kept (location, level,
   department, employee type/status, gender for eligibility, dates of joining/confirmation,
   mapped shift/weekly off/calendar/policies); everything else is dropped on arrival and
   only the *names* of dropped keys are reported. Profiles are never written to traces,
   logs, knowledge or embeddings, and are not sent to the model; blocks say "applies to
   you" instead.
4. **Test profiles for demo tenants only:** `make profiles` builds a whitelisted JSON from
   an export's mapping reports into `profiles/` (gitignored); `CCE_PROFILE_PROVIDER=file`.

## Consequences

- Answers become personal without CCE holding personal data.
- EazeWork owns consent and the hook; CCE's contract is: employee code + token in,
  whitelisted attributes out, nothing kept.
- Answers themselves can reflect the profile ("your notice period is 60 days") and traces
  keep answers; trace retention needs a policy before production.

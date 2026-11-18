# 08 — First real EazeWork configuration export ("Widget") and how it fits

*2026-10-10. Input: 11 `.xlsx` widget exports from EazeWork for one company, "Widget".
Not committed to the repo (customer data); kept on Ambar's machine.*

## What arrived

| Group | File | Rows | What it is | Fits CCE as |
|---|---|---|---|---|
| Master | leavesetupdata | 18 leave types × 33 params | type, basis, value, accrual, carry-forward, request limits, eligibility expression, initiator/approver, status | **PolicyRule** per leave type (+ gaps) |
| Master | attendancesetupdata | ~90 param rows | recording, geofencing, late allowance, warning letters, **2 late-coming policies with deduction rule tables**, OD, attendance regularisation, tour, WFH | **PolicyRule** per policy/section |
| Master | shiftdata | 4 shifts | times, grace, breaks | **Definition** per shift |
| Master | weeklyoffdata | 5 patterns | which weekdays, which weeks of month | **Definition** per pattern |
| Master | calendars2026 | 4 calendars | holidays, restricted vs compulsory, one with an eligibility condition | **Definition** per calendar (effective-dated 2026) |
| Employee | rptemployeePolicylist | 47 | notice policy+days, probation policy+months, attendance policy | asker **profile** (scope), and policy names→values |
| Employee | rptShiftDetails / rptWeeklyOffMapping / rptCalendarDetails / rptAttMethodDetails | 47 each | per-employee mapping with periods | asker **profile** (scope) |
| Employee | rptemployeedtl | 47 × 51 cols | full HR profile | **only a whitelisted subset** (see below) |

This is the configuration-first path from doc 07 §3, with a real export format: the
evidence ADR-0017 was waiting for.

## Three ways it fits

1. **Master data → certified policy (UC-H0).** One parser per widget turns rows into
   proposals through the normal certify pipeline: e.g. *Casual Leave: 10 days/year, pro-rata
   from confirmation, quarter-day steps, carry-forward up to 30, may be encashed / adjusted
   against notice, applies to Regular + Working employees, approved by Manager (exceptions up
   to 2 days by HR Manager)*. Closed/Draft types become history or are skipped.
2. **Eligibility expressions → applicability.** Leave and attendance rows carry expressions
   such as `Job Duration >= 80 Day AND Gender = Female`, `Employment Status = On Probation`,
   `(Work Location <> Delhi HO) AND (Work Location <> Greater Kailash) …`. Today scope
   exclusion only compares entities of the same type; this needs a small **eligibility
   evaluator** (attr op value, AND/OR, parentheses) run against the asker's profile at
   compile time. Attributes seen: Work Location, Office Location, Department, Gender,
   Employee Type, Employment Status, Job Duration.
3. **Employee mappings → the asker's profile.** In production EazeWork's backend sends the
   asker's attributes as `scope` (ADR-0015). For testing we derive the same profile from the
   mapping reports by employee code, so "I was 20 minutes late, how much leave is cut?"
   answers differently for a Delhi HO employee (¼ day from Casual Leave, then LOP, if ≥8 h
   worked) and a Chandigarh employee (¼ day from Monthly Prorata, then Probation Leave,
   then LOP). Profiles are **not** knowledge objects and are not embedded.

## Personal data

`rptemployeedtl` includes date of birth, religion, national ID, addresses, father's name,
blood group, mobile and emails. The data looks synthetic (test domains, 9999999999,
mismatched addresses), so it is probably EazeWork's demo tenant — to confirm. Either way
the rule stands (`pii: none`): the loader reads only a **whitelist** — employee code,
level, department, office/work location, employee type, employment status, gender
(used in eligibility), date of joining (→ job duration), shift, weekly off, calendar,
policy names — and never stores names, contact, identity or family fields. For real
customers, ask for the mapping reports plus a stripped profile export instead.

## Configuration gaps the parser should raise (UC-H0 working as intended)

- Weekly off **"Flexi"** is mapped to 2 employees but not defined in the weekly-off master.
- Notice policies **Retainers (75 d)** and **Subcontractors (65 d)** appear only in mappings;
  there is no notice-policy master in the export (only per-employee days).
- **Adoption** and **Commissioning** leave carry the same `Gender = Female` criterion as
  maternity-type leaves — likely copied; confirm intent.
- **Probation Leave** and **Birthday leave** have no approver; **Medical Leave** is Draft with
  no effective date or short code; **Earn Leave** is Closed.
- Late-coming allowance is 5 min, but the Delhi NCR policy starts deducting at 10 min.
- Attendance regularisation is approved by the **CEO** at level 1 — unusual; confirm.
- Calendars "Employee Calendar 2" and "Delhi calendar" have no employees mapped.

## What it does not give us (still asked for)

- Prose policy documents (the reasons, process notes and exceptions that config can't hold).
- Real employee/HR questions for Widget, to score against instead of our own.
- Confirmation that Widget is a demo tenant and may be used for testing.

## Build plan (done 2026-10-10 — ADR-0017, ADR-0018)

1. `connectors/eazework_export.py`: one reader per widget → proposals (+ ConfigurationGap),
   source keys `config:widget:<widget>:<row>` so re-exports diff cleanly.
2. Eligibility: store the parsed expression on the object; evaluator in compilation;
   excluded-by-eligibility reported like scope exclusions.
3. Profiles: `scripts/profiles_from_export.py` → whitelisted JSON outside the repo;
   `make ask … EMP=<code>` fills `scope` from it.
4. Golden set v2 for Widget: ~20 questions generated from config (leave, late coming per
   location, WFH limit, shifts, holidays, notice by policy), answers checked by EazeWork.

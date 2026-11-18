# 0024. People, roles, and EazeWork as the identity provider

Date: 2026-10-15
Status: **Accepted — people, roles and access implemented 2026-10-15** (migration 0010; `cce.access`,
`cce.auth`, `cce.api_admin`; CI: `scripts/check_access.py`). **Amended 2026-10-19:** sign-in through
the gate in front of an instance (Cloudflare Access) is built — `cce.access_sso`, below. Sign-in
through EazeWork itself: still the integrator key + asker profile path (design 09 §7.1).
Design: [design/09](../design/09-roles-plans-and-settings.md) §1

## Context
Today a company has one shared admin key and EazeWork has an integrator key; actions record a
free-text name. EazeWork wants three kinds of users: a module owner at EazeWork, several HR
admins per client company, and employees. Client content belongs to the client.

## Decision
1. **Principals**: a person or service, belonging to one tenant, with one role: `eazework_admin`
   (platform tenant: companies, plans, help-center releases, defaults, staff), `account_manager`
   (platform tenant, assigned to specific client companies, where it acts at HR admin level),
   `hr_admin` and `employee` (client tenant). The operator stays outside the product (server
   token and command line). Revised 2026-10-15: module owner and platform admin merged; account
   manager added; support and HR reviewer dropped.
2. **Permissions, not role checks**: each endpoint requires a named permission (`knowledge.certify`,
   `questions.answer`, `sync.now`, `settings.company.write`, `plan.write`, …); a role is a set of
   permissions. Account managers act on assigned companies only for the permissions listed in design/09 §1.3.
3. **EazeWork is the identity provider**: requests carry a short-lived token signed by EazeWork
   (who, company, role); CCE verifies it and records the person on every action. Until then,
   personal keys per person replace shared keys.
4. **What EazeWork sees**: assigned account managers see and act on a company's rules, HR answers,
   Questions for HR, syncs and settings, recorded as "EazeWork: name". They can't manage the
   company's HR admins or its access settings. **Individual employee questions** are visible to
   EazeWork only under a time-limited grant from an HR admin (ends by itself, revocable, every
   read logged); otherwise counts and topics only. A company can switch account manager access off.

## Consequences
- `decided_by` and similar fields become principal ids; the audit trail names real people.
- Integrator keys keep working for asking on behalf of employees, and gain a verified identity
  once tokens are signed.
- Needs from EazeWork: the token format and signing key; whether SSO (OIDC) is available.

## Implementation notes (2026-10-15)

- `principals` (person, tenant, role, email), personal keys (`api_keys.principal_id`),
  `am_assignments`, `access_grants` (scope `employee_questions`, 1–30 days, revocable). These tables
  are read and written only through the admin engine after a permission check (RLS on, no policy).
- Every endpoint requires a named permission (`cce.access`), resolved per request from the role,
  the tenant acted in (`X-CCE-Tenant`), the assignment, the company's `access.account_managers`
  setting and active grants. Old shared keys keep working: `admin` = HR admin in a company, EazeWork
  admin in the platform tenant; `integrator` = ask, propose, read knowledge.
- The acting person is recorded on every change (`change_log.actor_id`, `detail.actor`, and
  `decided_by`); EazeWork people acting in a company appear as "EazeWork: <name>". The name in a
  request body is used only for old shared keys.
- The employee question log = decision traces and the corrections that quote them. Visible to the
  company's HR admins, to whoever asked (employees see their own answers, `decision_traces.asked_by`),
  and to EazeWork only under a grant; every such read is written to the company's activity log
  (`GET /activity`). Unanswered employee questions in Questions for HR stay visible to assigned
  account managers, because answering them is part of their job.
- An EazeWork admin acting in a company can set it up (first HR admin, EazeWork values and locks)
  but does HR work there only if assigned to it like an account manager.
- Sign-in: personal keys for now. The signed per-user token from EazeWork (design/09 §1.5, option
  1) will be a second way to establish the same person and role; roles and permissions don't change.
- Review fixes (same day, independent review): EazeWork admins not assigned to a company never get
  grant access there; confidential knowledge follows the caller's permissions, not a role in the
  request, and its titles aren't shown to others; employees ask as themselves (a profile or another
  code in the request is ignored); "HR details are wrong" items and the trace link of unanswered
  questions are hidden without a grant, and reads of corrections under a grant are logged;
  `access.*` settings can't be set by EazeWork at all; keys fail closed (missing person, wrong
  tenant, unknown role); the first-HR-admin path counts old shared keys and is serialised; EazeWork
  admins can't assign themselves; EazeWork's actions are always labelled in a company's activity log.
- EazeWork admins add client companies (`POST /companies`), which have no shared key.
- Known limits: any tenant of kind `platform` is treated as EazeWork, but settings use the oldest
  one; keep one platform tenant per installation. The integrator key can't read decision traces
  (EazeWork gets each answer in the `/ask` response). Until sign-in through EazeWork, whoever
  provisions a person sees their first key.


## Amendment (2026-10-19): the gate as identity provider

Hosted instances sit behind Cloudflare Access, which verifies the visitor (email one-time code or
Google) and adds a signed token to every proxied request. `cce.access_sso` verifies the token
(RS256 against the team's published keys; audience = the Access application; issuer = the team
domain) and maps the email to a person. The person then acts as with their personal key: same
role, permissions and name on every change. Rules:

- Off unless `CCE_ACCESS_TEAM_DOMAIN` and `CCE_ACCESS_AUD` are set. Keys keep working and win
  when both are present (the API, scripts, anyone outside the gate).
- An email that is a person in two tenants chooses (`X-CCE-As`), never acts as both.
- An email nobody has is not auto-created; the Tool Administrator adds people. Removed people
  are not identities.
- The token is trusted because the API is reachable only through the tunnel; the signature check
  is defence in depth against a header forged from inside the host.

CI: `scripts/check_access_sso.py`. The browser pages (`/demo`, `/demo/manage`) try the session
first and fall back to the key dialog.

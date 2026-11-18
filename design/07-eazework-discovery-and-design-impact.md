# Cumulative Context Engine – EazeWork Discovery Findings & Design Impact

**Version:** 0.1 (draft, for discussion; migration numbers corrected 2026-10-07)
**Date:** 2026-10-06
**Source:** Answers from EazeWork's founder to the discovery questions (2026-10-06) and the `System_Details.xlsx` sheet (HR Master Data fields; Policy Parameters eligibility matrix).
**Status:** Proposed changes. Nothing here is decided. Items marked **(open)** need discussion before the redesign acts on them.

---

## 1. What we learned

| Finding | Evidence | Why it matters to the design |
|---|---|---|
| Written policies are rare. Clients build policy "on the fly" inside the configuration. | "written down policies are rarity… develop their policies on the fly" | The design's HRMS cold-start (UC-H6: ingest policy documents → extract PolicyRules) assumes documents that mostly don't exist. The **source of truth for tenant policy is the configuration**, not prose. |
| The founder wants written policies *generated from configuration*. | "It would be great if written down policies can be generated based on the configuration" | A new use case, and a near-perfect fit: configuration is exact, scoped and effective-dated, which is what `PolicyRule` + scope edges were designed to hold. |
| Policy eligibility is a matrix: 21 policy types × 27 employee attributes (Company, Department, Designation, Level, Location, Gender, Marital Status, Job Duration, Shift, Work Nature, …). | `Policy Parameters` sheet | These 27 attributes are the **entity types** for the HRMS pack (Ontology doc §5 Stage 1), and the `applies_to` / `scoped_to` edges of migration 0002 (shipped 2026-10-07). The HRMS ontology can be derived from the product rather than invented. |
| Two audiences, two answer paths. End users: attendance policy, payslip, documents → answered by account managers from the help center. HR admins: configuration and "which report" → answered by module owners. | Q1, Q3 | UC-H1 (employee Q&A) has no channel: EazeWork gives employees no chat tool. UC-H7 (response drafting for support) is the real Tier-1 entry point. UC-H5 (configuration assistant) is closer than the roadmap assumed. |
| Volume: 4–5 tickets/customer/month at ~10 min; 50–60/month for new customers. | Q2 | Onboarding is where cost concentrates, and also when configuration gets built. Policy-from-config lands exactly there. Gives the first hard baseline for the eval: **10 minutes per ticket**. |
| Wrong interpretation of a payroll rule → wrong code → wrong tax deduction → penalty. | Q4 | Strongest real-world case for the governance loop: statutory rules as certified, effective-dated objects with impact preview before code changes. HIGH risk; a destination (UC-H8 territory), not a start. |
| Clients ask for cross-module interactive reports; competitors show chatbots and candidate matching; EazeWork is considering a retention index. | Market Qs | Report *catalog* ("which report gives X") is cheap knowledge for the assistant. Cross-module querying and the retention index are separate products, out of CCE's scope for now. |
| Target form: a feature inside EazeWork or inside the help center. Owners: Director Development + Director Product Management. | Future Qs | CCE must expose an API an EazeWork/help-center feature can call. Confirms the modular-monolith + API posture (ADR-0001); no UI of our own needed. |
| HR Master Data holds Aadhaar, PAN, ESI, bank accounts, health, religion, disability. | `HR Master Data` sheet | None of it is needed for the pilot. "No employee-level data" becomes an explicit pack rule, and `employee-context` data product (Use Cases §2.3) is **deferred** until a use case needs it and a privacy review exists. |

---

## 2. Proposed design changes

### 2.1 New use case: UC-H0 · Policy generation from configuration — Tier 1 · Risk MED

Read a tenant's configuration (policy parameters + eligibility matrix), produce plain-language `PolicyRule` objects per policy type and scope ("Probation of 6 months applies to Employee Type = Permanent, Level ≤ L3, Office Location = Pune, effective 2025-04-01"), and submit them as **proposals** for module-owner certification. Certified rules become the citeable source for every later answer.

- **Consumes:** `TenantConfiguration` data product (read-only snapshot of configuration, no employee rows), platform defaults.
- **Produces:** `PolicyRule` proposals with `applies_to` edges to entity nodes; `ConfigurationGap` signals (attribute combinations no policy covers; two policies claiming the same scope) — these feed the conflict detector (Ontology doc Stage 2) with real data from day one.
- **Why MED not LOW:** output is deterministic from exact inputs, so the bar can be high (≥ 90% statements correct), but a wrong generated policy that gets certified becomes a wrong citation everywhere.
- **Compounding test:** passes. Consumes configuration + defaults; produces certified rules, gap signals, and (over time) the module owners' corrections to how configuration should be read, which is the tribal knowledge Q4 says is most expensive to lose.

### 2.2 Re-sequence the HRMS Tier-1 entry point

| Roadmap item | Before | Proposed |
|---|---|---|
| First HRMS agent | UC-H1 employee policy Q&A | **UC-H7 support-response drafting** for account managers, over help-center articles + tickets + UC-H0 rules. Employees have no channel; account managers already answer from the help center, so drafting fits their workflow and module-owner edits are the correction signal. |
| Cold start | UC-H6 document ingestion | **UC-H0 configuration import first, UC-H6 document ingestion second** (help center, any written policies that do exist, past tickets). See §3 — this is a reorder, not a removal. |
| First golden questions | Hand-written from policy docs | **Mined from real tickets** (de-identified). Baseline: 10 min/ticket; metric: share of drafts accepted with minor edits. |

### 2.3 HRMS domain pack: derive the ontology from the product

- **Entity types** (migration 0002 `entities`): the 27 eligibility attributes from the Policy Parameters sheet, starting with those used by most policies (Company, Department, Designation, Division, Level, Employee Type, Employment Status, Office Location, Work Location, Work Nature, Work Status, Gender, Marital Status, Job Duration, Month of Joining, Shift).
- **Object types:** add `ConfigurationGap`; keep `PolicyRule`, `Definition`, `ComplianceRequirement`, `TenantOverride`, `ProcessPlaybook`. Add `ReportCatalogEntry` (cheap, answers "which report do I use").
- **Relationship ontology** (Stage 2): `PolicyRule --applies_to--> <entity type>` with the 21×27 matrix as the validation table — the sheet *is* the relationships.yaml, nearly verbatim.
- **Data products:** add `tenant-configuration` (read-only snapshot, versioned per export, no PII). **Defer** `employee-context` and `payroll-register`.
- **Sharing policy:** configuration is tenant-private. Statutory layer unchanged.
- **Pack rule (open):** `pii: none` — the pack declares that no employee-row data product exists; a future product carrying any HR Master Data field needs its own ADR.

### 2.4 Platform core: what this touches

1. **A connector type for structured configuration** (`connectors/` in the pack): snapshot → entity upserts + `PolicyRule` proposal batch. Spec'd as a pack-owned connector against a platform-owned "structured source → proposals" interface. **(open)** whether this is the same interface as document ingestion with a different parser, or a sibling. Prefer the same interface; if it can't be, that's the ADR-0012 signal that the abstraction is wrong.
2. **Batch certification UX** moves earlier (from Phase 2 to Phase 1): UC-H0 produces dozens of proposals per tenant; certifying them one-by-one via the API is not viable for module owners. Minimum: CLI list/approve-by-filter and a confidence-sorted batch (ADR-0014 `decide()` on each statement is the natural confidence source).
3. **Conflict detector** gets real inputs in Phase 1 instead of Phase 2, because configuration gaps and overlaps are computable in SQL over scope edges. The `decide()` precision filter can still wait.
4. **API surface for an embedded feature:** `/ask` and `/proposals` already exist; add a `/drafts` shape (question + ticket context → draft + citations + confidence) that the help-center feature can call. No UI.
5. **Eval harness:** baseline comparison added to the exit gate — CCE draft vs. the same help-center articles pasted into a general model. If CCE doesn't beat it, the governance layer isn't earning its keep on this use case.

### 2.5 What does *not* change

- Architecture (four layers, modular monolith, Postgres+pgvector, RLS tenancy, proposals pipeline): unchanged. Everything above is pack content plus one connector type and a batch-certify path.
- Two-layer rule (ADR-0013): generated policies are proposals, never auto-certified.
- Gathr and Campaigns packs: unaffected.
- Phase gates: Phase 1's gate wording changes ("golden set green" → "ticket-mined golden set green, and beats the baseline"); Phase 2–4 unchanged.

---

## 3. Open discussion: document ingestion is not dropped

The founder's answers lower the *priority* of document ingestion for the first HRMS pilot; they do not remove the need for it. Points for the discussion:

**For keeping document ingestion in Phase 1 (alongside UC-H0):**
- The help center *is* documents. Track 2 of the pilot (support drafting) needs help-center articles ingested; that is UC-H6-style ingestion by another name.
- Some clients do have written policies, and statutory sources (circulars, acts) are always documents. The payroll-penalty story (Q4) is a document-interpretation failure, which is the case for governed ingestion of statutory text.
- Gathr and Campaigns start from documents (rules, post-mortems), so the ingestion path has to exist for the platform claim regardless of HRMS.
- Doing UC-H0 only risks building a configuration-specific tool and calling it a platform.

**For sequencing configuration first:**
- It is the only source that exists for most tenants' policies.
- Deterministic input means a high, honest bar (≥ 90%) and an eval that doesn't depend on LLM extraction quality.
- It delivers something the founder asked for by name, which buys the pilot its goodwill.

**Proposed resolution (open):** Phase 1 carries both. UC-H0 configuration import lands first (weeks 1–3) because its eval is cleaner; help-center ingestion lands in parallel for Track 2 (weeks 2–6) using the existing document path. Statutory-document ingestion with interpretation objects stays Phase 2+. Decide at the Phase 1 gate which of the two produced the better citations.

Other items to settle before the redesign acts:
- Whether `TenantConfiguration` is a snapshot per export (simple, versioned) or a live read (needs DB/API access we may not get).
- Whether UC-H0's generated text should be one `PolicyRule` per (policy type × scope) or one per policy type with scope as edges only. The former cites better; the latter is fewer objects.
- Where the pilot's "module owner" certification happens: CLI is enough for the pilot; a help-center-embedded review UI is EazeWork's to build if it goes to production.

---

## 4. Immediate implications for STATUS / ROADMAP

- Phase 1 item list gains: `TenantConfiguration` connector + UC-H0 generator; batch-certify CLI; ticket-mined golden set; baseline comparison in the gate.
- Phase 1 "HRMS policy_qa agent (UC-H1)" becomes "HRMS support_drafting agent (UC-H7)"; UC-H1 moves to Phase 3 (needs an employee-facing channel EazeWork doesn't have yet).
- Blocking inputs from EazeWork: customer count and onboarding rate; configuration export mechanism; help-center and ticket exports; data sign-off.

---

*Draft for discussion between Ambar and the redesign session. Supersedes nothing yet; once the open items are settled, fold into the Use Cases doc (§2) and ROADMAP.md, and record the configuration-first decision as ADR-0017.*

# Dalil — Business Gap Analysis (scenarios, gaps, and coverage plan)

> Produced 2026-07-07 from four research streams: (1) full PRD v1.6 + FS v2.0 + DECISIONS.md digest,
> (2) as-built backend inventory (every route, model, stub — 1943 tests green), (3) ZATCA/FATOORA +
> KSA VAT/PDPL/payments regulatory research (sources cited in the research appendix of each finding),
> (4) competitive research on the agency-management category (Productive.io, Scoro, Accelo, Workamajig,
> Function Point, Kantata, Bonsai, Teamwork, Monday + GCC locals Daftra/Qoyod/Wafeq/Zoho).
>
> **How to read this:** §3 proves the happy path works as built. §4 walks every other direction the
> business can take and marks each covered/gap. §5 is the numbered gap register — the work list.
> §6 records the deliberate scope rulings. §7 lists spec-vs-build corrections. The launch PRD
> (`01-PRD-launch.md`) and FS amendments (`02-FS-launch.md`) implement this document.
> Since this analysis was written, the **frontend design gap has closed**: the DXS design system
> (tokens + ten screen designs) is delivered — see `08-UI-DESIGN-BRIEF.md` §4/§13. Frontend work
> for any gap here implements those tokens; the remaining design-adjacent open items are the
> Saudia Sans licence (F) and a standalone logo export.

---

## 1. The business, in one page

Dalil sells subscriptions (SAR, tiered plans with entitlements) to **creative agencies** in KSA/GCC.
Each agency (tenant) uses Dalil to run its whole delivery lifecycle:

**configure** (services catalog, project blueprints with phase dependencies, task boards) →
**sell** (leads through a custom pipeline → convert → quotation with exact-match blueprint →
milestone-grouped payment plan → branded proposal → client accepts in the portal) →
**deliver** (project starts by snapshotting the blueprint; phases unlock along the dependency DAG;
teams work kanban boards; internal review then client approval completes each phase) →
**get paid** (completed milestone auto-generates an invoice; payments recorded against it;
scope changes billed via change requests).

Money flows in two loops: **agency → Dalil** (plan subscription — Dalil is the merchant) and
**client → agency** (project invoices — Dalil is the invoicing system, the agency is the merchant).
Both loops happen in Saudi Arabia, which means both are subject to KSA VAT (15%), ZATCA e-invoicing
law, and PDPL. **This regulatory layer is where the original PRD/FS are silent — it is the largest
gap class found.**

Market context (2026 research): no Arabic-first, ZATCA-native agency-management product exists.
Global all-in-ones (Productive.io ~$9–40/seat, Scoro ~$20–50/seat, Accelo ~$50+) have no RTL/ZATCA;
GCC locals (Daftra, Qoyod, Wafeq) are accounting tools without pipeline/blueprints/portal. The slot
Dalil targets is genuinely open — provided the compliance layer exists, because the incumbents that
DO have it (the accounting tools) are exactly what agencies would otherwise keep using.

## 2. Method

1. Reconstruct the end-to-end happy scenario from PRD §6 and verify each step against the as-built
   API (routes, guards, FSMs, error codes).
2. Branch at every step: what else can happen here? (client says no, money is wrong, someone leaves,
   the law applies…) — 40+ scenario branches across 9 directions.
3. For each branch: **covered** (built + specced), **partial** (specced or built, not both / not
   wired), or **GAP** (neither). Every GAP gets a register entry with severity and a resolution.
4. Cross-check the register against the competitor feature baseline and the KSA regulatory findings
   so business gaps invisible from inside the spec (ZATCA, retainers, time tracking) are caught.

## 3. The happy scenario — verified against the as-built API ✅

Every step below exists, is guarded by the right permission key, and was runtime-proven during the
Phase-6.5 demo walk. (Numbers = the frontend demo script in `05-FRONTEND-START.md` §9.)

1. **Setup** — Admin creates Services (`POST /services`), a Blueprint with dependent phases
   (`POST /blueprints`, DAG-validated, `CIRCULAR_PHASE_DEPENDENCY` guarded), boards + task templates.
2. **Sell** — Lead created (`POST /leads`), moved across the custom pipeline (TransitionLog written),
   converted YES (`PATCH /leads/:id/convert`) → onboarding token issued → client sets password.
3. **Quote** — Builder picks contact + services (XOR package) → `POST /quotations/blueprint-matches`
   (exact set-equality) → milestone grouping validated (dependency-closed, exactly-once, sums ±0.01)
   → proposal generated (versioned) → sent (`quotations.send`).
4. **Negotiate** — Client amends (`POST /proposals/:id/amend`) → agency revises (REVISABLE statuses
   enforced) → resends → client **accepts** → Project provisioned PENDING_START.
5. **Deliver** — `POST /projects/:id/start` (entitlement check, blueprint-deleted guard, snapshot,
   SELECT FOR UPDATE) → phases unlock along the DAG → tasks to Done (terminal column) → phase
   IN_REVIEW → admin approves (append-only Approval, IP captured) → CLIENT_PENDING → client approves
   → COMPLETED → next phases unlock.
6. **Get paid** — last phase of a milestone completes → invoice generated (deduped by milestoneId)
   → notification fans out → partial payment recorded (`OVERPAYMENT` guarded) → derived status
   UNPAID → PARTIALLY_PAID → PAID.
7. **Scope change** — CR raised from the portal → approved (`change_requests.approve`) → second
   invoice auto-generated (XOR source invariant).

**Verdict: the core lifecycle is real, correct, and demo-ready.** The gaps are in the directions the
happy path doesn't take — and in the legal wrapper around step 6.

## 4. Scenario sweep — every other direction

Verdict key: ✅ covered · 🟡 partial · 🔴 GAP (→ register entry).

### 4.1 Sales directions
| Scenario | Verdict |
|---|---|
| Client says NO at conversion → lead STOPPED, archived; later reopens | ✅ (`PATCH /leads/:id`, STOPPED→OPEN) |
| Duplicate contact email; same email across two agencies | ✅ (`DUPLICATE_CONTACT_EMAIL` per tenant; global identity by email) |
| Stage deleted while holding leads | ✅ (cascade to previous stage; `force:true` confirm) |
| Lead assigned to a deactivated user | ✅ (`USER_INACTIVE`) |
| Existing client wants a second project | ✅ (portal `POST /portal/project-requests` → agency quotes) |
| Client never answers a sent proposal — does it expire? | 🔴 **G-15** (no validity window; a stale proposal is acceptable forever) |
| Quotation needs a visible discount (not a silent total override) | 🟡 **G-12** (only `totalOverridden` exists — no gross/discount/net) |
| No blueprint matches the chosen service set | ✅ (`NO_BLUEPRINT_FOUND` fallback + editor link) — strict rule still under design-partner validation (open item E) |
| Lead arrives from the website / WhatsApp automatically | 🔴 deferred deliberately (webhook capture — roadmap Wave 2) |

### 4.2 Money directions (the critical class)
| Scenario | Verdict |
|---|---|
| Milestone completes → invoice; partial payments; overpayment blocked | ✅ |
| **Invoice is a legal KSA tax invoice** (Arabic, sequential number, VAT 15%, TINs, QR) | 🔴 **G-01/G-02/G-03/G-05** — Invoice model has `amount` only; no VAT anywhere in spec or code |
| **Tenant is VAT-registered → ZATCA Phase 2 clearance/reporting** (Wave 24 = SAR 375k already in force since 30 Jun 2026) | 🔴 **G-01** — launch-gating |
| Client is refunded / invoice must be corrected | 🔴 **G-04** — no credit/debit note entity (legally mandatory on adjustment, Art. 54; also matches house append-only pattern) |
| Deposit: 40% on signing, rest on milestones (advance payments must be invoiced when received — Art. 53(1)(a)(2)) | 🟡 **G-11** — engine supports full-upfront (`ON_PROJECT_START`) but not a deposit milestone inside a MILESTONE_BASED plan |
| Client pays online from the invoice (link/mada/Apple Pay) | 🔴 **G-07** — no gateway; **Stripe is NOT available to Saudi entities** (verified) → SAMA-licensed PSP (Moyasar/Tap) |
| Invoice overdue → reminder ladder | 🟡 **G-14** (OVERDUE derived ✓; reminders specced in FS §13 but unwired) |
| Zero-cost scope change (no invoice needed) | 🔴 **G-13** (CR amount must be > 0; approve always invoices) |
| Discount / goodwill credit on an existing invoice | 🔴 folds into **G-04** (credit note) |
| Multi-currency client (UAE) | 🔴 deferred deliberately (SAR-only at launch; zero-rating handled in G-02; FX — roadmap Wave 3) |
| Retainer / monthly recurring engagement | 🔴 deferred deliberately (**the #1 competitive gap** — roadmap Wave 1, see §6) |

### 4.3 Delivery directions
| Scenario | Verdict |
|---|---|
| Phase paused / resumed | ✅ (ON_HOLD toggle, `projects.start` key) |
| **Whole project paused** (client stops paying / goes quiet) | 🟡 **G-10** — only per-phase hold; no project-level hold semantics |
| **Project cancelled mid-flight** | 🔴 **G-09** — `CANCELLED` enum exists, no transition implemented (D-059); no rule for freezing future milestone invoices vs keeping earned amounts billable |
| Client sits on CLIENT_PENDING forever | 🟡 folds into **G-14/G-16** (overdue/nudge automation unwired) |
| Deadline exceeded → overtime alert → OVERTIME change request | 🟡 **G-16** (manual CR path ✅; the hourly overdue scan is specced, unwired) |
| Team member leaves mid-project | ✅ (deactivate; task `assignedTo` SetNull; last-admin guards) |
| Concurrent reviewers race on one phase | ✅ (`PHASE_ALREADY_REVIEWED`, SELECT FOR UPDATE) |
| Blueprint edited/deleted while projects run | ✅ (snapshot isolation; soft delete; `BLUEPRINT_DELETED` at Start) |
| Client uploads/downloads deliverable files | 🟡 **G-24** (schema ready — `attachments[]`, presigned-URL design — S3 adapter stubbed) |

### 4.4 Tenant lifecycle directions (agency ↔ Dalil)
| Scenario | Verdict |
|---|---|
| Provision, suspend, reactivate a tenant | ✅ (platform console) |
| Plan limits enforced (projects/users) | ✅ (`PLAN_LIMIT_EXCEEDED`; downgrade leaves over-limit blocking new creates) |
| **Agency pays Dalil** (checkout, renewal, dunning → suspension) | 🔴 **G-08** — no billing behavior specced beyond "Stripe, Phase 8"; and Stripe can't be the answer in KSA |
| Trial → paid conversion | 🔴 folds into **G-08** (14-day no-card trial is the category norm) |
| **Dalil's own invoices to agencies are ZATCA/VAT-compliant** | 🔴 folds into **G-01/G-08** — Dalil is itself a KSA seller |
| Tenant exports its data / leaves | 🔴 **G-19** (churn safety + PDPL access/portability right) |
| Tenant demands data deletion (PDPL) | 🔴 **G-06/G-19** |

### 4.5 Legal / compliance directions
| Scenario | Verdict |
|---|---|
| PDPL: privacy notice, consent basis, DSR (30-day), breach→SDAIA 72h, ROPA (+5y), DPA with tenants, SDAIA SCCs for Bahrain hosting | 🔴 **G-06** — PDPL appears in the specs only as a hosting-region rationale; enforcement is live (48 SDAIA decisions in 2025), fines to SAR 5M |
| Arabic on generated financial documents (Art. 53(5): mandatory) | 🔴 **G-05** |
| Data residency: AWS Bahrain lawful w/ SCCs; **AWS Riyadh (me-central-2) reportedly GA** → verify and prefer | 🟡 **G-06** action item |
| Sequential, tamper-evident invoice numbering + 6-year archive | 🔴 **G-03** (and archive: G-01) |

### 4.6 Client-portal directions
| Scenario | Verdict |
|---|---|
| Onboarding link expired/reused; magic-link re-entry; optional password | ✅ (token types, TTLs, single-use, rate-limited 3/hr) |
| Client belongs to several agencies | ✅ (identity by email, per-tenant scoping) |
| **15-minute portal session with no refresh** — client is mid-proposal-review and gets logged out | 🟡 **G-17** — FS §2 says client JWT "30-day/configurable"; built as 15 min. Spec-vs-build conflict needing a ruling |
| Archived/cancelled project still viewable read-only | ✅ |

### 4.7 Team/RBAC directions
All four admin-protection invariants, invite/activate (24h token), overrides, empty-group users —
✅ covered and heavily tested. Minor: catalog keys `invoices.create`, `analytics.view`,
`settings.manage`, `notifications.manage`, `contacts.manage` have no routes yet (→ §7 corrections;
`analytics.view` gets routes via **G-20**).

### 4.8 Operations directions
| Scenario | Verdict |
|---|---|
| Notifications for all 16 FS §13 triggers | 🟡 **G-16** — only invoice-generated is wired; preferences endpoint absent |
| Realtime updates (boards, portal) | 🟡 **G-21** (Socket.io stub; poll for demo) |
| Email actually delivered | 🟡 **G-21** (stub) |
| PDFs actually rendered (proposal + invoice) | 🟡 **G-21** (stub) — merges with G-05 (the real PDF must be the bilingual tax document) |
| Analytics dashboards (FS §14 formulas) | 🔴 **G-20** — no endpoints at all |
| Deadline math respects Sunday–Thursday work week (Fri–Sat weekend) | 🔴 **G-23** — business-day logic unspecified; Mon–Fri assumptions would be a defect |
| Hijri dates | ✅ as-is — Gregorian/ISO is correct system-of-record (ZATCA mandates Gregorian); optional dual Umm-al-Qura display is roadmap polish |
| Auth rate limiting (FS: 10/min/IP on `/auth/*`) | 🟡 **G-25** — verify wired; if not, add at deploy |

## 5. Gap register (the work list)

Severity: **L** = launch-blocking (cannot lawfully or credibly launch without) · **P** = pilot-blocking
(needed before real design-partner usage; demo fine without) · **D** = demo-visible polish ·
**R** = deferred to roadmap (recorded in `07-POST-LAUNCH-ROADMAP.md` only).

| # | Gap | Sev | Resolution (summary — full contract in `02-FS-launch.md`) | Owner |
|---|---|---|---|---|
| **G-01** | ZATCA e-invoicing (Phase 1 + Phase 2 clearance/reporting; per-tenant Fatoora onboarding; 6-yr in-KSA archive) | **L** | Compliance-adapter architecture: Dalil generates the fiscal data model; a **certified compliance API provider** (Wafeq / ClearTax / Complyance — pick after quotes) handles UBL 2.1, signing, PIH chain, QR, clearance/reporting, per-tenant device onboarding. Tenant compliance profile: NOT_REGISTERED / PHASE_1 / PHASE_2. Direct FATOORA integration = roadmap option once volume justifies. | backend + business |
| **G-02** | VAT data model (15% per-line configurable; Art. 53 fields: TINs, addresses, ex-VAT unit price, per-line VAT, zero-rated narration; treatment per invoice: STANDARD/ZERO_RATED/EXEMPT/OUT_OF_SCOPE — never auto-zero-rate by country) | **L** | New invoice fiscal layer: `InvoiceLine`, tenant tax settings (TRN, address, VAT-registered flag), quotation→invoice line carry-through. VAT applies on top of existing DECIMAL money handling. | backend |
| **G-03** | Sequential human-readable invoice number + tamper-evident counter | **L** | Per-tenant atomic sequence (`INV-2026-00001`), gapless, assigned at issuance inside the txn; never reused; credit notes share the sequence rules. | backend |
| **G-04** | Credit / debit notes (refunds, corrections, goodwill credits) | **L** | Append-only `CreditNote`/`DebitNote` referencing the original invoice + Art. 40 reason; flows through the same ZATCA pipeline; invoice derived status accounts for credits. No invoice mutation, ever. | backend |
| **G-05** | Arabic-primary bilingual financial documents (invoice/credit-note/proposal PDFs) | **L** | Real PDF adapter renders Arabic-primary + English translation, RTL-correct, QR embedded; replaces the stub at deploy phase. | backend/deploy |
| **G-06** | PDPL compliance pack | **L** | Product: consent + privacy-notice surfaces, DSR export/delete endpoints (30-day SLA), breach runbook (72h → SDAIA platform), ROPA doc, tenant DPA template, SDAIA SCCs. Infra: verify AWS me-central-2 (Riyadh) GA — if live, host there and moot the transfer analysis. Legal counsel: NDGP registration ("main activity" test). | business + backend |
| **G-07** | Client invoice payment collection | **P** | SAMA-licensed PSP (shortlist **Moyasar** or **Tap**; mada + Apple Pay + cards; Tap's WhatsApp/SMS payment links fit the flow). Payment-link on invoice → gateway webhook → append Payment row (idempotent, signature-verified). Manual recording remains for cash/transfer. | backend + business |
| **G-08** | Dalil's own subscription billing (agency→Dalil): checkout, renewal, tokenized recurring, dunning→suspend, trial | **P** | Same PSP, tokenized cards (Visa/MC guaranteed; mada recurring = verify contractually). Billing engine in Dalil (plans exist; add subscription lifecycle + webhook→`subscriptionStatus`). Dalil's own invoices go through the same ZATCA pipeline. 14-day no-card trial. | backend + business |
| **G-09** | Project cancellation semantics | **P** | Implement PENDING_START/ACTIVE→CANCELLED (`projects.archive` key holder): freezes all future milestone triggers, keeps issued invoices payable, cancels queued jobs, portal shows read-only; kill-fee billed as a manual CR if contracted. | backend |
| **G-10** | Project-level hold | **D** | `PATCH /projects/:id/hold|resume` = bulk ON_HOLD across non-terminal phases + suppression of overdue alerts; project remains ACTIVE (status derived display "On hold"). | backend |
| **G-11** | Deposit milestone in a MILESTONE_BASED plan | **P** | Allow the first milestone (order 1, no phases) with `triggerType: ON_PROJECT_START` — invoiced at Start (satisfies Art. 53 advance-payment invoicing). Validation: at most one, must be order 1. | backend |
| **G-12** | Explicit discounts (quotation + invoice display) | **P** | Quotation gets `discountAmount` (+ reason); documents show gross/discount/net; VAT computed on net (Art. 53(5)(f) discounts field). Replaces silent total override for the discount use case (override stays for custom pricing). | backend |
| **G-13** | Zero-amount change request | **D** | Allow `amount: 0`; approval of a zero CR skips invoice generation (status APPROVED terminal instead of INVOICED). | backend |
| **G-14** | Overdue dunning ladder | **P** | BullMQ scheduled scan (already specced FS §13): reminders at due−3d, due, +7d, +14d (email + portal + optional WhatsApp later); tenant-configurable off. | backend/deploy |
| **G-15** | Proposal validity window | **D** | `validUntil` on quotation (default 30d, configurable); client accept/amend after expiry → `422 PROPOSAL_EXPIRED`; agency can reissue. | backend |
| **G-16** | Notification triggers + preferences | **P** | Wire the remaining FS §13 triggers (payment recorded, CR submitted/approved/rejected, phase overdue, proposal events) through the existing NotificationService; add `GET/PATCH /me/notification-preferences`. | backend |
| **G-17** | Client portal session length | **D** | Ruling needed: FS says 30-day configurable, build is 15 min. **Recommendation:** 12-hour client JWT + silent re-issue on activity; keep 15 min only for token-verify step-up. | human ruling → backend |
| **G-19** | Tenant data export & deletion | **P** | `POST /tenants/self/export` (async JSON/CSV bundle) + offboarding deletion procedure (retention carve-out: fiscal documents 6 years under tax law even after PDPL deletion request — documented in the DPA). | backend + business |
| **G-20** | Analytics endpoints (FS §14 formulas exist, no routes) | **D** | `GET /analytics/tenant` (+ platform variant) computing the six specced metrics on read; guards `analytics.view`; feeds the two dashboard screens. | backend |
| **G-21** | Deploy infra (email/SES, S3 presign, real PDF, Redis+BullMQ, Socket.io) | **P** | The planned D-062 deploy pass; adapters exist — swap stubs for real implementations. PDF work merges with G-05. | deploy |
| **G-22** | Platform auth refresh rotation + real JWT secrets in CI/prod | **P** | Banked D-034 — close during deploy pass. | deploy |
| **G-23** | Sunday–Thursday business-day math | **D** | Shared `business-days` helper (Fri/Sat non-working, tenant-configurable weekend) used by deadline defaults + overdue scans; week starts Sunday in all frontend calendars. | backend + frontend |
| **G-24** | File upload/download (feedback attachments, deliverables) | **P** | S3 presign endpoints (schema + FS §10 design already exist); tenant-prefixed paths; part of deploy pass. | deploy |
| **G-25** | Auth rate limiting per FS §16 (10/min/IP) | **P** | Verify what's wired; add throttler guard at deploy if absent. | backend/deploy |

Sequencing: **demo needs none of these** (build the frontend now). **Pilot** (real design partners,
manual billing tolerated): G-09, G-11–G-16, G-21, G-24. **Launch** (public, paid): everything L + P,
with G-01/G-06/G-07/G-08 started earliest because they involve third parties (provider contracts,
counsel, gateway onboarding).

## 6. Deliberate scope rulings (recorded so they're decisions, not oversights)

1. **Time tracking, per-project profitability, resource planning, expenses, retainers → NOT in
   launch.** The competitive research says these define the category baseline — but they are whole
   modules, the design-partner cohort is sold on the delivery+compliance core, and the legal gaps
   (G-01…G-08) are non-negotiable while these are competitive. They are **Wave 1 of the roadmap**,
   scheduled immediately after launch stabilizes. Marketing must not promise them at launch.
2. **Proposal e-acceptance** — already built (authenticated portal Accept + audit trail). E-signature
   upgrade → roadmap.
3. **Multi-currency** — SAR-only at launch; zero-rating for non-GCC/UAE clients handled via G-02
   treatment field (a SAR invoice can be zero-rated). FX documents → roadmap.
4. **Exact-match blueprint rule** stays as specced (open item E) — validate with the first design
   partner before any fallback work.
5. **WhatsApp notifications** — high regional value, not launch-gating → roadmap Wave 2 (design the
   notification service so a channel can be added without touching triggers — already true).
6. **Hijri calendar** — Gregorian system-of-record is correct and ZATCA-mandated; optional dual
   display → roadmap polish.
7. **Self-serve signup** — stays Phase-8/launch-tail as originally planned; pilot tenants are
   assisted-onboarded. Trial mechanics arrive with G-08 billing.

## 7. Corrections (spec/build/doc errors found — "fix all the errors")

1. **Client-JWT TTL conflict** — FS §2 "30-day/configurable" vs built 15 min vs frontend-contract
   doc "15 min". → needs the G-17 ruling; whichever wins, update FS + `../contracts/frontend-contract.md`
   together.
2. **`ProjectStatus.CANCELLED` unreachable** — enum shipped, no transition (D-059 noted the FS gap).
   → G-09 specifies the trigger; FS launch amendment adds the missing contract.
3. **Unused permission keys** (`invoices.create`, `analytics.view`, `settings.manage`,
   `notifications.manage`, `contacts.manage` partial) — `analytics.view` gets routes via G-20;
   `invoices.create` stays reserved (manual invoices deliberately don't exist — system-generated
   only); the rest get owners in the FS amendment or are documented as reserved. Frontend must not
   render dead capabilities.
4. **`agents/state.json` module board stale** — `rbac` shows BLOCKED and several completed modules
   show IN_PROGRESS despite passed gates and `completedAt` timestamps. → reconcile via the state API
   (never hand-edit, D-017).
5. **`../contracts/frontend-contract.md`** says "92 paths" and "cursor pagination being rolled out" —
   refresh after the next backend slice; the fable-5 pack supersedes its narrative sections.
6. **PRD pricing metric** ("MRR via Stripe dashboard") — invalid in KSA (Stripe unavailable);
   corrected in `01-PRD-launch.md` (PSP dashboard / internal MRR analytics).
7. **`session-primer.md` still bundles a stale crm-1 carry-forward** below the standing rules —
   the split into manual vs snapshot is still owed (housekeeping, banked).
8. **Error-code catalog `blockedOn` note** — AUTH_TOKEN_INVALID 401-vs-422 historical conflict is
   resolved in code as 401 (D-020/D-024); clear the stale blocked entry when next touching state.

## 8. New error codes the gap work introduces (extend `error-codes.json`, FS §18 style)

`PROPOSAL_EXPIRED` (422) · `INVOICE_NUMBER_SEQUENCE_CONFLICT` (500-class internal retry) ·
`CREDIT_EXCEEDS_INVOICE` (422) · `TENANT_TAX_PROFILE_INCOMPLETE` (422 — VAT-registered tenant
issuing without TRN/address) · `ZATCA_CLEARANCE_FAILED` (422/502 with provider detail) ·
`PAYMENT_WEBHOOK_INVALID` (401) · `PROJECT_NOT_CANCELLABLE` (422) · `DEPOSIT_MILESTONE_INVALID`
(422). Exact shapes in `02-FS-launch.md`.

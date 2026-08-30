# Dalil — Functional Specification: Launch Amendments (FS-L)

> **Extends FS v2.0** (`agents/context/fs/**`). Nothing here weakens an existing contract; where an
> FS v2.0 rule is changed it is explicitly marked **[AMENDS FS-x.y]**. Every section is written as a
> testable behavioral contract in the house style (entities → endpoints → validation → error codes →
> transactions), ready to be turned into harness `task.txt` slices. Conventions (flat error shape,
> `{ data }` envelope, Zod-first, tenant scoping, DECIMAL(12,2), UTC ISO-8601, append-only ledgers)
> apply throughout and are not restated per section.
> Milestone tags: **[L1]** pilot · **[L2]** public launch (see PRD §4). Untagged = L1.
> UI note: every screen these amendments introduce (tax profile, payment-link page, credit
> notes, billing/trial, preferences, export) is designed and built from the DXS design tokens —
> see `08-UI-DESIGN-BRIEF.md` §4/§13; no new visual language may be invented per feature.

---

## FS-L1 · Tenant Tax Profile

**Entity `TenantTaxProfile`** (1:1 Tenant): `legalNameAr`, `legalNameEn`, `addressLine`, `city`,
`postalCode`, `countryCode` (default `SA`), `vatStatus` enum `NOT_REGISTERED | REGISTERED`,
`trn` (15-digit ZATCA TRN, required iff REGISTERED, validated format `^3\d{13}3$`),
`einvoicingPhase` enum `NONE | PHASE_1 | PHASE_2` (must be `NONE` when NOT_REGISTERED).

- `GET /settings/tax-profile` · `PUT /settings/tax-profile` — **[PK settings.manage]** (this gives
  the reserved key its first route).
- A VAT-registered tenant attempting to issue any invoice with an incomplete profile →
  `422 TENANT_TAX_PROFILE_INCOMPLETE { missing: [...] }`.
- Changing `vatStatus`/`einvoicingPhase` never mutates already-issued documents.

## FS-L2 · Invoice Fiscal Layer

**Entity `InvoiceLine`** (N:1 Invoice, ≥1 per invoice): `description` (2–500), `quantity`
(DECIMAL(10,2) > 0), `unitPriceExVat` (DECIMAL(12,2) ≥ 0), `vatRate` enum-like decimal `15.00 | 0.00`,
`vatAmount` (computed = round(qty × unitPrice × rate/100, 2)), `lineTotalExVat`, `lineTotalIncVat`.

**Invoice additions:** `taxTreatment` enum `STANDARD | ZERO_RATED_EXPORT | EXEMPT | OUT_OF_SCOPE`
(default STANDARD when tenant REGISTERED, OUT_OF_SCOPE when NOT_REGISTERED); `treatmentNarration`
(required, 2–500, iff treatment ≠ STANDARD — the Art. 53(5)(k) narration, e.g. "Zero-rated export of
services under Art. 33"); `subtotalExVat`, `discountAmount` (≥ 0, default 0), `vatTotal`,
`grandTotal` — all stored at issuance (fiscal documents snapshot their math; nothing derived later).
`amount` **[AMENDS FS-11]** becomes an alias of `grandTotal` for backward compatibility of the
payment/overpayment logic.

Rules:
- **Never auto-select ZERO_RATED_EXPORT from client country** — treatment is an explicit choice by
  the issuer (Art. 33 eligibility depends on facts the system cannot infer). UI shows guidance only.
- `vatTotal = Σ line.vatAmount` computed on the **discounted** base when `discountAmount > 0`
  (discount applied pro-rata across lines before VAT; document shows gross / discount / net / VAT /
  total). Tolerance ±0.01 as per house money rules.
- Invoice generation (milestone/CR) builds lines from the quotation milestone (one line per phase
  group or a single line, per the generation contract) — generation contracts in FS-11 otherwise
  unchanged (XOR source, dedup, derived payment status).
- Client identity on the invoice: contact name + (optional) `buyerTrn`, `buyerAddress` captured on
  the Contact (new optional fields) — required for ZATCA standard (B2B) invoices **[L2]**.

## FS-L3 · Sequential Invoice Numbering

- `invoiceNumber` string, unique per tenant, format `INV-{YYYY}-{seq:5}` (e.g. `INV-2026-00042`);
  credit notes `CRN-{YYYY}-{seq:5}`, debit notes `DBN-...`, sharing one per-tenant, per-type,
  per-year gapless sequence.
- Assigned **inside the issuing transaction** from `DocumentSequence { tenantId, docType, year,
  nextValue }` with `SELECT ... FOR UPDATE` — no gaps, no reuse, no reset. Sequence-row contention
  retries once; persistent conflict → `500 INVOICE_NUMBER_SEQUENCE_CONFLICT` (job retry path).
- Existing invoices (pre-migration) are back-numbered once by `createdAt` order in the migration.

## FS-L4 · Credit & Debit Notes

**Entity `CreditNote`** (append-only; `DebitNote` symmetric): `invoiceId` (FK, required),
`noteNumber` (FS-L3), `reason` enum `CANCELLATION | PRICE_ADJUSTMENT | REFUND | GOODWILL |
CORRECTION`, `reasonText` (2–500), lines (same shape as FS-L2, references original invoice content),
totals, `issuedBy`, `issuedAt`. **No UPDATE/DELETE endpoints — ever.**

- `POST /invoices/:id/credit-notes` — **[PK invoices.credit]** (new permission key, category
  Invoices; default groups: Administrator, Finance).
- `GET /invoices/:id/credit-notes`, `GET /credit-notes/:id` — **[PK invoices.view]**; portal
  read-only via Client-JWT.
- Validation: Σ(credit totals) over an invoice ≤ invoice `grandTotal` →
  else `422 CREDIT_EXCEEDS_INVOICE { maximum }`.
- **[AMENDS FS-11 derived status]** effective invoice balance = `grandTotal − Σ credits`; derived
  status uses the effective balance (a fully-credited invoice with 0 payments reads `PAID`-equivalent
  display state `SETTLED`; partially credited behaves as a reduced total). Payments recorded after
  crediting still respect OVERPAYMENT against the effective balance.
- Refund of money actually paid = credit note (reason REFUND) + a negative-direction `Payment` is
  **not** used; instead append `Refund { paymentId?, invoiceId, amount, method, refundedAt }`
  (append-only) so the Payment ledger stays strictly positive. Invoice effective paid =
  `Σ payments − Σ refunds`.
- Credit/debit notes flow through the ZATCA pipeline (FS-L5) exactly like invoices **[L2]**.

## FS-L5 · ZATCA E-Invoicing Compliance **[L2]**

Architecture: `ZatcaComplianceAdapter` interface behind a DI token (house stub-adapter pattern,
D-042/D-062). Launch implementation delegates to the selected certified provider's API (open item G);
a `StubZatcaAdapter` serves dev/demo. Direct FATOORA integration = a future adapter, same interface.

- **Tenant onboarding:** `POST /settings/zatca/onboard { otp }` **[PK settings.manage]** — passes
  the tenant's Fatoora OTP to the provider, stores provider device/CSID references on the tax
  profile. Per-tenant credentials; never shared.
- **Document submission FSM** — new fields on Invoice/CreditNote/DebitNote:
  `zatcaStatus` enum `NOT_APPLICABLE | PENDING | CLEARED | REPORTED | REJECTED | FAILED`,
  `zatcaUuid`, `zatcaQr` (base64 TLV), `submittedAt`, `clearedAt`, `zatcaError?`.
  - Tenant `einvoicingPhase = NONE` → `NOT_APPLICABLE` (document is a normal invoice).
  - `PHASE_1` → generate compliant content + QR locally/via provider; no submission.
  - `PHASE_2` + B2B (buyer has TRN) → **clearance before delivery**: document is created `PENDING`,
    submitted synchronously-with-retry; only on `CLEARED` is it emailed/portal-published (the
    pre-clearance artifact is marked "not a valid tax invoice" and not delivered).
  - `PHASE_2` + B2C/simplified → issue immediately, **report within 24h** (BullMQ job, retry ladder).
  - Provider outage: issuance queues (`PENDING`), retries with backoff, surfaces
    `zatcaStatus` honestly in UI; persistent failure → `FAILED` + notification to Finance +
    `ZATCA_CLEARANCE_FAILED` on manual retry endpoint `POST /invoices/:id/zatca/retry`.
- **Archive:** cleared XML + rendered PDF stored 6 years minimum, tenant-prefixed, delete-protected
  (S3 object lock or equivalent); PDPL deletion requests carve out fiscal documents (FS-L15).
- QR (`zatcaQr`) rendered on every PDF (FS-L6 of PRD: Arabic-primary bilingual document).

## FS-L6 · Client Invoice Payment Links **[L2]**

- `POST /invoices/:id/payment-link` **[PK invoices.view]** → creates/reuses a PSP checkout link
  (mada, Apple Pay, cards) for the **effective outstanding balance**; stored as
  `InvoicePaymentLink { invoiceId, provider, providerRef, url, status, expiresAt }`. Portal invoice
  view shows the link; regenerating after expiry is idempotent-per-outstanding-amount.
- **Webhook** `POST /webhooks/payments/:provider` **[Public + signature verification]** — invalid
  signature → `401 PAYMENT_WEBHOOK_INVALID`. Verified success event → idempotently (by provider
  event id) append `Payment { method: provider, providerRef }`; overpayment impossible by
  construction (link amount = outstanding), race with manual recording guarded by the existing
  OVERPAYMENT check inside the payment transaction.
- Manual payment recording (`invoices.record_payment`) is unchanged and remains the path for bank
  transfer/cash.

## FS-L7 · Tenant Subscription Billing **[L2]**

- New platform-level entities: `Subscription { tenantId, planId, status ENUM(TRIALING, ACTIVE,
  PAST_DUE, SUSPENDED, CANCELLED), currentPeriodStart/End, paymentMethodRef? }` and append-only
  `SubscriptionInvoice` (goes through FS-L2/L3/L5 with **Dalil as the seller** — Dalil eats its own
  compliance dogfood) + `SubscriptionPayment`.
- Lifecycle: provision → `TRIALING` (14 days, no card) → capture payment method (PSP tokenization;
  Visa/MC guaranteed, mada-recurring only if the PSP contract confirms it) → `ACTIVE` → renewal
  charge on period close (BullMQ) → failure → `PAST_DUE` + dunning (retry day 1/3/7, email each) →
  day 14 unpaid → `SUSPENDED` (sets `Tenant.subscriptionStatus = SUSPENDED`, which the existing
  middleware already enforces as 403) → payment → reactivate. Manual override endpoints on the
  platform console for assisted/bank-transfer tenants.
- Plan change: upgrade immediate (prorated charge next cycle — simple credit line, no mid-cycle
  charge); downgrade at period end; over-limit behavior unchanged (blocks new creates only).

## FS-L8 · Project Cancellation & Project Hold

- **Cancel** `PATCH /projects/:id/cancel { reason (2–500) }` — **[PK projects.archive]**.
  Allowed from `PENDING_START | ACTIVE` → `CANCELLED` (fills the D-059 gap; enum already exists).
  Effects, one transaction + `SELECT FOR UPDATE` on Project: all non-terminal phases → terminal
  snapshot state (kept as-is, no new phase status; they simply stop being actionable), **all future
  milestone/one-time invoice triggers permanently disarmed**, queued jobs cancelled, TransitionLog
  written. Already-issued invoices remain payable; earned-but-unbilled work is billed manually via a
  change request or manual credit decision (kill fee = CR by contract). Portal shows read-only
  "Cancelled". Wrong source state → `422 PROJECT_NOT_CANCELLABLE { currentStatus }`.
- **Hold** `PATCH /projects/:id/hold` / `/resume` — **[PK projects.start]**. Bulk-applies the
  existing phase ON_HOLD/resume to every ACTIVE phase, suppresses overdue alerts while held, writes
  TransitionLog per phase. Project.status stays ACTIVE; a derived `isOnHold` (all non-terminal
  phases ON_HOLD) drives display. Idempotent.

## FS-L9 · Deposit Milestone

**[AMENDS FS-6.4]** In a `MILESTONE_BASED` quotation, the milestone at `order 1` MAY be a **deposit
milestone**: `triggerType: ON_PROJECT_START`, `phases: []` (exactly zero phases). Validation:
at most one; only at order 1; amount ≥ 0; still counts toward the Σ(amounts) = total ± 0.01 rule;
dependency-closure and exactly-once phase rules then apply to the remaining milestones over the full
phase set. Invoice fires at project Start (satisfies the VAT advance-payment invoicing duty,
Art. 53(1)(a)(2)). Violations → `422 DEPOSIT_MILESTONE_INVALID { rule }`.

## FS-L10 · Quotation Discount

**[AMENDS FS-6.3]** Quotation gains `discountAmount` (DECIMAL(12,2) ≥ 0, default 0) and
`discountReason?` (2–200). `totalAmount` = (services-sum | bundledPrice | override) −
`discountAmount`; must remain ≥ 0. Proposal and invoice documents render gross / discount / net.
The existing `totalOverridden` mechanism remains for custom pricing; discounts are for *visible*
concessions. Milestone sum rule unchanged (sums to the discounted total).

## FS-L11 · Zero-Amount Change Request

**[AMENDS FS-12]** `amount: 0` is valid on create (error `CHANGE_REQUEST_AMOUNT_REQUIRED` now fires
only for `amount < 0`). Approving a zero-amount CR sets status `APPROVED` (terminal) and generates
**no invoice**; `INVOICED` remains reserved for `amount > 0`. Notification unchanged.

## FS-L12 · Proposal Validity Window

Quotation gains `validUntil` (date, default: sentAt + 30 days, tenant-configurable default in
settings). Client `amend | accept` after `validUntil` → `422 PROPOSAL_EXPIRED`. Tenant may
re-send (existing REVISABLE flow) which re-stamps `validUntil`. Reject remains allowed after expiry.
Portal shows the deadline.

## FS-L13 · Notification Triggers, Preferences, Dunning

- **Wire the remaining FS §13 triggers** through the existing `NotificationService`:
  payment recorded → Finance keys; CR submitted → `change_requests.approve` holders; CR
  approved/rejected → creator + portal contact; proposal sent/amended/accepted/rejected (portal +
  tenant sides); phase overdue (scan below). All fan-outs filter `isActive: true` (fixes the known
  `createForUser` gap).
- **Preferences:** `GET/PATCH /me/notification-preferences` — per event type, per channel (in-app,
  email); critical security events non-optional. Contact preferences deferred.
- **Overdue scans [L1, needs BullMQ]:** hourly — (a) phase deadline passed & not COMPLETED & not
  ON_HOLD → notify `projects.view`; (b) invoice dunning ladder at due−3d / due / +7d / +14d →
  email + portal to the contact, in-app to Finance; per-tenant toggle. Business-day aware (FS-L16).

## FS-L14 · Analytics Endpoints

`GET /analytics/tenant` — **[PK analytics.view]** (first route for the reserved key) — computed on
read per FS §14 formulas: lead conversion rate, proposal win rate, active projects by status,
overdue projects, revenue (Σ payments − Σ refunds, period-filterable `?from&to`), outstanding
(Σ effective balances). Division-by-zero → 0. `GET /platform/analytics` **[Platform]**: MRR, plan
distribution, active tenants (30d). p95 > 2s → materialize before L2.

## FS-L15 · Data Export & PDPL Surfaces

- `POST /settings/export` **[PK settings.manage]** → async job builds a tenant data bundle
  (JSON/CSV per entity + fiscal PDFs), delivered as a presigned link (24h) + notification;
  rate-limited 1/day.
- DSR support (agency-side controllers use it for their clients; Dalil uses it for tenants):
  `POST /settings/dsr/contact/:id/export` and `.../delete` **[PK settings.manage]** — delete
  anonymizes the Contact (name/email → tombstone) while **retaining fiscal documents** (tax-law
  6-year duty overrides; recorded in the DPA) and append-only logs (anonymized reference).
- Consent & privacy: portal onboarding screen records `Contact.privacyNoticeAcceptedAt`; tenant
  signup records the DPA version accepted. Breach runbook + ROPA are documents in `docs/legal/`
  (business deliverables, not code).

## FS-L16 · Business-Day Rules

Shared helper `src/common/business-days/`: weekend = Friday+Saturday (tenant-configurable pair),
`addBusinessDays`, `isBusinessDay`. Used by: default deadline computation (`defaultDurationDays`
interpreted as business days — **[AMENDS FS-7]**, flagged to tenants), dunning ladder scheduling,
and any SLA math. Storage stays UTC ISO-8601 Gregorian everywhere (ZATCA mandate).

## FS-L17 · Client Session Length — ⚠ HUMAN DECISION REQUIRED

Conflict: FS-2 says client JWT "30-day/configurable"; the build ships 15 minutes, no refresh.
**Recommendation to ratify:** 12-hour Client-JWT for portal browsing; sensitive actions (accept
proposal, approve phase) unaffected (already authenticated); magic-link/onboarding token TTLs
unchanged. On ruling: update FS-2, `../contracts/frontend-contract.md`, and the guard TTL together, one
commit. Until ruled, the frontend treats expiry gracefully (re-entry screen).

## FS-L18 · New Error Codes (append to `error-codes.json`, FS §18 conventions)

| Code | HTTP | Details payload |
|---|---|---|
| `TENANT_TAX_PROFILE_INCOMPLETE` | 422 | `{ missing: string[] }` |
| `CREDIT_EXCEEDS_INVOICE` | 422 | `{ maximum }` |
| `PROPOSAL_EXPIRED` | 422 | `{ validUntil }` |
| `PROJECT_NOT_CANCELLABLE` | 422 | `{ currentStatus }` |
| `DEPOSIT_MILESTONE_INVALID` | 422 | `{ rule }` |
| `ZATCA_CLEARANCE_FAILED` | 422 | `{ providerCode?, providerMessage? }` |
| `PAYMENT_WEBHOOK_INVALID` | 401 | — |
| `INVOICE_NUMBER_SEQUENCE_CONFLICT` | 500 | — (internal retry) |

New permission key: `invoices.credit` (Invoices category; Administrator + Finance defaults).
Existing reserved keys gaining routes: `settings.manage` (FS-L1/L5/L15), `analytics.view` (FS-L14).
`invoices.create` remains reserved-unused (invoices stay system-generated only — by design).

## FS-L19 · Migration & Sequencing Notes (for the Orchestrator)

Slice order respects data dependencies: L1 wave = FS-L8…L13, L16 (no fiscal schema); L2 wave =
FS-L1 → L3 → L2 → L4 → L5 → L6 → L7 (tax profile before numbering before lines before notes before
ZATCA before payments before subscriptions), FS-L14/L15 anytime. Every slice through the harness
(D-060), every new tenant-scoped model into the isolation bucket, every new module registered in
`app.module.ts` (D-036), runtime proof per the standing discipline.

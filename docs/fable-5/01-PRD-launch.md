# Dalil — Product Requirements Document v2.0 (First Launch)

> **Supersedes PRD v1.6 for launch planning.** Sections unchanged from v1.6 are incorporated by
> reference (marked ⤷) so this document stays focused; everything new or corrected is written out in
> full. The Functional Specification lives in FS v2.0 + `02-FS-launch.md` (amendments); behavior
> always follows the FS. This PRD covers the **first launch only** — nothing beyond launch is in
> scope here, by design.

---

## 1. Product identity

Dalil is a multi-tenant subscription SaaS that runs a creative agency's entire commercial lifecycle
— CRM pipeline → quotation → proposal → phased project delivery → client portal → **compliant
invoicing** → payment — in one Arabic-first product for the KSA/GCC market.

**What changed since v1.6 (research-driven):**
1. **Compliance is now a core product pillar, not an afterthought.** KSA law (ZATCA e-invoicing
   Phase 2 — mandatory for all businesses above SAR 375k revenue since 30 June 2026; VAT Art. 53
   Arabic invoice content; PDPL, enforced since Sept 2024) applies to every invoice Dalil generates.
   "Compliant invoicing built in" is simultaneously a legal requirement and — per the competitive
   research — Dalil's sharpest differentiator: no global agency platform has it, and no GCC
   compliance tool has agency workflow.
2. **Stripe is removed everywhere.** Stripe does not serve Saudi-incorporated businesses (verified
   2026). All payment rails go through a SAMA-licensed local PSP (shortlist: Moyasar, Tap Payments).
   Success metrics previously phrased as "Stripe dashboard" now read "PSP dashboard / internal MRR
   analytics".
3. **Launch is defined as three explicit milestones** (§4) so "done" is testable.

## 2. Market & positioning (⤷ v1.6 §1–3 for vision; updated evidence)

- **Target:** creative/digital agencies in KSA first (branding, web, marketing, packaging — any
  phased-delivery service business), 2–50 seats. GCC expansion later.
- **The open slot (2026 research):** global agency platforms (Productive.io, Scoro, Accelo,
  Workamajig, Function Point) have no Arabic RTL, no ZATCA, no local rails; GCC local software
  (Daftra, Qoyod, Wafeq, Zoho) is accounting-first with no pipeline/blueprints/quotation-to-delivery
  engine or client portal. **No Arabic-first, ZATCA-native agency management product exists.**
- **Positioning statement:** *"The agency operating system for Saudi Arabia — from lead to legally
  compliant invoice, in Arabic and English."*
- **Category adoption killers to design against** (from competitor review analysis): steep
  onboarding (weeks), click-heavy dated UX, brittle accounting sync, seat-cost creep. Dalil's
  counter: guided setup with 3–5 reference blueprints, <30-min to first project, honest SAR pricing.

## 3. Actors, personas, RBAC (⤷ v1.6 §4, 4A, 4B — unchanged)

Three actors (Platform Owner · Tenant User with permission-key RBAC · Client with passwordless
portal), 33-key permission catalog, 8 default groups, four admin-protection invariants. All built
and tested; no changes for launch.

## 4. Launch definition — three milestones

**L0 — Demo (internal + sales demos).** The frontend (`dalil-web`) built over the existing API,
walking the full lifecycle on stub infrastructure (no real email/PDF/payments/realtime). Exit
criteria: the 10-step demo walk in `05-FRONTEND-START.md` §9 passes in Arabic and English.

**L1 — Pilot (3–5 design-partner agencies, assisted onboarding, real work).** Adds: deploy-phase
infrastructure (real email, S3 files, real PDFs, Redis/BullMQ, Socket.io), notification triggers +
dunning, project cancellation/hold, deposit milestones, discounts, proposal expiry, data export,
analytics dashboards. Billing of tenants may remain manual (invoice + bank transfer). Exit criteria:
one partner agency runs a real client project end-to-end, including getting paid, without touching
another tool for the covered workflow; open item E (exact-match rule) validated with partners.

**L2 — Public launch (paid, self-onboarding-ready).** Adds: full fiscal layer (VAT model, sequential
numbering, credit notes, Arabic-primary documents), ZATCA Phase 1+2 via certified compliance
provider, online payment links (mada/Apple Pay/cards) for client invoices, Dalil's own subscription
billing (tokenized recurring + 14-day no-card trial + dunning→suspension), PDPL pack (privacy
surfaces, DSR endpoints, DPA, breach runbook). Exit criteria: a VAT-registered tenant issues a
cleared standard tax invoice from Dalil; an agency signs up, pays, and is provisioned without
manual steps (assisted onboarding remains the default motion).

## 5. Feature modules at launch

### 5.1 Built and demo-ready (⤷ v1.6 §7 and FS v2.0 for full contracts)
CRM & custom pipeline · services & packages catalog · blueprints with dependency DAG + task boards ·
quotation with exact-match filter and milestone grouping · versioned proposals with client
amend/accept/reject · project execution (snapshot-on-start, phase FSM, three-lane approval flow) ·
client portal · invoicing (milestone/CR-triggered, partial payments, derived status) · change
requests · notifications (in-app) · analytics data capture (TransitionLog) · RBAC & team ·
platform console (plans, entitlements, tenant lifecycle).

### 5.2 New for launch — the fiscal & compliance layer (contracts in `02-FS-launch.md`)
1. **Tenant tax profile** — legal name (AR+EN), address, VAT registration status, TRN; compliance
   mode NOT_REGISTERED / PHASE_1 / PHASE_2.
2. **VAT-aware documents** — line items with ex-VAT unit price, per-line rate (15%/0%/exempt),
   explicit tax treatment per invoice with zero-rating narration; discounts shown gross/discount/net.
3. **Sequential invoice numbering** — per-tenant gapless sequence, tamper-evident.
4. **Credit & debit notes** — append-only corrections referencing the original invoice (refunds,
   goodwill, price adjustments). No invoice is ever edited or deleted.
5. **ZATCA e-invoicing** — via certified compliance-API provider (Wafeq / ClearTax / Complyance;
   selection is business open item G-01): per-tenant Fatoora device onboarding, B2B clearance,
   B2C 24-hour reporting, QR codes, 6-year archive. Dalil's architecture keeps this behind a
   compliance adapter so the provider is swappable and direct FATOORA integration remains possible.
6. **Payments** — client-facing: payment link on every invoice (mada, Apple Pay, cards) with
   webhook-reconciled Payment records; manual recording remains. Dalil-facing: subscription
   checkout, tokenized renewal, dunning → suspension, 14-day trial. Both on one SAMA-licensed PSP.
7. **PDPL pack** — consent & privacy notices, data-subject-request export/delete (30-day SLA),
   breach runbook (72h SDAIA notification), ROPA, tenant DPA. Hosting: verify AWS Riyadh
   (me-central-2) GA and prefer it; else Bahrain + SDAIA SCCs.

### 5.3 New for launch — delivery & billing completions (from the gap analysis)
Project cancellation + project-level hold · deposit milestone (invoice-on-start inside a milestone
plan) · zero-amount change requests · proposal validity window · overdue reminder ladder ·
remaining notification triggers + per-user preferences · analytics endpoints (six specced metrics)
· tenant data export · Sunday–Thursday business-day handling · file upload/download (S3 presign).

### 5.4 Explicitly NOT in launch (recorded rulings — see `03-GAP-ANALYSIS.md` §6)
Time tracking, profitability, resource planning, expenses, retainers (fast-follow; do not promise at
launch) · e-signature (portal Accept + audit trail is the launch acceptance record) · multi-currency
(SAR-only; zero-rated exports supported) · WhatsApp channel · Hijri dual display · native mobile ·
accounting integrations · public API/webhooks.

## 6. Screens (⤷ v1.6 §8 — the 60-screen inventory stands)

Launch adds to that inventory: tenant tax-profile settings; invoice payment-link page (public,
post-payment status); credit-note create/view; cancellation/hold dialogs with money-impact summary;
notification preferences; data-export request; trial/billing screens (L2). Build order and
per-screen API mapping: `05-FRONTEND-START.md`.

**Visual design:** the design foundations and first ten screen designs are delivered as the
**Dalil Experience System (DXS)** — warm cream/navy/terracotta, Arabic-first, light + dark
tokens — in the Claude Design project
(`https://claude.ai/design/p/a299c726-a791-4824-b931-1776a2cd9204`). All launch screens,
including the fiscal additions above, are designed and built from those tokens
(`08-UI-DESIGN-BRIEF.md` §4/§13); no per-screen palette or type decisions remain open.

## 7. Pricing & packaging (⤷ v1.6 §13 structure; validated against 2026 category norms)

Tiers stay Free / Growth (SAR 249) / Agency (SAR 599) / Enterprise — within category bands
(global entry $10–15/seat, mid $25–45/seat) while flat-tiered pricing stays simpler than per-seat —
a deliberate wedge. Adjustments from research: **14-day no-card trial** of Growth (category norm;
Free tier remains the fallback), client-portal users always free, annual billing at ~17% discount
(2 months free). ZATCA compliance included in ALL paid tiers (it is the wedge — never an add-on
fee). Pricing validation with the pilot cohort remains open item D.

## 8. Non-functional requirements (⤷ v1.6 §12 + FS §16; additions)

- **Compliance NFRs (new):** fiscal documents immutable and archived 6 years; invoice issuance
  remains available if the ZATCA provider is degraded (queue + retry; clearance completes async;
  status surfaced honestly) · PDPL DSR 30-day SLA · breach → SDAIA 72h.
- **Localization NFRs:** Arabic default UI; generated financial documents Arabic-primary bilingual
  (legal requirement, Art. 53(5)); Gregorian ISO-8601 dates system-wide; Sunday-start calendars;
  Fri–Sat treated as non-working by all deadline/reminder math.
- Performance, security, concurrency, testing NFRs: unchanged (p95 <200ms reads, RS256 JWTs,
  SELECT FOR UPDATE set, >80% service coverage, Storybook mandate D-007.8).

## 9. Success metrics (corrected)

- 200 paying tenants in 12 months; MRR SAR 185k by month 12 — measured from **internal MRR
  analytics + PSP dashboard** (not Stripe).
- <30 min from signup to first live project; >80% activation; <5% monthly churn; NPS >50.
- **New:** 100% of invoices issued by VAT-registered tenants clear ZATCA; >50% of invoice value
  paid via payment links within 6 months of L2; zero PDPL reportable incidents.

## 10. Open items (updated register)

| # | Item | Status |
|---|---|---|
| A | ~~Stripe SAR support~~ → **PSP selection: Moyasar vs Tap** (fees, mada-recurring support in writing, payout terms) | open — decide before L1 ends |
| B | Trademark check "Dalil" / buslahub umbrella | open — before public branding |
| C | Monitoring stack + deploy target; **verify AWS me-central-2 (Riyadh) GA** and prefer it | open — decide at L1 infra pass |
| D | Validate SAR pricing with pilot cohort | open — during L1 |
| E | Exact-match blueprint rule vs real agency service matrix | open — validate during L1 |
| F | "Saudi Air" / Saudia Sans font licence — the face is now IN USE as the custom "Dalil" family in the DXS design system (fallback: Inter, swappable via the `--font-english` token) | open — confirm before L2 polish |
| **G** | **ZATCA compliance provider selection** (Wafeq / ClearTax / Complyance — pricing is quote-based) | **new — start immediately; longest lead time** |
| **H** | **PDPL counsel review** (NDGP registration "main activity" test, DPA template, SCCs if Bahrain) | **new — before L1 takes real client data** |

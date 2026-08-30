# Dalil — Post-Launch Expansion Roadmap

> **Scope guard:** this document is deliberately EXCLUDED from the launch PRD and FS. Nothing here
> is launch work; nothing here may be referenced by launch specs, estimates, or task files. It exists
> so that (a) launch architecture decisions don't paint us into a corner, and (b) when launch is
> stable, the next bets are already researched and ordered.
> Trigger to open this file: first launch shipped, stable, with paying tenants (churn and support
> load under control).

Ordering principle: each wave funds and de-risks the next. Wave 1 closes the gap to the
agency-management category baseline (what Productive.io / Scoro / Accelo buyers assume exists).
Waves 2–3 deepen the moat. Waves 4–5 grow Dalil from agency tool toward a service-business ERP for
the GCC — the long-term prize, since no Arabic-first player occupies that slot either.

UI note for every wave: new surfaces extend the delivered DXS design system (tokens + patterns,
`08-UI-DESIGN-BRIEF.md` §4/§13) — post-launch features never introduce a second visual language.

---

## Wave 1 — Category parity (first 1–2 quarters after stable launch)

The competitive research (2026) is unambiguous: buyers of agency-management software treat these as
the definition of the category. Ship them before widening marketing beyond the design-partner cohort.

1. **Time tracking & timesheets** — the substrate everything else needs. Timer + manual entry on
   tasks, billable/non-billable flag, per-user cost rate vs billable rate, weekly timesheet view,
   approval flow. Architecture note: `TimeEntry` is tenant-scoped, append-only-ish (corrections as
   adjustments), keyed to task/phase/project so it can roll up. *This was consciously deferred from
   v1 (FS 1.4); it is the #1 fast-follow.*
2. **Per-project & per-client profitability** — revenue (invoices/payments already exist) minus cost
   (time entries × cost rate + expenses). Dashboard: margin by project, by client, by service type.
   This is the category's "dealbreaker differentiator" and Productive.io's entire brand.
3. **Retainer engagements** — recurring revenue is the dominant agency model and Dalil is
   milestone-only today. Retainer = engagement type with period (monthly), allowance
   (fixed-hours / fixed-value / unlimited), pre/post-paid, rollover policy, overage rate,
   auto-invoice on period close. Competitors handle retainer OR milestone well, rarely both on one
   client — Dalil's phase-milestone engine plus retainers is a genuine differentiator.
4. **Expense tracking** — billable expense lines on projects (subcontractor invoices, stock photos,
   ad spend), optional markup %, pull-through onto client invoices. Completes the profitability math.
5. **Resource & capacity planning (basic)** — per-user weekly capacity, allocation view from task
   assignments + time data, utilization % report. Scenario planning comes later.
6. **Reporting layer v2** — role-based dashboards (leadership: margin; ops: utilization; finance:
   AR aging), building on the analytics endpoints from launch.

## Wave 2 — Sales & client-experience deepening

1. **Proposal e-signature** — upgrade the portal Accept (already an authenticated acceptance record)
   to a signature capture + signed-PDF artifact; optional DocuSign integration for enterprise clients.
2. **Email integration** — Gmail/Outlook sync onto lead/contact timelines (the thing agency
   salespeople miss most from dedicated CRMs); email-in lead capture (forward-to-CRM address) and
   website webhook capture.
3. **WhatsApp Business integration** — GCC-specific and high-leverage: proposal-sent, invoice-issued,
   payment-reminder and phase-approved notifications over WhatsApp templates (client opt-in), since
   WhatsApp far outranks email for client communication in the region.
4. **Client portal v2** — white-label theming per tenant (Agency tier), custom domain support,
   client-side file exchange, comment threads on deliverables.
5. **CRM v2** — bulk lead actions, lead sources & campaign attribution, global activity feed,
   simple sales forecasting from pipeline value × stage probability.

## Wave 3 — Billing & finance deepening

1. **Accounting integrations** — GCC-first: Zoho Books, Qoyod, Wafeq (push invoices/payments/credit
   notes to the tenant's ledger); QuickBooks/Xero for non-GCC expansion. Brittle sync is the #1
   integration complaint across competitors — invest in idempotent, reconciliation-report-driven sync.
2. **Multi-currency** — currency + FX rate snapshotted per document, overridable at issue, base-SAR
   reporting with FX gain/loss. Never recompute historic documents.
3. **Dunning automation v2** — configurable reminder ladders, late-fee policies (pre-declared,
   jurisdiction-aware), collections handoff export.
4. **Payment experience v2** — saved cards / tokenized repeat payments for client invoices,
   BNPL-for-B2B experiments (Tabby/Tamara business products), partial-payment plans on large invoices.
5. **UAE readiness** — 5% VAT profile, AED currency profile, UAE e-invoicing (Peppol-based rollout
   expected 2026-2027) — the natural second market.

## Wave 4 — Platform & scale

1. **Public API + webhooks** (Agency tier `api_access` entitlement exists already) — OpenAPI-first,
   per-tenant API keys, outbound webhooks for invoice/project/lead events.
2. **Automation builder** — "when X then Y" rules per tenant (lead assigned → WhatsApp; phase overdue
   → escalate), replacing today's fixed notification triggers.
3. **Self-serve growth loop** — in-product plan upgrades, usage-based nudges at entitlement limits,
   referral mechanics, template gallery (tenant-shareable blueprints).
4. **Mobile companion app** (post-PMF per D-006/PRD ruling) — task board, approvals, time entry,
   notifications; not full parity.
5. **Advanced analytics** — the TransitionLog has been capturing lead-stage and phase-status history
   since day one precisely for this: stage-velocity, bottleneck detection, cycle-time benchmarks.

## Wave 5 — ERP modules (the long-term shape: GCC service-business ERP)

Each module reuses the platform primitives (tenancy, RBAC keys, append-only ledgers, document
numbering, ZATCA pipeline) and sells as an add-on entitlement:

1. **HR-lite** — employee records (extends User), leave requests + Sunday–Thursday/Hijri-aware
   calendars, onboarding checklists. GCC angle: Saudi labor-law leave types, GOSI fields, iqama/visa
   expiry reminders.
2. **Payroll (KSA)** — salary structures, GOSI calculations, WPS (Wage Protection System) file
   export, end-of-service benefit accruals. High regulatory moat, high willingness-to-pay.
3. **Procurement & vendor management** — purchase orders against projects (subcontractor engagements
   formalized), vendor bills, three-way match-lite, vendor portal.
4. **Inventory-lite** — for packaging/print agencies holding physical stock; item catalog behind an
   entitlement flag.
5. **Full accounting core** (build vs partner decision) — GL, chart of accounts, VAT return
   preparation, ZATCA-native (already built for invoicing). Either becomes "Wafeq with an agency
   front-end" or stays integration-first — decide with Wave 3 data on integration friction.
6. **Multi-entity / branches** — one agency, multiple legal entities (KSA + UAE), consolidated
   reporting; prerequisite for mid-market/enterprise tier.

## Standing architecture guardrails (so launch code never blocks this roadmap)

- Every new financial fact = a new append-only document linked to its cause (already the house
  pattern: Payment, CreditNote, ChangeRequest). Never mutate issued documents.
- Every new module = entitlement-gated from day one (`Entitlement.key` mechanism already exists).
- Keep `TimeEntry`, `Expense`, `Retainer` in mind when touching Project/Task schemas — don't
  denormalize progress or money in ways that assume tasks are the only cost driver.
- All notification triggers route through the notification service (never inline sends) so the
  Wave-4 automation builder can subsume them.
- Keep the platform console thin but real — plans/entitlements are the monetization surface for
  every wave above.

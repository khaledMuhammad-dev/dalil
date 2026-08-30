# Dalil — Launch Checklist

> Operational companion to the PRD milestones (L0 demo → L1 pilot → L2 public). Check items off in
> order; each block lists its external dependencies (the long-lead items to start first).
> Engineering items reference their contract (FS-L#) or gap (G-##).

## Start-immediately (longest lead times — all business-owner actions)

- [ ] **ZATCA compliance provider** — request quotes/demos from Wafeq API, ClearTax KSA, Complyance;
      confirm: per-tenant device onboarding via API, B2B clearance + B2C reporting, sandbox,
      pricing model. Decide (open item G). *Everything in L2 fiscal work keys off this.*
- [ ] **PSP selection** — Moyasar vs Tap Payments: get written confirmation of (a) mada +
      Apple Pay + card checkout links, (b) tokenized recurring for SaaS billing (mada-recurring
      support in the contract or Visa/MC-only), (c) fees, (d) payout terms to the company's bank.
      Decide (open item A).
- [ ] **PDPL counsel session** — NDGP registration ("main activity" test), DPA template for
      tenants, SCC need if hosting stays in Bahrain, breach-runbook review (open item H).
- [ ] **Verify AWS me-central-2 (Riyadh) GA** in the console; if GA, plan primary region there
      (removes the cross-border transfer question). Else me-south-1 + SDAIA SCCs.
- [ ] Trademark check "Dalil" / buslahub (open item B) · Saudia Sans font licence — the face is
      already shipping in the DXS design system as the "Dalil" family, so confirm the licence or
      swap the `--font-english` token to Inter (open item F) · company VAT/ZATCA posture for
      Dalil's OWN invoices to agencies.
- [ ] Export a standalone logo asset (SVG lockup + icon mark, both polarities) — the DXS screens
      carry only an inline decorative SVG (flagged in `08-UI-DESIGN-BRIEF.md` §11).

## L0 — Demo

- [ ] `dalil-web` scaffolded and built per `05-FRONTEND-START.md` slices 0–8.
- [ ] DXS design system installed in Slice 0 (tokens + SaudiaSans/"Dalil" fonts copied from the
      Claude Design project, Tailwind mapped to the CSS custom properties) and every screen built
      token-only per `08-UI-DESIGN-BRIEF.md` §13; screens with a delivered `.dc.html` design
      visually match it.
- [ ] 10-step demo walk passes in Arabic and English, light and dark.
- [ ] Storybook test-runner green; every shared component has its 8 story states.
- [ ] Backend untouched-but-verified: tsc 0 · eslint 0 · 1943+ tests green · built server boots.
- [ ] Demo dataset: seeded tenant enriched with realistic AR/EN sample data (services, blueprint,
      leads, one mid-flight project) so demos don't start empty.

## L1 — Pilot (design partners, real work, manual tenant billing allowed)

**Backend slices (harness, in `02-FS-launch.md` §FS-L19 order):**
- [ ] FS-L8 project cancel + hold · FS-L9 deposit milestone · FS-L10 discount ·
      FS-L11 zero-amount CR · FS-L12 proposal expiry · FS-L16 business days
- [ ] FS-L13 notification triggers + preferences (needs Redis/BullMQ from infra below for scans)
- [ ] FS-L14 analytics endpoints · FS-L15 data export/DSR
- [ ] FS-L17 client-session ruling made by Khaled and applied end-to-end
- [ ] G-25 verify auth rate limiting (10/min/IP)

**Deploy-phase infra (D-062 escalations):**
- [ ] Postgres + Redis managed instances · deploy target chosen (open item C) · CI deploy pipeline
- [ ] Real email adapter (SES) — all stubbed flows (onboarding, magic link, reset, invite,
      proposal, invoice, dunning) actually deliver; bilingual templates
- [ ] S3 + presign endpoints (uploads/deliverables/PDF storage), tenant-prefixed
- [ ] Real PDF rendering — Arabic-primary bilingual proposal + invoice templates (pre-fiscal
      versions; fiscal fields land in L2)
- [ ] Socket.io transport replaces polling where it matters (boards, notifications)
- [ ] Real JWT keys in secrets manager · platform refresh rotation (D-034) · e2e-smoke in CI
- [ ] Monitoring: Sentry + uptime/log stack (open item C); error alerting to the team
- [ ] Backups: automated Postgres PITR + restore drill performed once

**Pilot operations:**
- [ ] 3–5 design partners onboarded (assisted); pricing conversation logged (open item D)
- [ ] Exact-match blueprint rule validated against partners' real service matrices (open item E)
- [ ] Privacy notice + tenant DPA signed by pilots; ROPA drafted; breach runbook adopted
- [ ] Weekly feedback loop; churn-risk log

## L2 — Public launch (paid, fiscal, compliant)

**Fiscal engineering (order matters — FS-L19):**
- [ ] FS-L1 tax profile → FS-L3 numbering (+ back-numbering migration) → FS-L2 invoice lines/VAT
      → FS-L4 credit/debit notes + refunds → FS-L5 ZATCA adapter (provider) → FS-L6 payment links
      + webhooks → FS-L7 subscription billing + trial + dunning
- [ ] Invoice/credit-note PDFs carry: Arabic-primary content, sequential number, TINs, per-line
      VAT, treatment narration, ZATCA QR
- [ ] Fiscal archive: 6-year delete-protected storage verified
- [ ] Frontend: tax-profile settings, payment-link page, credit-note screens, billing/trial
      screens, ZATCA status surfaces

**Compliance sign-off:**
- [ ] One pilot tenant (VAT-registered) clears a real standard invoice through ZATCA production
- [ ] Simplified-invoice 24h reporting proven (if any B2C tenant)
- [ ] Dalil's own subscription invoice is itself compliant (Dalil-as-seller through the same pipeline)
- [ ] PDPL: DSR export/delete proven end-to-end; NDGP registration done if counsel says required
- [ ] Penalty-risk review: no invoice mutation paths, sequences gapless, archives immutable

**Go-to-market:**
- [ ] Pricing final (open item D closed) · trial live (14-day, no card) · plan entitlements verified
- [ ] Public site AR/EN, positioning: "from lead to legally compliant invoice"
- [ ] Support channel + SLA; onboarding guide + 3–5 reference blueprints installed per new tenant
- [ ] Status page; incident process; on-call rotation (even if it's one person)

## Standing rule

Nothing ships to a real tenant while a launch-blocking (L) gap in `03-GAP-ANALYSIS.md` §5 is open
for that tenant's usage: a NOT_REGISTERED micro-agency may pilot on L1; a VAT-registered agency may
not issue tax invoices from Dalil until the L2 fiscal block above is green for them.

# fable-5/ — Dalil Launch Pack

> Produced 2026-07-07 by a full-depth review: PRD v1.6 + FS v2.0 + DECISIONS.md digested, the
> as-built backend inventoried route-by-route (1943 tests green at time of writing), and external
> research run on the agency-software market and KSA regulation (ZATCA e-invoicing, VAT, PDPL,
> payment rails). These files are the **single briefing needed to take Dalil from "backend done"
> to a launched product** — written to be handed to an AI agent or engineer with zero context.

## Reading order

| File | What it is | Read when |
|---|---|---|
| `01-PRD-launch.md` | Updated PRD v2.0 — launch-scoped product requirements; corrects Stripe/KSA errors; defines launch as L0 demo → L1 pilot → L2 public | First — the "what & why" |
| `02-FS-launch.md` | FS launch amendments (FS-L1…L19) — testable behavioral contracts for every gap-closing feature, in the house style, harness-ready | When building any gap feature |
| `03-GAP-ANALYSIS.md` | The evidence: happy-path verification, 40+ edge-case scenarios, the numbered gap register (G-01…G-25), deliberate scope rulings, and the errors found & their fixes | Second — the "why these changes" |
| `04-AI-INSTRUCTIONS.md` | Permanent operating manual for any AI working on Dalil — every rule, trap, and verified lesson from the whole build. **Read every session, first.** | Always |
| `05-FRONTEND-START.md` | `dalil-web` kickoff: scaffold, API client layer, auth/RBAC/i18n wiring, slice-by-slice build order mapped to live endpoints, the 10-step demo acceptance walk | When starting frontend (= now) |
| `06-LAUNCH-CHECKLIST.md` | Operational checklist per milestone + the start-immediately long-lead business actions | Weekly, and before each milestone |
| `07-POST-LAUNCH-ROADMAP.md` | Post-launch expansion (category parity → GCC ERP modules). **Quarantined: never referenced by PRD/FS; not launch work.** | Only after launch is stable |
| `08-UI-DESIGN-BRIEF.md` | Complete UI design brief — all 51 Phase 0–6 screens, component library, status vocabulary, Arabic-first rules, asset manifest, acceptance checklist. **§4 now documents the delivered design foundations (DXS) and §13 is the build-implementation contract** | When designing OR implementing `dalil-web` screens |

## The design system (DXS) — delivered 2026-07

The Dalil visual language is no longer pending: the **Dalil Experience System** (foundations —
color, type, spacing, radius, elevation, motion, grid, iconography — plus ten high-fidelity
screen designs) lives in the Claude Design project *"Dashboard design brief"*
(`https://claude.ai/design/p/a299c726-a791-4824-b931-1776a2cd9204`). Identity: warm cream
`#F2EDDF` + navy ink `#232E52` + terracotta accent `#DE7356`; English set in the custom
"Dalil" face (Saudia Sans — licence still open item F), Arabic in IBM Plex Sans Arabic;
light + dark token layers, WCAG-AA tuned. `08-UI-DESIGN-BRIEF.md` §4 documents the tokens and
§13 tells the build agent exactly how to wire them into `dalil-web`.

## The five headlines (if you read nothing else)

1. **The demo is buildable today.** The backend lifecycle (lead → quote → proposal → project →
   invoice → payment) is complete, tested, and read-surface-ready with OpenAPI at `/api-docs`.
   Start `dalil-web` immediately per `05-FRONTEND-START.md`.
2. **ZATCA is launch-gating.** Since 30 June 2026 (Wave 24, SAR 375k), essentially every
   VAT-registered KSA business must clear/report e-invoices with ZATCA. Dalil's specs had zero
   VAT/tax modeling. The fiscal layer (FS-L1…L5) + a certified compliance provider is the price of
   admission — and, per the market research, also Dalil's sharpest differentiator (no Arabic-first,
   ZATCA-native agency platform exists).
3. **Stripe is not an option in KSA.** All payment flows (client invoice links AND Dalil's own
   subscriptions) run on a SAMA-licensed PSP — shortlist Moyasar / Tap. Get mada-recurring support
   confirmed in writing before designing subscription billing around it.
4. **PDPL is enforced now** (48 SDAIA decisions in 2025, fines to SAR 5M). The pack adds consent
   surfaces, DSR endpoints, a DPA, a 72-hour breach runbook — and prefer AWS Riyadh if it's GA.
5. **Time tracking / profitability / retainers are consciously NOT in launch** despite being the
   category baseline — they're Wave 1 of the post-launch roadmap. Don't promise them at launch;
   don't let scope creep pull them forward past the legal gaps.

## Provenance & verification status

- Backend facts verified against source on 2026-07-07: tsc 0, eslint 0, **1943/1943 tests, 99
  suites**, all routes/guards/models read from code, not docs.
- Regulatory findings carry inline sources in the research (ZATCA regs, VAT Implementing
  Regulations Art. 33/40/53/54/66, SDAIA transfer regs, stripe.com/global, PSP pricing pages).
  Items the research could not fully verify are flagged in `03-GAP-ANALYSIS.md`/`06-LAUNCH-CHECKLIST.md`
  as explicit confirm-before-relying actions (AWS Riyadh GA, mada recurring, provider pricing,
  NDGP registration duty).
- These are working documents: when a ruling lands (e.g. FS-L17 session length, provider picks),
  update the affected file in the same commit as the change, per the stewardship loop.

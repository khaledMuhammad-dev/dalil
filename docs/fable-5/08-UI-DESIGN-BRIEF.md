# Dalil — UI Design Brief (`dalil-web`)

> **Audience:** the AI designer producing every screen of the Dalil product — and now equally
> the build agent implementing them.
> **DESIGN STATUS (2026-07-10):** the foundations layer is DELIVERED as the **Dalil Experience
> System (DXS)** in the Claude Design project *"Dashboard design brief"*
> (`https://claude.ai/design/p/a299c726-a791-4824-b931-1776a2cd9204`), together with ten
> high-fidelity screen designs. §4 below now documents the real, extracted tokens; §13 is the
> implementation contract for `dalil-web`. The component library (§7) and the remaining screens
> are still to be designed/built **from these tokens only** — never invent a value outside them.
> **Scope:** all screens for PRD Phases 0–6 (Foundation → Invoicing & Change Requests).
> Phase 7 (analytics/polish) and Phase 8 (self-serve beta/billing) are explicitly OUT of scope.
> **You design against a finished backend:** every behavior described here is already
> implemented and tested (1943 green tests). Do not invent flows — design the ones listed.

---

## 1. What Dalil is (read this first)

Dalil is a multi-tenant SaaS for **creative agencies in Saudi Arabia and the GCC** (branding,
web, marketing, packaging — any phased-delivery business). It runs the agency's entire
commercial lifecycle in one system:

```
Lead (CRM pipeline) → Quotation (built from service blueprints) → Proposal (client reviews,
amends, accepts) → Project (phases + task boards) → Client approvals → Invoices (per milestone)
→ Payments → Change requests (extra scope → extra invoices)
```

The buyer is the **agency owner/manager** — a businessperson, not a developer. Every screen
must answer their real question fast: *Who owes me money? What is stuck? What does the client
think?* The design must feel like a calm, professional business tool — closer to a well-made
banking app than a playful startup dashboard.

## 2. The three surfaces (one design system, three moods)

| Surface | URL group | Users | Mood |
|---|---|---|---|
| **Agency app** | `(agency)` | Agency staff, RBAC-gated | The main product. Dense but calm; productivity-first; sidebar navigation |
| **Client portal** | `(portal)` | The agency's clients | Simple, reassuring, minimal choices, zero jargon. The agency's brand should be able to sit on it (white-label surface: agency logo + accent) |
| **Platform console** | `(platform)` | Dalil's own operator | Small internal tool; function over polish; same components, no marketing shine |

All three share one component library. Design the components once (§7), then compose screens.

## 3. Design principles (in priority order)

1. **Arabic-first.** Arabic (RTL) is the default language and the primary design target.
   Design every screen in Arabic RTL **first**, then derive the English LTR mirror — never the
   reverse. Layouts must be fully mirrored: navigation on the right, reading flows right-to-left,
   directional icons (arrows, chevrons, "back") flipped. Kanban boards flow right→left in RTL.
2. **Serve the business.** Every list leads with the business-critical fact: money amounts,
   statuses, and dates are the loudest elements after the entity name. Overdue/blocked items
   are visually impossible to miss.
3. **Easy and understandable.** The target user is a busy agency owner, not a power user.
   One primary action per screen. Wizard steps for anything with more than one decision
   (quotation builder, milestone grouping). Plain-language labels — never expose internal
   terms like FSM state names raw (map `PENDING_START` → "بانتظار البدء" / "Awaiting start").
4. **Trust through precision.** Money is always exact (SAR, 2 decimals, never rounded away).
   Statuses always come from the fixed vocabularies in §6 — never invent a status label.
5. **Never dead-end the user.** Every error state names what happened and offers the next
   step (see §9). Every empty state teaches ("No blueprints yet — create your first").

## 4. Foundations — DELIVERED as the Dalil Experience System (DXS)

> These are no longer requests — they are **named, extracted tokens** in the Claude Design
> project: `styles.css` (single entry point) importing
> `tokens/{fonts,colors,typography,spacing,radius,elevation,motion,grid}.css`, with specimen
> cards under `guidelines/`. Every hex/px was extracted from Dalil's own built screens, not
> invented. Token naming is two-layer: raw scale (`--ink-500`, `--cream-200`, `--brand-700`)
> → semantic alias (`--text-secondary`, `--surface-card`, `--brand-solid`) — **screens and
> components consume the semantic layer only** (exception: `--cat-*` and status tokens are
> already semantic and used directly). Implementation mapping for `dalil-web`: §13.

- **Identity in one line:** warm cream canvas + deep navy ink + one terracotta accent —
  deliberately NOT a blue/indigo/gray SaaS look. No gradients or photography in UI chrome.
- **Typography:** English = the brand's custom face **"Dalil"** (Saudia Sans files
  self-hosted as woff2/woff, weights 400/500/600/700, registered under `font-family: 'Dalil'`);
  Arabic = **IBM Plex Sans Arabic** (400–700). Dense, data-forward scale — nothing above
  24px (stat numbers); page titles 18px/700 (tracking −0.2px); section titles 16px/700;
  card titles 14px/600; body 13px/1.7; buttons 13px/600; labels/captions 12px; eyebrow/helper
  11px. **Hierarchy is carried by weight (500/600/700), not size jumps** — this suits Arabic,
  where large size jumps read poorly. Arabic gets looser leading at the same sizes (titles 1.5,
  body 1.75, captions 1.65). Numbers/amounts in Western digits (0–9) in both locales.
- **Color (light, the shipped theme):** app background cream `#F2EDDF`; cards/panels/inputs
  `#FFFDF6`; sunken surfaces (table headers, chips) `#F7F2E4`; control tracks `#EDE6D2`.
  Text and borders are **navy ink `#232E52` at varying opacity — never a flat gray**
  (borders at 7–18% alpha: `--border-subtle/default/strong/input`). Brand accent is one
  terracotta: `#DE7356` for accents/active nav/focus ring, `#C9603F` (`--brand-solid`) for
  primary button fills (AA-tuned), `#B4522F` hover/links-strong. **Two solid button colors:**
  terracotta CTA + plain ink navy (`#232E52`) for Edit/Save-type solids. Text on solids is
  warm off-white (`#FFF6ED` / `#F6F0E0`), never pure white. `::selection` is gold `#EDC488`.
- **Category & status hues (four pairs, bg/fg):** branding `#F8E1D8`/`#B4522F` · web
  `#E4EAF7`/`#33406B` · marketing `#EFE7CF`/`#8A7434` · software `#DFEDE5`/`#2C6B51`. They
  double as semantics: success = the green pair (+dot `#4E9E76`), danger = the terracotta pair,
  info = the blue pair, warning = gold tints (`--warning-*`). Use the `--success-* /
  --warning-* / --danger-* / --info-*` tokens for the §6 status-chip system.
- **Dark mode:** delivered **at the token level** — a full `[data-theme="dark"]` block
  re-points the semantic layer (deep navy surfaces `#141B30`/`#1E2740`, warm off-white text
  `#F6F0E0`, terracotta lightened to `#F4A986` as accent), every pairing WCAG-AA verified.
  The delivered screens are rendered light; verify each screen against the dark tokens as it
  is designed/built.
- **Elevation:** cards/rows/panels are flat surfaces with an ink-tinted **border, not a
  shadow**. Shadow appears only on true floaters: dropdowns, the mobile drawer, toasts, and
  the terracotta glow under the primary CTA (`0 2px 8px rgba(222,115,86,.32)`). Modal scrim
  `rgba(20,26,46,.55)`.
- **Radius:** 6 / 8 / **10 (workhorse: buttons, inputs, nav rows)** / 14 (cards, modals) /
  16 (large containers) / pill.
- **Spacing:** strict **8-point grid** — base 8px with a single 4px half-step for micro gaps
  (`--space-1…11` = 4/8/12/16/24/32/40/48/64/80/96).
- **Motion:** fades + small translateY entrances; 150ms hover / 200ms entrances / 350ms
  panel-step transitions; shimmer loop 1300ms (skeletons), breathe loop 3600ms (hero mark).
  No spring easing in use (one is reserved). `prefers-reduced-motion` zeroes every duration.
- **Focus (never removed):** 2px terracotta outline with 2px offset, plus a soft
  `rgba(222,115,86,.16)` halo on focused inputs — consistent everywhere.
- **Iconography:** Lucide only, 2px stroke, `currentColor`; 17px nav/section, 15px
  search/inline, 11–14px chevrons/meta. No icon fonts, no emoji as UI icons.
- **Base kit:** the build uses TailwindCSS + Shadcn UI. Stay within Shadcn's component
  vocabulary (dialog, sheet, popover, dropdown, tabs, table, toast, badge, skeleton) so the
  design is directly buildable — custom components only where §7 names them; skin everything
  with DXS tokens.
- **Layout:** web-only, responsive, desktop-first. Real shell constants: sidebar 240px
  expanded / 68px collapsed / 264px mobile drawer; app header 68px; workspace sub-page header
  64px; inspector panel 320px; content max-width 1080px. Design desktop (1440) as primary,
  plus tablet (768) and mobile (375) for every screen. Agency app: collapsible sidebar +
  topbar (tenant name/logo, notification bell, locale toggle, user menu — no tenant switcher).
  Portal: simple top navigation only.
- **Dates & calendar:** the work week is **Sunday–Thursday** (weekend = Fri/Sat) — any
  calendar or date-picker must reflect this. Dates render in local time, Gregorian.
- **Money:** always `SAR 12,500.00` style with currency; never truncate decimals on
  invoices/quotations/milestones.
- **Density:** comfortable, not cramped. Tables max ~8 columns; overflow into detail views.

## 5. Personas (design for these three)

1. **Agency owner/admin (Arabic-first):** lives in projects list, approval queue, invoices.
   Needs at-a-glance status, minimal clicks to approve/record payment.
2. **Sales member:** lives in the leads kanban and quotation builder. Sees only what their
   permissions allow — the UI must degrade gracefully when actions are hidden (§10).
3. **The agency's client (least technical):** enters via a magic link in email. Sees a
   simple portal: my projects, the proposal to approve, deliverables to review, invoices to
   pay. Must never see agency-internal language (no "kanban", no "FSM", no permission talk).

## 6. Status vocabulary (fixed — design a chip/badge for every value)

Design one coherent status-chip system covering all of these. Suggested semantic mapping in
parentheses; keep colors consistent across modules (e.g. all "completed" states share one green).
DXS token mapping: success → `--success-bg/fg/dot` · warning → `--warning-*` · danger →
`--danger-*` · info → `--info-*` · neutral/muted → ink-alpha chips on `--surface-sunken`.
Both themes come free from the tokens — never hardcode a chip color.

| Domain | Values |
|---|---|
| Lead | OPEN (neutral) · CONVERTED (success) · STOPPED (muted) |
| Quotation/Proposal | DRAFT (neutral) · SENT (info) · AMENDMENT_REQUESTED (warning) · ACCEPTED (success) · REJECTED (danger) · REOPENED (neutral) |
| Project | PENDING_START (neutral) · ACTIVE (info) · COMPLETED (success) · ARCHIVED (muted) · CANCELLED (danger) |
| Project phase | PENDING (muted — waiting on dependencies) · ACTIVE (info) · IN_REVIEW (warning) · CLIENT_PENDING (warning) · COMPLETED (success) · ON_HOLD (muted/striped) |
| Invoice | UNPAID (warning) · PARTIALLY_PAID (info) · PAID (success) · OVERDUE (danger) |
| Change request | REQUESTED (warning) · APPROVED (success) · REJECTED (danger) · INVOICED (info) |

Client-portal phase labels are simplified: Completed / Active / Pending only (clients never
see IN_REVIEW / ON_HOLD internals).

## 7. Shared component library (design these before screens)

Every data view ships **8 designed states**: default · variants · loading (skeleton) · empty ·
empty-on-filter ("no results, clear filters") · error (with retry) · RTL · dark mode.

1. **App shells** — agency sidebar layout, portal topnav layout, platform layout; auth-page
   centered layout.
2. **Status chip** system (§6) + **progress bar** (phase %, project %) + **New/unread badge**.
3. **Data table** — sortable header, cursor "load more" pagination, row actions, filter bar
   with chips, the 8 states.
4. **Kanban board** — column (header: name + count), card, drag affordance + drop target,
   optimistic-move state and its **rollback error** state ("move failed, card returned").
   Used twice: leads pipeline and project task board. RTL: columns flow right→left.
5. **Wizard/stepper** — used by quotation builder and lead-convert flow. RTL-aware step order.
6. **Forms** — RHF+Zod based: text/number/money input (SAR affix), select, combobox with
   search (contact picker), date picker (Sun–Thu week), multi-select, toggle; **inline
   field-level errors** (validation errors arrive per-field from the API).
7. **Dialogs** — confirm (destructive variant), form dialog, and the **force-confirm
   pattern**: some actions fail with a warning and require an explicit "do it anyway" retry
   (e.g. deleting a sales stage that has active leads).
8. **Notification drawer** — bell with unread count, feed grouped by day, mark-as-read,
   deep-link rows. (Updates arrive by polling — no "live" indicator needed.)
9. **Money summary block** — total vs paid vs remaining; used on invoice, project, milestone
   views.
10. **Timeline / activity feed** — contact history, task activity, audit entries.
11. **Document frame** — branded proposal/invoice preview page (paper-like, printable,
    agency logo slot, works in AR and EN).
12. **Error screens** — 404, 403 (says which permission is missing in friendly terms), 500
    (with retry + support reference code `requestId`).
13. **Empty-state illustration style** — define one consistent style (line, duotone, or
    geometric — pick one and use everywhere).

## 8. SCREEN INVENTORY — everything to design, grouped by build phase

Numbering follows PRD §8 (the authoritative 60-screen inventory). Each screen: purpose, key
elements, and its critical edge states. **Every screen: AR-RTL + EN-LTR, light + dark,
desktop + tablet + mobile.**

### Phase 0 — Foundation & access

| # | Screen | Notes for the designer |
|---|---|---|
| 1 | **Login (agency)** | Email + password, forgot-password link, locale toggle visible pre-login. Error: invalid credentials (non-specific message). |
| 2 | **Forgot / reset password** | Email entry → confirmation state → new-password form (token from email link). Expired-token error state. |
| 34 | **User profile & password** | Name, email, avatar, password change, language preference. |
| 48 | **Notifications drawer** | See component §7.8. |
| 49/50/51 | **404 / 403 / 500** | See component §7.12. 403 shows the missing permission in plain words. |

**Team & RBAC (agency settings area):**

| # | Screen | Notes |
|---|---|---|
| 31 | **Team members list + invite** | Roster with groups + active/inactive; invite-by-email dialog (invited state: "invitation sent"); deactivate with confirm. Edge: cannot deactivate the last Administrator — design this exact 422 error as a clear inline explanation, not a toast. |
| 31b | **Assign groups to user** | Group multi-assign + optional per-user permission override list (GRANT/REVOKE per key, grouped by category). |
| 31c | **Permission Groups** | List (8 defaults + custom), create/duplicate/edit/delete. Editor: permission checkboxes grouped by category (33 keys, 16 categories — render from data, design the grouped-checklist pattern). Edge states: system Administrator group is visibly locked; the three admin-protection violations (can't delete system group / can't remove last admin / can't revoke core admin permissions) each get a specific, friendly inline error. |
| 32 | **Tenant settings** | Agency name, logo upload slot, subscription plan display, billing info. |

**Platform console (Dalil operator — keep visually plain):**

| # | Screen | Notes |
|---|---|---|
| 44 | **Platform dashboard** | Tenant count, MRR, plan distribution, system health. Aggregate numbers only — stat tiles + one table, no fancy charts (analytics polish is Phase 7). |
| 45 | **Tenants management** | List with subscription status; provision (form dialog), suspend / reactivate with confirm. |
| 46 | **Plans & pricing editor** | Plan list; editor with price, currency, entitlement flags (max users, max active projects, feature toggles). |
| 47 | **Platform settings** | Global config + announcements. Low priority — one settled layout is enough. |
| — | **Platform login** | Separate, unbranded-tenant variant of screen 1. |

### Phase 1 — Catalog & CRM

| # | Screen | Notes |
|---|---|---|
| 4 | **Services catalog** | List: name, default price, "used by N blueprints". Add/edit/delete. Edge: deleting a service that's in use → offer **Archive** instead (design the archived visual state + filter). |
| 5 | **Service editor** | Name, description, default price; usage panel (which blueprints/packages reference it). |
| 6 | **Packages list** | Name, included services (chips), bundled price. Edge badge: "contains archived service" warning on affected packages. |
| 7 | **Package editor** | Select ≥2 services, set combined price; validation error state for <2 services. |
| 8 | **Leads pipeline board** | THE flagship CRM screen. Kanban of custom stages + fixed Converted/Archived columns. Card: contact name, company, value?, source, assignee avatar, age. Drag with optimistic move + rollback. Search + filter bar. |
| 9 | **Lead detail / contact profile** | Contact info, activity/history timeline, linked quotations & projects, primary **Convert** action. Convert dialog is a mini-wizard: YES (creates client + sends portal invite) / NO (stop with reason). Reopen action for stopped leads. |
| 10 | **New lead form** | Contact fields, source, assignee. Fast entry — single column, sensible tab order (RTL tab order!). |
| 11 | **Sales flow configurator** | Add/rename/reorder/remove stages, drag-to-reorder. Edge: removing a stage with active leads → warning dialog listing count, explicit "move them and delete" force-confirm (§7.7). |
| 35 | **Sales dashboard** | "My leads" summary: counts per stage, conversion rate, recent activity. Scoped to the signed-in salesperson. Stat tiles + list — keep simple. |
| 3 | **Magic-link landing (client onboarding)** | Client clicks email link → token validated → welcome + set-up account. Branded warmly; this is the client's first touch. Expired/used-token error state with "request a new link". |
| 3a | **Client login (re-entry)** | Email entry → "we sent you a link" state; OR email+password if set. Session-expired variant ("your session ended, enter email again"). |
| 3b | **Client set-password (optional)** | Offered after first sign-in; skippable. |

### Phase 2 — Blueprint engine (agency setup screens)

| # | Screen | Notes |
|---|---|---|
| 18 | **Blueprint library** | Cards or table: name, phase count, services covered, linked project count. |
| 19 | **Blueprint editor** | The most technical agency screen — make it visual. Ordered phase list; per phase: name, one service, dependencies on other phases (sequential/parallel), board assignment (dedicated or shared). Show the dependency structure graphically (simple DAG / connected column view). Edge: circular dependency error, shown on the offending link, not as a toast. |
| 20 | **Task board template editor** | Define columns (order matters — **the last column is "Done"**; make that explicit in the UI), task templates per column, assign templates to phases. |

### Phase 3 — Quotation & Proposal (the revenue path — highest design priority)

| # | Screen | Notes |
|---|---|---|
| 12 | **Quotations list** | Table: contact, blueprint, payment model, total, status chip (6 states §6), date. Filters by status. |
| 13 | **Quotation builder (wizard)** | Steps: ① contact → ② services **or** package (exclusive choice — design the XOR clearly) → ③ blueprint (system shows only **exact-match** blueprints; design the no-match state: "no blueprint covers exactly these services" + link to create one) → ④ payment model → ⑤ milestone grouping (screen 14) → review. |
| 14 | **Milestone grouping step** | ⚠ THE hardest, most original UI in the product — invest the most design effort here. Phases displayed in dependency order; user draws billing boundaries to group phases into named milestones and assigns an amount per milestone. Live validation footer: sum of milestones vs quotation total (must match to the halala). Four inline error states, each anchored to the offending element: (a) grouping breaks a dependency chain — highlight the phases that must move together; (b) a phase in no milestone; (c) a phase in two milestones; (d) amounts don't sum — show expected vs actual. Zero-amount milestone is legal. One-time payment model = one milestone containing everything (collapsed simple state). |
| 15 | **Proposal preview** | Branded document frame (§7.11): agency logo, services, phases, milestones with amounts, total. Send action; version history rail (v1, v2…). |
| 16 | **Proposal sent / tracking** | Status, sent-to, read state, amendment requests received, resend. |
| 17 | **Proposal amendment review** | Client's comments displayed alongside the quotation; "revise" → back into builder → regenerate & resend as new version. Old versions clearly marked stale/read-only. |
| 23 | **Project setup / start screen** | After client accepts: review accepted proposal, blueprint, milestones → single big **Start project** action. Edge errors: plan's active-project limit reached (upsell-toned message); source blueprint was deleted (explain + dead end with support hint); already started. |

### Phase 4 — Projects & task boards

| # | Screen | Notes |
|---|---|---|
| 21 | **Projects list / dashboard** | Filter by status; per row: client, progress bar, current phase(s), status chip, unpaid-amount hint. This is the owner's home screen — make status scanning effortless. |
| 22 | **Project detail — overview** | Overall progress, phase list with per-phase progress + status chips (6 phase states §6), milestone/payment summary block (§7.9), timeline. Phase hold/resume actions. |
| 24 | **Phase detail** | The phase's task board (or shared board **filtered to this phase**), phase progress, approval status, asset panel (design the panel; file upload is deferred — show a disabled/coming state seam). |
| 25 | **Kanban task board** | Columns from the board template; task cards: title, assignee avatar, due date, phase badge (on shared boards), "New concept" flag. Drag between columns; phase status banner when ON_HOLD (board read-only visual). |
| 26 | **Task card detail (modal/sheet)** | Title, description, assignee picker (edge: inactive user can't be assigned), due date, status, comments, activity feed, attachments area (deferred seam — visible but marked). |

### Phase 5 — Approvals & Client portal

**Agency side:**

| # | Screen | Notes |
|---|---|---|
| 27 | **Approval queue** | Pending phase reviews across all projects, grouped by project. Zero state: "nothing waiting on you" (a *good* empty — celebrate it). |
| 28 | **Approval review** | Deliverable context, comment field, **Approve** / **Request changes** actions. Edge: someone else already reviewed it — design the "already handled" resolution state. |

**Client portal (simple, warm, jargon-free, agency-brandable):**

| # | Screen | Notes |
|---|---|---|
| 36 | **Portal home / project list** | Client's projects with big progress indicators + "request a new project" button (simple request form). |
| 37 | **Proposal review** | Document view + three clear paths: **Accept** (celebratory confirm), **Request changes** (comment box), **Reject**. This screen closes deals — it must feel trustworthy and effortless. |
| 38 | **"Not yet started" holding state** | Post-accept reassurance: "Your project is confirmed and will begin shortly." |
| 39 | **Project overview (client)** | Progress bar, phases as Completed/Active/Pending only, milestone & payment summary. |
| 40 | **Phase deliverables view** | Deliverables list for the active phase, unread badges. |
| 41 | **Deliverable viewer** | Inline preview (image/PDF/video frame), feedback thread, comment input. (Real files deferred — design with placeholder media.) |
| 42 | **Approval confirmation (client)** | Approve the phase → success state + "what happens next" (the next invoice, the next phase). |
| 43 | **Invoice view (client)** | Invoice list with status chips; detail with line items; PDF download (deferred seam); payment instructions area. |

### Phase 6 — Invoicing & Change requests

| # | Screen | Notes |
|---|---|---|
| 29 | **Invoices list (agency)** | Number, project, source (milestone / change request), amount, paid, status chip (4 derived states §6), due date. Filters: project, status, source. OVERDUE rows visually urgent. |
| 30 | **Invoice detail** | Line items, source reference, **amount vs amount-paid vs remaining** money block, payment ledger (append-only history), record-payment action, PDF export (deferred seam). |
| 30a | **Record payment (dialog)** | Amount, date, method. Edge: amount exceeds remaining balance → inline error stating the exact maximum accepted. On success, status recomputes live (UNPAID → PARTIALLY_PAID → PAID). |
| 30b | **Change request** | Create (staff or client-initiated from portal): description, amount, type **SCOPE** or **OVERTIME**. Review: approve/reject with confirm — approval **auto-generates an invoice**; design the success state that shows the new invoice immediately. List + detail with 4 CR status chips. |

### Explicitly OUT of scope (do not design — Phase 7/8)

- Tenant analytics dashboard (34a), platform analytics charts (34b), workload view (31a)
- Notification preferences screen (33)
- Self-serve tenant signup, subscription checkout/billing screens
- Real-time presence/live indicators (data refreshes by polling — no live badges)
- File upload/download UX beyond the disabled seams noted above; online payment checkout

**Screens in scope: 51.** (PRD total is 60; the 9 excluded above belong to Phases 7–8.)

## 9. States are the deliverable, not an afterthought

For EVERY data view, deliver: **loading (skeleton) · empty (with guidance) · empty-on-filter ·
error (with retry + `requestId` support code)** — plus the happy state. For every form:
default · field-level validation errors · submitting · server-rejection (business-rule 422
with a specific message) · success. The backend returns ~40 distinct business error codes;
the ones that matter visually are already embedded per-screen above — design those as
**inline, anchored, specific messages**, never generic toasts.

## 10. Permission-aware UI

Every action in the agency app is gated by a permission key. When the user lacks it:
**hide primary/nav actions** (don't tease), **disable-with-tooltip** only where hiding would
confuse layout. Design both looks for: action buttons, table row actions, nav items. A
salesperson's app and an admin's app are the same screens with fewer controls — verify each
screen still composes well when its gated actions vanish.

## 11. Assets — current status

1. **Logo: STILL NEEDED.** The "logo" in the delivered screens is an inline SVG (concentric
   navy/terracotta circles) copy-pasted per file — not a portable asset. Request from the
   owner: full lockup + icon-only mark, light-on-dark and dark-on-light, standalone SVG.
2. **Brand palette: DELIVERED** — the warm cream/navy/terracotta system in `tokens/colors.css`
   (light + dark semantic layers). No swap pending; these ARE the brand colors.
3. **English typeface: DELIVERED** — Saudia Sans woff2/woff (4 weights) in `assets/fonts/`,
   registered as the "Dalil" family. ⚠ Licence confirmation is still PRD open item F
   (fallback: Inter) — flag before L2, do not treat as legally settled.
4. Product wordmark in Arabic and English ("دليل" / "Dalil") and preferred usage — still needed.
5. Favicon/app icon — still needed.
6. Any photography/illustration assets, or approval of the empty-state illustration style (§7.13) — still needed.
7. Tone-of-voice samples for Arabic microcopy (or you propose; formal-friendly Saudi business Arabic — address the user with respect, no slang) — still needed.

## 12. Deliverables & acceptance checklist

- [x] Design tokens: color (light+dark), type scale (AR+EN), spacing, radius, elevation —
      **delivered as DXS** (`styles.css` + `tokens/`, see §4/§13). Motion + grid included.
- [ ] Component library (§7) — every component in all 8 states, RTL and LTR.
- [ ] All 51 screens (§8): Arabic-RTL primary + English-LTR, light + dark, at 1440 / 768 / 375.
- [ ] The five flagship flows as connected prototypes/flows:
      ① lead → convert, ② quotation builder incl. milestone grouping, ③ client proposal
      review → accept, ④ task board → phase review → client approval, ⑤ invoice → record
      payment → paid.
- [ ] Every screen's empty/loading/error states delivered alongside its happy state.
- [ ] Status chips for every value in §6 — one consistent system.
- [ ] Microcopy in both languages for all statuses, empty states, and the named error cases.
- [ ] File/frame naming: `phase-<n>/<screen-number>-<slug>/<locale>-<theme>-<breakpoint>`.

**Definition of done for the whole engagement:** an engineer can build the entire demo walk
(login → configure services & blueprint → lead → convert → quotation with milestones →
proposal → client accepts in portal → project starts → tasks → phase approval → invoice →
partial then full payment → change request → second invoice) using only these designs,
without asking a single "what does this state look like?" question.

## 13. Implementing the design — DXS → `dalil-web` (the build contract)

**Source of truth:** the Claude Design project *"Dashboard design brief"*
(`https://claude.ai/design/p/a299c726-a791-4824-b931-1776a2cd9204`). Files that matter to the
build: `styles.css` (entry point) · `tokens/*.css` (8 files) · `assets/fonts/SaudiaSans-*`
(8 files) · `guidelines/*.card.html` (specimens) · the `*.dc.html` screen designs ·
`readme.md` (extraction rationale + caveats).

1. **Install the tokens first (Slice 0).** Copy `styles.css` + `tokens/` + `assets/fonts/`
   into `dalil-web` and load them globally; map Tailwind theme values onto the CSS custom
   properties (colors, spacing, radius, fonts reference `var(--…)`). Never hardcode a hex/px
   that exists as a token — that is a review failure, same class as a physical `pl-`/`pr-`.
2. **Consume semantic aliases only** (`--text-*`, `--surface-*`, `--border-*`, `--brand-solid`,
   `--icon-*`, `--focus-ring*`). Raw scales (`--ink-*`, `--cream-*`, `--brand-500` etc.) are
   internal. Exception: `--cat-*` and `--success/warning/danger/info-*` are consumed directly.
3. **Respect the AA tuning baked into the aliases:** body secondary text is `#5E6480` (NOT
   `--ink-500`); links are `#A94A2C`; the primary button FILL is `--brand-solid` = brand-600
   `#C9603F` with label `#FFF6ED` — the identity terracotta `#DE7356` is for accents, active
   nav, and the focus ring, never a button fill behind a label. Status text on tints uses the
   `-aa` variants (`--warning-fg-aa`, `--danger-fg-aa`).
4. **Two solid buttons, one CTA per view:** terracotta (`--brand-solid`, + `--shadow-cta-glow`)
   for THE primary action; navy solid (`--navy-solid` = ink-900) for secondary solid actions
   (Edit/Save). Everything else is ghost/outline on cream. Text on solids: `#FFF6ED` /
   `#F6F0E0`, never `#FFFFFF`.
5. **Dark mode = `data-theme="dark"` on `<html>`.** The semantic layer re-points; component
   code touches nothing. Screens were designed light — check each against dark tokens as built
   (the D-007.8 dark story state is the gate).
6. **Fonts:** self-host the 8 SaudiaSans files as `font-family: 'Dalil'` (weights 400/500/600/
   700, `font-display: swap`); switch by locale via `--font-english` / `--font-arabic`.
   `tokens/fonts.css` pulls IBM Plex Sans Arabic from Google Fonts — fine for the demo;
   self-host it at deploy. Licence caveat: §11.3 / open item F.
7. **Icons:** the design files use a `<l-icon>` web component loading Lucide from unpkg —
   in `dalil-web` use `lucide-react` instead (same set). 2px stroke, `currentColor`, sizes per
   §4. Do not import any other icon set.
8. **Layout constants come from `tokens/grid.css`** (sidebar 240/68/264-drawer, headers 68/64,
   inspector 320, content max 1080) — don't re-measure the screen HTML.
9. **Spacing discipline:** the token scale is a strict 8pt grid, but the delivered screen HTML
   still contains older off-grid values (5/7/9/11/13/22…). **Build from tokens, not by
   pixel-copying screen markup** — the screens are visual precedent, the tokens are law.
10. **Elevation discipline:** cards/rows get `--border-default`, `--shadow-card` is literally
    `none`. Only dropdowns (`--shadow-dropdown`), the mobile drawer, toasts, and the CTA glow
    cast shadows; scrim `--shadow-scrim`. Z-index comes from the `--z-*` scale.
11. **Motion:** use `--motion-*` durations + `--ease-*` (entrances: `--ease-out` 200ms fade +
    translateY; panels/steps 350ms; skeleton shimmer 1300ms). Reduced-motion zeroing is already
    in the tokens. GSAP mapping (if used): fast/normal → `power2.out`, slow → `power2.inOut`,
    loops → `sine.inOut` yoyo.
12. **Focus & selection:** never remove focus styles — `--focus-ring` (2px terracotta, 2px
    offset) everywhere, `--focus-ring-halo` on inputs; `::selection` gold via `--selection-bg`.

**Delivered screen designs → inventory mapping** (open the `.dc.html` files in the design
project as the pixel reference; each ships a working RTL `dir` toggle):

| Design file | Covers (§8) |
|---|---|
| `Dalil Dashboard.dc.html` | Owner home / dashboard patterns (#21 list-scanning, stat tiles) |
| `Dalil Services (archived).dc.html` | #4 Services catalog incl. archived state |
| `Dalil Quotation Builder.dc.html` | #13 builder wizard + #14 milestone grouping |
| `Dalil Start Project.dc.html` | #23 project setup/start |
| `Dalil Project Workspace.dc.html` | #22 project overview / #24 phase detail / #25 board context |
| `Dalil Execution Plan Studio v2.dc.html` | #19 blueprint/plan editing patterns |
| `Dalil Client Portal.dc.html` + `…- Desktop.dc.html` | #36–43 portal surface |
| `Dalil Proposal A4.dc.html` | #15/#37 document frame (§7.11) |
| `Foundations.html`, `DXS-06A/06B/07.html` | token/foundation specimens, shell explorations |

Screens NOT in that list (auth, CRM board, RBAC/settings, approvals, invoicing, platform
console…) have no dedicated design yet: compose them from the DXS foundations + the component
precedents visible inside the delivered screens, per §7/§8 of this brief.

**Open flags for the owner:** standalone logo export (§11.1) · Saudia Sans licence (item F) ·
the component library (§7) has no formal spec sheet yet — the delivered screens are the
precedent until one exists.

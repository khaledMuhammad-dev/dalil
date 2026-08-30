# Dalil — Documentation

Shared documents for both applications. Anything app-specific and operational
(each app's `CLAUDE.md`, agent roster, `_ai-context/` session logs, changelog)
stays inside that app's repo.

## Product

The authoritative product specs. When the PRD and the FS conflict, or either is
silent on a needed behavior, that is a hard stop — it becomes a numbered ruling
in the decision log rather than a silent interpretation.

| Document | File |
| --- | --- |
| Functional Specification v2.0 (49 pp) | [`product/Dalil_Functional_Specification_v2.0.pdf`](product/Dalil_Functional_Specification_v2.0.pdf) |
| Product Requirements Document v1.6 (57 pp) | [`product/Dalil_PRD_v1.6.pdf`](product/Dalil_PRD_v1.6.pdf) |

## Contracts

The HTTP contract is the only coupling between the two apps. The live OpenAPI
spec at `GET /api-docs-json` is authoritative for exact shapes; these two docs
are the maintained human-readable views of it, one per side.

| Document | File |
| --- | --- |
| Frontend contract — the API as its consumers see it | [`contracts/frontend-contract.md`](contracts/frontend-contract.md) |
| Backend contract — how `dalil-web` consumes the API | [`contracts/backend-contract.md`](contracts/backend-contract.md) |

## Launch pack (`fable-5/`)

The full-depth launch briefing produced from PRD v1.6 + FS v2.0. Read
[`fable-5/04-AI-INSTRUCTIONS.md`](fable-5/04-AI-INSTRUCTIONS.md) every session —
it is the permanent operating manual for the whole project.

| Document | File |
| --- | --- |
| Launch-pack overview | [`fable-5/00-README.md`](fable-5/00-README.md) |
| PRD at launch | [`fable-5/01-PRD-launch.md`](fable-5/01-PRD-launch.md) |
| FS amendments at launch (FS-L1…L19) | [`fable-5/02-FS-launch.md`](fable-5/02-FS-launch.md) |
| Gap analysis | [`fable-5/03-GAP-ANALYSIS.md`](fable-5/03-GAP-ANALYSIS.md) |
| Permanent operating manual | [`fable-5/04-AI-INSTRUCTIONS.md`](fable-5/04-AI-INSTRUCTIONS.md) |
| Frontend build guide (slices 0–8) | [`fable-5/05-FRONTEND-START.md`](fable-5/05-FRONTEND-START.md) |
| Launch checklist | [`fable-5/06-LAUNCH-CHECKLIST.md`](fable-5/06-LAUNCH-CHECKLIST.md) |
| Post-launch roadmap | [`fable-5/07-POST-LAUNCH-ROADMAP.md`](fable-5/07-POST-LAUNCH-ROADMAP.md) |
| UI design brief | [`fable-5/08-UI-DESIGN-BRIEF.md`](fable-5/08-UI-DESIGN-BRIEF.md) |

## Decisions

Append-only rulings, referenced by number from code and agent prompts. Never
rewrite a past entry — supersede it with a new one.

| Log | Numbering | File |
| --- | --- | --- |
| Backend / global | `D-xxx` | [`decisions/api.md`](decisions/api.md) |
| Frontend | `FE-D-xxx` | [`decisions/web.md`](decisions/web.md) |

Global `D-xxx` rulings bind the frontend too where applicable.

## Factory

| Document | File |
| --- | --- |
| How the backend's supervised multi-agent build pipeline works | [`factory/FACTORY-GUIDE.md`](factory/FACTORY-GUIDE.md) |

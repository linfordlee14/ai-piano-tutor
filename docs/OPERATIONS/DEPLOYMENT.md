# Deployment

## 1. Purpose

How AI Piano Tutor gets built and deployed: which Firebase services host which parts of the system, the recommended build workflow from scaffolding through precision engineering, and the deployment mechanics for Firestore rules/indexes alongside the app itself.

Source of truth: `PRODUCT_SPEC.md` §16 (Firebase Architecture) and §35 (Build Instructions for Google AI Studio / Antigravity).

## 2. Scope

Deployment targets, the recommended scaffold-to-production workflow, and open questions specific to deployment (Firebase project region). Excludes environment variable/secret specifics (`ENVIRONMENT_CONFIGURATION.md`), post-deploy monitoring (`MONITORING.md`), and incident/rollback procedure (`INCIDENT_RESPONSE.md`).

## 3. What Deploys Where

| Component | Deploys to | Why | Source |
|---|---|---|---|
| Next.js app (`apps/web`) | **Firebase App Hosting** | Next.js-native — supports SSR/framework-aware builds directly, avoiding a hand-rolled Cloud Run deployment for the Next.js server | `PRODUCT_SPEC.md` §16 |
| Cloud Functions (`functions/`) | **Cloud Functions, 2nd gen**, deployed alongside the app | The only component allowed to call Gemini with real credentials; also hosts scheduled jobs | `PRODUCT_SPEC.md` §16 |
| Firestore Security Rules (`firestore.rules`) | Deployed via the **Firebase CLI** | Default-deny, explicit-allow rules must ship in lockstep with any schema/access change — never left stale | `PRODUCT_SPEC.md` §19, §27 (file present in project root) |
| Firestore Indexes (`firestore.indexes.json`) | Deployed via the **Firebase CLI** | See `docs/DATA/FIRESTORE_INDEXES.md` for the one MVP composite index | `PRODUCT_SPEC.md` §17, §27 |
| Lesson content seed (`content/lessons/` → `lessons/{lessonId}`) | A **seed script**, run separately from continuous deployment | Static JSON loaded into Firestore, not hand-authored per-document; this is a repeatable data-loading step, not something the Firebase CLI's app/rules/indexes deploy targets handle automatically | `PRODUCT_SPEC.md` §17, §27 |
| Piano audio samples (Salamander Grand Piano, trimmed) | Static assets on the **same CDN as the app** — explicitly **not** Firebase Storage | Public, immutable files; Storage would add an auth-rule and latency layer for zero benefit | `PRODUCT_SPEC.md` §1 (pushback table), §12, §16 |
| App Check | Configured on Firestore and the Gemini-calling Functions, from initial deployment | "Cheap to add early, painful to retrofit after abuse shows up" | `PRODUCT_SPEC.md` §16 |

## 4. Recommended Build Workflow

`PRODUCT_SPEC.md` §35 gives a specific, sequenced recommendation — reproduced here as the operational workflow, not restated as architecture:

1. **Scaffold with Google AI Studio Build** (or Antigravity). Prompt it with `PRODUCT_SPEC.md` §16–20 (Firebase architecture, data model, auth, security, API shape) to provision Firestore, Auth, Functions, and App Hosting correctly from the start.
2. **Generate a first-draft UI shell** for the screens in §10, in AI Studio Build or Antigravity. Treat this output as a draft establishing routes and component boilerplate — not the piano's timing-sensitive internals.
3. **Move the repo to GitHub immediately.** The browser-based builder must never become the system of record for the codebase — both tools are under a year old in their current form and evolving fast (Firebase Studio's own sunset in the same year is the concrete evidence for this risk — §1, §30, §35).
4. **Hand the precision work to Claude Code**, in a real repo: `packages/music-engine`, Firestore security rules, the Cloud Functions Gemini proxy and scoring logic, and the full test suite. `PRODUCT_SPEC.md` §35 calls this "the most important instruction in this section" — AI Studio/Antigravity are for scaffolding and Firebase wiring, not for timing-sensitive audio code or security rules that need to be exactly right.
5. **Build milestone by milestone** (`PRODUCT_SPEC.md` §28), feeding the relevant section(s) of the spec as context for each milestone in order — not the whole 35-section spec at once, which "produces a shallow implementation of everything instead of a solid one of the MVP."
6. **Treat lesson content authoring as real work**, driven by the founder (Gemini-assisted drafting is fine), not left to a coding agent's judgment about what a beginner needs first — it's static JSON, but pedagogically sequencing 12–15 beginner lessons is content design.

## 5. Open Question — Firebase Project Region (PROPOSED, Needs Verification)

**`PRODUCT_SPEC.md` never specifies a Firebase project region anywhere in the document.** This is a real gap, not an oversight to route around silently: region selection affects Firestore and Cloud Functions latency, and for a South Africa-based founder building for a South African userbase (`PRODUCT_SPEC.md` §18's POPIA framing confirms this is the primary market), it should be a deliberate choice made once — Firestore's location setting cannot be changed after a database is created; it must be re-created and data migrated to change it.

**Proposal (PROPOSED — not verified against current Firebase region availability):** at implementation time, check the current list of available Firestore and Cloud Functions regions for the one with the lowest latency to South Africa. As general background at the time of this document, Firebase/Google Cloud has historically had no Africa region, and `europe-west1` (Belgium) has often been cited as a practical closest option for South Africa-based latency — but **this is explicitly not verified as current fact and must not be treated as a decision.** Region availability and relative latency both change; confirm against Firebase's own current region list and, ideally, a real latency check from South Africa before locking this in.

**Categorize this as needing resolution before first production deploy** — not before development/scaffolding begins (the Emulator Suite and even a default-region dev project work fine for that), but before any real user's data is ever written, given how costly a post-launch region migration would be.

## 6. Dependencies

- A Firebase project must exist and be linked before any of the above can deploy — see `docs/OPERATIONS/ENVIRONMENT_CONFIGURATION.md` for the recommended dev/staging/prod project split.
- The Gemini API credential must be provisioned as a Cloud Functions secret (Secret Manager) before Functions referencing it can deploy successfully — `PRODUCT_SPEC.md` §19; see `ENVIRONMENT_CONFIGURATION.md`.
- App Check must be configured before (or, at latest, immediately upon) first deployment of the Gemini-calling Functions — `PRODUCT_SPEC.md` §16.

## 7. Out of Scope

- Environment variable and secret specifics — `docs/OPERATIONS/ENVIRONMENT_CONFIGURATION.md`.
- Post-deploy monitoring and alerting — `docs/OPERATIONS/MONITORING.md`.
- Incident/rollback procedure (e.g. a security-rules regression) — `docs/OPERATIONS/INCIDENT_RESPONSE.md`.
- Actual CI/CD pipeline configuration (GitHub Actions YAML, etc.) — not authored in this documentation batch; this document describes the deployment targets and workflow, not a checked-in pipeline definition.

# Environment Configuration

## 1. Purpose

What environment variables and per-environment configuration AI Piano Tutor needs: which values are safe to be public, which are secrets, the recommended environment split, and the local development environment.

## 2. Scope

Configuration and environment strategy only. Excludes deployment mechanics (`DEPLOYMENT.md`) and the detailed secrets-handling process (`docs/SECURITY/SECRETS_MANAGEMENT.md`, parallel batch — referenced, not duplicated, here).

## 3. Firebase Client Configuration — Public, Not a Secret

Firebase's own security model treats client-side project configuration (API key, auth domain, project ID, storage bucket, messaging sender ID, app ID) as **safe to be public**. This is a real, load-bearing characteristic of the platform, not a gap: it identifies which Firebase project the client talks to, nothing more. Security is enforced at the Firestore Security Rules and App Check layers (`PRODUCT_SPEC.md` §16, §19), not by hiding this configuration — it ships in the client bundle by necessity, in `NEXT_PUBLIC_*` environment variables (or equivalent).

This stands in deliberate contrast to the Gemini API credential (§4) — the two categories must not be handled the same way, and conflating them is exactly the mistake `PRODUCT_SPEC.md` §19 and §29 are explicit about preventing.

## 4. Gemini API Credential — Server-Side Secret, Never Client-Exposed

`PRODUCT_SPEC.md` §19, stated as a rule: *"Gemini credentials: never in client code, never in a public repo, never in a `NEXT_PUBLIC_*` env var. Stored as a Cloud Functions secret (Secret Manager), read only inside the Function's server-side execution."*

`PRODUCT_SPEC.md` §29 (Acceptance Criteria, Security) makes this a tested, CI-enforced requirement, not a code-review habit: *"No Gemini credential appears anywhere in the built client bundle — grepped as a CI check, not a code-review habit."*

**Cross-reference `docs/SECURITY/SECRETS_MANAGEMENT.md`** (parallel batch — not yet written at the time of this document) for the full secrets-handling process; this document states the environment-configuration-level rule (where the credential lives, what it must never touch), not the operational handling procedure.

## 5. Recommended Environment Split — RECOMMENDATION, Not Extracted Fact

**`PRODUCT_SPEC.md` does not specify a dev/staging/production environment strategy anywhere in the document.** This is a genuine gap, not something to infer from an adjacent section.

**Recommendation (standard practice, marked as such — not a spec-derived requirement):** maintain separate Firebase projects for development, staging, and production. Each Firebase project is a fully isolated set of Firestore data, Auth users, and Cloud Functions — this is the standard mechanism for preventing local/test activity (including AI Studio/Antigravity scaffolding experiments, per `DEPLOYMENT.md` §4) from ever touching real user data or spending real Gemini budget. At minimum:

| Environment | Purpose | Notes |
|---|---|---|
| Development | Local iteration, primarily via the Firebase Emulator Suite (§6) | A cloud dev project is still useful for anything the emulator can't cover (e.g. a real Gemini call, App Check against real traffic) |
| Staging (optional at MVP scale, recommended before real user traffic) | Pre-production verification against real Firebase services, isolated from production data | Could be deferred past M0–M2 (`PRODUCT_SPEC.md` §28) if solo-founder velocity matters more early on — but should exist before the security/launch-readiness milestone (M5) |
| Production | Real user data | The only project with production Gemini spend and real learner PII |

This recommendation is made because `PRODUCT_SPEC.md` is silent, not because it's derived from the spec — flag it for explicit founder sign-off rather than treating it as settled.

## 6. Local Development — Firebase Emulator Suite

`PRODUCT_SPEC.md` §26 names this explicitly for testing: *"Integration tests: Cloud Functions run against the Firebase Local Emulator Suite (Firestore + Functions + Auth), covering the write-only-via-Function pattern and security-rule behavior without touching production data."*

The Emulator Suite (Firestore + Functions + Auth, at minimum) is the recommended local development environment generally, not just the test-runner target — it lets engineers iterate on security rules, Cloud Function logic, and data model changes without touching any real cloud project or incurring any Firebase cost.

**Gap worth noting (implementation clarification, not spec fact):** the Gemini API itself has no local emulator — it's an external Google-hosted API, not a Firebase service the Emulator Suite can stand in for. Local development against `getTutorResponse`/`getExerciseFeedback` therefore needs either (a) a real (ideally low-budget or usage-capped) Gemini credential configured for the dev environment, or (b) a mocked Gemini response layer for engineers iterating purely on UI/UX who don't need real model output. `PRODUCT_SPEC.md` does not address this; flagged here as a practical dev-environment decision left to the implementation team.

## 7. Environment Variables — Summary

Illustrative categories, not a checked-in `.env` file:

| Category | Example | Public or Secret | Consumed by | Source |
|---|---|---|---|---|
| Firebase client config | project ID, API key, auth domain, etc. | Public (`NEXT_PUBLIC_*`) | `apps/web` client bundle | `PRODUCT_SPEC.md` §16; Firebase's own platform model |
| Gemini API credential | API key / service account reference | **Secret** — Cloud Functions secret via Secret Manager, never `NEXT_PUBLIC_*` | `functions/src/ai/*` only, server-side | `PRODUCT_SPEC.md` §19 |
| Firebase Admin SDK credentials | service account (usually implicit via the Functions runtime, not a manually-set env var) | Secret / platform-managed | Cloud Functions runtime | `PRODUCT_SPEC.md` §18 ("every Cloud Function verifies the caller's Firebase ID token via the Admin SDK") |
| Environment identifier | e.g. which Firebase project (dev/staging/prod) is active | Not sensitive, but must not be hardcoded per §5's recommended split | Build/deploy tooling | This document, §5 (recommendation) |

## 8. Out of Scope

- The operational secrets-handling process (rotation, who has access, storage mechanics beyond "Secret Manager") — `docs/SECURITY/SECRETS_MANAGEMENT.md` (parallel batch).
- Deployment targets and workflow — `docs/OPERATIONS/DEPLOYMENT.md`.
- Region selection — `docs/OPERATIONS/DEPLOYMENT.md` §5 (flagged there as an open question).

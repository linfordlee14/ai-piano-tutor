# Firebase Architecture

**Status:** Draft v1.0 — architecture documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §16 (primary), also §1, §17, §18, §20, §22, §23
**Companion documents:** `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md`, `docs/ARCHITECTURE/SECURITY_ARCHITECTURE.md`, `docs/DATA/FIRESTORE_SCHEMA.md` (written in parallel — full field-by-field schema and indexes), `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`

## 1. Purpose

Explain which Firebase/Google Cloud services this project uses, what each one is responsible for, why Firebase Storage is deliberately *not* used for piano audio, and why this project runs on Firestore/Firebase at all when the org's default stack elsewhere is Supabase/Postgres.

## 2. Scope

In scope: the Firebase service table, the rationale for each inclusion and the one notable exclusion (Storage), the Firestore-vs-Postgres deviation callout, and how the callable-function API surface fits together.

Out of scope: literal Firestore Security Rules syntax (→ `SECURITY_ARCHITECTURE.md` for philosophy, `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` for syntax), the full field-by-field Firestore schema (→ `docs/DATA/FIRESTORE_SCHEMA.md`), Gemini call internals (→ `AI_ARCHITECTURE.md`).

## 3. Architecture

### 3.1 Service table

Reproduced and expanded from `docs/PRODUCT_SPEC.md` §16:

| Service | Role | Why |
|---|---|---|
| **Firebase Authentication** | Identity | Anonymous try-first, Google Sign-In, email/password (`docs/PRODUCT_SPEC.md` §18) |
| **Firestore** | Primary datastore | User profiles, progress, lessons, sessions, AI memory doc (`docs/PRODUCT_SPEC.md` §17) |
| **Cloud Functions (2nd gen)** | Gemini proxy + scheduled jobs | The only component allowed to call Gemini with real credentials (`AI_ARCHITECTURE.md`, `SECURITY_ARCHITECTURE.md`) |
| **Firebase App Hosting** | Deployment | Next.js-native — supports SSR/framework-aware builds directly, avoiding a hand-rolled Cloud Run deployment for the Next.js server |
| **App Check** | Abuse prevention | Enabled from day one on Firestore *and* the Gemini-calling Functions — cheap to add early, painful to retrofit after abuse shows up |
| **Firebase Storage** | **Not used for piano audio** | Reserved for genuinely user-generated/private assets if they exist later (e.g. a future avatar upload) — none are needed in MVP; see §3.2 |

### 3.2 Why not Storage for piano audio

Piano samples (`AUDIO_ARCHITECTURE.md`) are **public, immutable files** — every learner loads the identical Salamander Grand Piano sample set, nothing about them is per-user or ever changes at runtime. Firebase Storage is designed for exactly the opposite shape of asset: private-by-default, access-controlled per object, mutable. Routing static samples through Storage would add:
- An **auth-rule evaluation** on every sample fetch, for content that never needs one — the samples are meant to be world-readable.
- **Per-request billing** characteristics suited to occasional private-file access, not to an asset class the app wants aggressively edge-cached.
- A **latency layer** (Storage's access-check path) between the learner and a file that a CDN would otherwise serve directly from the nearest edge node.

Static files that are public and immutable belong on a CDN, served alongside the rest of the app's static assets — not behind an auth-rule layer built for private, user-scoped content. This is a deliberate exclusion (`docs/PRODUCT_SPEC.md` §1, §16), not an oversight: Storage remains available and correctly scoped for a genuinely different future need (e.g., a user-uploaded avatar), which is why it stays in the service table as "not used for piano audio" rather than "not used at all."

### 3.3 The Firestore/Postgres deviation — flagged explicitly, not a silent drift

**This project's default stack elsewhere is React/Next.js/TypeScript/Python with Supabase/Postgres, not Firebase/Firestore.** That is not a contradiction here — it is a deliberate, named exception. `docs/PRODUCT_SPEC.md` §16 states the actual justification directly: **Google AI Studio's build flow provisions Firebase resources natively**, and that build-tooling fit is the reason Firebase is used on *this* project, not a general stack shift. This document repeats the callout because it is exactly the kind of thing that quietly becomes an unexamined default if it isn't stated somewhere load-bearing: whoever maintains this project going forward should know they are working in a document-model datastore here, deliberately, alongside Postgres-based projects elsewhere in the same portfolio — not because Firestore was independently judged the better long-term choice for this kind of data, but because it matched the scaffolding tool this project's build workflow was built around (`docs/PRODUCT_SPEC.md` §35).

### 3.4 Collections — what's in Firestore and what deliberately isn't

Full schema: `docs/DATA/FIRESTORE_SCHEMA.md`. At the architecture level, the shape that matters:

| Collection/path | Shape | Notes |
|---|---|---|
| `users/{uid}` | `UserProfile` (embeds `ProgressSummary`, `settings`) | One doc per user; see `SECURITY_ARCHITECTURE.md` for the field-level write-ownership split within this single document |
| `lessons/{lessonId}` | `Lesson` | Public-read, client-write blocked entirely; seeded via script from static JSON, not hand-authored per-doc |
| `users/{uid}/practiceSessions/{sessionId}` | `PracticeSession` | Append-only; server-written only |
| `users/{uid}/aiMemory/summary` | `AIMemorySummary` | Single doc, rewritten in place; server-written only |

**Deliberately cut from the brief's original list for MVP** (`docs/PRODUCT_SPEC.md` §17), with the architectural reasoning for each:
- **`exercises` as a Firestore collection** — drill definitions (note-ID pool, scale/chord lists) rarely change and don't need a database round-trip; shipped as static app config instead (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.3). Only lesson *content* lives in Firestore, because that's what a future admin CMS (Phase 3) would edit without a deploy.
- **A separate `achievements` collection** — MVP only has streak + XP, both already fields on `UserProfile`. No collection for data that doesn't exist yet.
- **`learningPaths` as its own collection** — with one linear beginner path, `Lesson.level`/`Lesson.prerequisiteLessonId` already encode it (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2). A real collection is warranted only once there's a second path to choose between (e.g. a "rusty returner" fast-track).
- **`songs`, `AIConversations`** — Phase 2+; no schema committed yet, since the song-licensing approach isn't settled.

### 3.5 Indexes

A single composite index on `practiceSessions` for `(userId, completedAt desc)` covers both the Progress/Stats screen and the "most recent session" read `getTutorResponse` performs when assembling `recentExerciseContext` (`DATA_FLOW.md` §5.2, step 3c). Firestore's automatic single-field indexes handle the rest at this scale. Full index configuration: `docs/DATA/FIRESTORE_SCHEMA.md` / `firestore.indexes.json`.

## 4. Data structures

Owned by `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` and `docs/DATA/FIRESTORE_SCHEMA.md`. This document's contribution is §3.4's table — which collection holds which type, and which collections were deliberately not created.

## 5. Interfaces / contracts

No public REST API in MVP (`docs/PRODUCT_SPEC.md` §20). Two access patterns:
- **Client → Firestore, directly** (SDK, realtime `onSnapshot` listeners) — for reads only, on collections/fields the client is allowed to read per `SECURITY_ARCHITECTURE.md`.
- **Client → Cloud Functions callable functions** — for anything requiring server logic or secrets. Full list: `SYSTEM_ARCHITECTURE.md` §6.

**Callable over raw HTTPS functions, specifically:** callables handle auth-context passing and CORS automatically — less boilerplate, fewer footguns, consistent with the "don't add complexity without a reason" principle (`docs/PRODUCT_SPEC.md` §20).

**Versioning:** not needed at this scale — one client version deployed alongside its Functions. Revisit only if a public API or third-party integration appears; none is planned (`docs/PRODUCT_SPEC.md` §20).

## 6. Dependencies

- Firebase Authentication, Firestore, Cloud Functions (2nd gen), Firebase App Hosting, App Check — all first-party Firebase/Google Cloud services, no third-party equivalents layered in.
- Secret Manager (Google Cloud, accessed from Cloud Functions) — holds the Gemini credential (`SECURITY_ARCHITECTURE.md`).
- Firebase's built-in Google Analytics integration for MVP funnel/engagement tracking (`docs/PRODUCT_SPEC.md` §25) — not a core architectural dependency, noted here for completeness since it's part of the same Firebase project.

## 7. Security considerations

Full treatment: `SECURITY_ARCHITECTURE.md`. At the Firebase-service level specifically:
- **App Check** is enabled on Firestore *and* the Gemini-calling Functions from day one (§3.1) — cheap now, painful to retrofit after abuse shows up.
- **Server-side trust:** every Cloud Function verifies the caller's Firebase ID token via the Admin SDK — a client-supplied `uid` is never trusted on its own (`docs/PRODUCT_SPEC.md` §18).
- **Anonymous users are real users:** anonymous sign-in still issues a real Firebase UID, so the same security rules and Function-side checks apply uniformly — there's no separate "unauthenticated" code path to accidentally leave unguarded (`docs/PRODUCT_SPEC.md` §19).

## 8. Performance considerations

- **Flattened `ProgressSummary`** as a field on `UserProfile`, not a subcollection — at ~12–15 MVP lessons, one document read covers the whole dashboard instead of N reads; revisit only if lesson count grows enough to matter (`docs/PRODUCT_SPEC.md` §17).
- **Single composite index** (§3.5) rather than ad hoc per-query indexes, keeping index-maintenance overhead low at MVP scale.
- **Batched writes at exercise completion, not per note** (`docs/PRODUCT_SPEC.md` §22) — the Sync Layer's batching behavior (`MUSIC_ENGINE_ARCHITECTURE.md` §4.5) is what keeps Firestore write volume proportional to exercises completed, not notes played.
- **Generous free tiers at MVP scale** (`docs/PRODUCT_SPEC.md` §31) — Firebase costs (Auth/Firestore/Hosting/Functions) are likely near-zero outside Gemini spend at a few hundred early users; check current free-tier limits at build time rather than trusting a fixed number, since they change.

## 9. Error handling

- **Firestore offline persistence** (`enableIndexedDbPersistence`) is enabled for resilience against ordinary network blips — writes made offline are queued and retried automatically by the Firestore SDK; this is closer to "flip a configuration flag" than "build a sync engine" (`docs/PRODUCT_SPEC.md` §23).
- **Degraded-service banner** for anything longer than a blip (`docs/PRODUCT_SPEC.md` §10) — a system-level concern detailed in `SYSTEM_ARCHITECTURE.md` §10.
- **Error contract:** every callable function returns `{ ok: true, data }` or `{ ok: false, error: { code, message } }` (`docs/PRODUCT_SPEC.md` §20) — see `SYSTEM_ARCHITECTURE.md` §6.

## 10. Testing requirements

- **Firebase Local Emulator Suite** (Firestore + Functions + Auth) for integration tests — covering the write-only-via-Function pattern and security-rule behavior without touching production data (`docs/PRODUCT_SPEC.md` §26).
- **Firestore rules unit tests** (`@firebase/rules-unit-testing`) — owned in depth by `SECURITY_ARCHITECTURE.md`, but run against the emulator infrastructure described here.

## 11. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29, Firebase-relevant items:
- [ ] Progress persists across sign-out/sign-in and across devices for the same account.
- [ ] An anonymous try-first session's progress survives linking to a real account.
- [ ] A client cannot write its own `practiceSessions`/progress/AI-memory documents directly (verified by a rules test) — full detail: `SECURITY_ARCHITECTURE.md`.
- [ ] A client cannot write to `lessons` at all.

## 12. Out of scope

- Literal Firestore Security Rules syntax → `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`
- Full field-by-field schema, exact index definitions → `docs/DATA/FIRESTORE_SCHEMA.md`
- Security philosophy (default-deny, score-tampering closure) → `SECURITY_ARCHITECTURE.md`
- Gemini call internals → `AI_ARCHITECTURE.md`
- Cloud Function business logic beyond "which service does it use" → `DATA_FLOW.md`

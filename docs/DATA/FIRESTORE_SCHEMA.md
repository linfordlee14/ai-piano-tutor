# Firestore Schema

## 1. Purpose

Defines the exact, precise Firestore collection and document layout for AI Piano Tutor's MVP: every collection/document path that exists, the shape at each path, and who writes it — plus, equally important, which collections were deliberately *not* created for MVP and why. This is the "what the shape is" document; `DATA_MODEL.md` in this same directory is its "why the shape is this way" companion.

Source of truth: `docs/PRODUCT_SPEC.md` §17 (Firestore Data Model). Field-level types are owned by `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` and are referenced here, not redefined — per that document's own §1 rule ("every other document... must reference the types defined here rather than redefining them").

## 2. Scope

Every Firestore path that exists in the MVP schema, plus the explicit non-existence list. Excludes: full field-by-field type definitions (TYPES_AND_CONTRACTS.md owns these), the read/write authorization matrix (`DATA_ACCESS_RULES.md`), index definitions (`FIRESTORE_INDEXES.md`), retention/deletion policy (`DATA_RETENTION.md`), and conceptual relationships between entities (`DATA_MODEL.md`).

## 3. Conventions

- MIDI note numbers, timestamps, and ID conventions all follow `TYPES_AND_CONTRACTS.md` §3 — not restated here.
- Paths below use Firestore's own path syntax: `collection/{docId}/subcollection/{docId}`.
- "Function-written only" means: written exclusively by a Cloud Function using the Firebase Admin SDK, which is not subject to Firestore Security Rules. See `DATA_ACCESS_RULES.md`.

## 4. Collections Created for MVP

Four Firestore locations exist in the MVP schema. Each entry gives the explicit path, a compact field-name reference (types live in `TYPES_AND_CONTRACTS.md` — the list below is not the canonical type definition), and the write-ownership rule from `PRODUCT_SPEC.md` §17/§19.

### 4.1 `users/{uid}`

**Path:** `users/{uid}`
**Canonical type:** `UserProfile` — `TYPES_AND_CONTRACTS.md` §4.15
**Cardinality:** exactly one document per Firebase Auth `uid`, including anonymous users (anonymous sign-in still issues a real, persistent `uid` — `PRODUCT_SPEC.md` §18/§19; there is no separate "anonymous user" shape).

Top-level fields (see §4.15 for exact TypeScript types and nullability): `uid`, `displayName`, `email`, `isAnonymous`, `createdAt`, `currentLevel`, `currentLessonId`, `streak` (map: `count`, `lastPracticeDate`), `xp`, `settings` (map: `keyboardMapping`, `volume`, `sustainDefault`), `progressSummary` (map, see 4.1.1).

**Write ownership:** `settings.*` is the *only* client-writable field, and only by the owning `uid`. Every other field — including `progressSummary` — is written exclusively by Cloud Functions using the Admin SDK (`PRODUCT_SPEC.md` §17; `TYPES_AND_CONTRACTS.md` §4.15's field-ownership table).

#### 4.1.1 `progressSummary` (embedded map field, not a subcollection)

**Location:** embedded at `users/{uid}.progressSummary` — not a separate document or subcollection.
**Canonical type:** `ProgressSummary` — `TYPES_AND_CONTRACTS.md` §4.7 (`Record<lessonId, LessonProgress>`).

A deliberate flattening: at ~12–15 MVP lessons, one map on the user document costs one Firestore read for the entire dashboard instead of N per-lesson-document reads (`PRODUCT_SPEC.md` §17). Revisit only if lesson count grows enough that the map's size becomes a real concern — no such threshold is defined in `PRODUCT_SPEC.md`; treat this as a "watch it, don't pre-solve it" item.

### 4.2 `users/{uid}/practiceSessions/{sessionId}`

**Path:** `users/{uid}/practiceSessions/{sessionId}`
**Canonical type:** `PracticeSession` — `TYPES_AND_CONTRACTS.md` §4.6
**Cardinality:** append-only log, one document per completed exercise/lesson attempt. `sessionId` is a Firestore auto-ID.

Top-level fields: `id`, `userId`, `lessonId?`, `exerciseId?`, `startedAt`, `completedAt`, `verdict` (embedded `ExerciseVerdict`, not referenced), `deviceType`.

**Write ownership:** written exclusively by the `getExerciseFeedback` Cloud Function via the Admin SDK. The client never writes to this collection directly, under any circumstance (`TYPES_AND_CONTRACTS.md` §4.6) — this is the specific mechanism that closes the "client self-reports a perfect score" cheating vector (`PRODUCT_SPEC.md` §17/§19). Documents are never updated after creation and are not deleted except as part of the full-subtree cascade on account deletion — see `DATA_RETENTION.md`.

### 4.3 `users/{uid}/aiMemory/summary`

**Path:** `users/{uid}/aiMemory/summary`
**Canonical type:** `AIMemorySummary` — `TYPES_AND_CONTRACTS.md` §4.8

**This is a single, fixed-ID document — not a subcollection holding multiple documents.** `aiMemory` is technically a Firestore subcollection (every document lives inside some collection), but by design it holds exactly one document, always at the literal ID `summary`, never `aiMemory/{autoId}`. Do not build tooling or queries that assume more than one document can exist at this path.

Top-level fields: `skillLevel`, `weakNotes[]`, `weakChords[]`, `strengths[]`, `currentGoal`, `lastUpdated`.

**Write ownership:** written exclusively by Cloud Functions, recomputed from recent `practiceSessions` records and rewritten in place each time — never appended to, never a growing transcript (`PRODUCT_SPEC.md` §17/§31; `TYPES_AND_CONTRACTS.md` §4.8). Read by the Tutor Q&A Cloud Function server-side to assemble `TutorRequest` context; never read or written directly by the client.

### 4.4 `lessons/{lessonId}`

**Path:** `lessons/{lessonId}`
**Canonical type:** `Lesson` — `TYPES_AND_CONTRACTS.md` §4.2
**Cardinality:** one document per lesson, ~12–15 for MVP. `lessonId` is a stable authored slug (e.g. `"l03-middle-c"`), not an auto-ID — this is a top-level collection, independent of any user.

Top-level fields: `id`, `title`, `level`, `type`, `expectedSequence[]`, `tempoBpm?`, `instructions`, `prerequisiteLessonId`, `scoringWeights`, `passThreshold`.

**Write ownership:** public-read (`allow read: if true`), write blocked entirely from any client (`allow write: if false`). Content changes only via the Admin SDK — a seed script today, an admin Cloud Function in a Phase 3 CMS (`PRODUCT_SPEC.md` §17/§19). Seeded from static JSON in `content/lessons/` (`PRODUCT_SPEC.md` §17/§27), not hand-authored per-document in the console.

## 5. Collections Deliberately Not Created for MVP

`PRODUCT_SPEC.md` §17 explicitly cuts these from the brief's fuller entity list. Listed here so a future contributor doesn't "rediscover" the need for them without first checking whether the stated reason still holds.

| Collection | Why not created for MVP | Revisit when |
|---|---|---|
| `exercises` | Drill definitions (Note ID pool, scale/chord lists) rarely change and don't need a database round-trip; shipped as static app config instead (`Exercise` type, `TYPES_AND_CONTRACTS.md` §4.3 — explicitly "not persisted in Firestore"). Only lesson *content* needs Firestore, because that's what a future admin CMS would edit without a deploy. | Drill content needs non-deploy editing — the same trigger condition as the `lessons` Admin CMS, Phase 3. |
| `achievements` | MVP only has streak + XP, and both are already fields on `users/{uid}` (`UserProfile.streak`, `UserProfile.xp`). No collection is built for data that doesn't exist yet. | Achievements beyond streak/XP ship (Phase 2, `PRODUCT_SPEC.md` §8). |
| `learningPaths` | One linear MVP path is already encoded via `Lesson.level` + `Lesson.prerequisiteLessonId` (`TYPES_AND_CONTRACTS.md` §4.2), forming a single acyclic chain. A dedicated collection is unnecessary overhead until there's a second path to choose *between*. | A second path exists — `PRODUCT_SPEC.md` §17 names a "rusty returner" fast-track as the plausible first case. |
| `songs` | Phase 2+ (`PRODUCT_SPEC.md` §8). No schema committed because the song-licensing approach (public-domain/original content only, §8/§30) isn't settled yet. | The licensing question is resolved and the song learning module is scoped. |
| `AIConversations` | Phase 2+. No schema committed. Also worth noting structurally: the MVP AI-memory design deliberately keeps one rewritten-in-place summary document (`aiMemory/summary`), never a growing transcript log (§17/§31) — so this isn't just "not built yet," it's a pattern that should be revisited deliberately, not defaulted into a raw conversation-log collection. | Phase 2 conversational features are scoped — and that scoping should explicitly re-examine whether transcript storage is needed at all, or whether the summary-only pattern extends. |

## 6. Indexes

See `FIRESTORE_INDEXES.md` for the full treatment. In brief: one composite index, on `practiceSessions` for `(userId, completedAt desc)`, covers the MVP stats screen; Firestore's automatic single-field indexes cover everything else at this scale (`PRODUCT_SPEC.md` §17).

## 7. Security Considerations

See `DATA_ACCESS_RULES.md` for the full read/write matrix. In brief, from `PRODUCT_SPEC.md` §17's own security preview: everything under `users/{uid}/**` is readable only by that `uid`; `progressSummary`, `practiceSessions/**`, and `aiMemory/**` are written only by Cloud Functions, never directly by the client; `lessons` is public-read and writable only via the Admin SDK.

## 8. Out of Scope

- Full TypeScript field definitions and nullability — owned by `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`.
- The read/write authorization matrix and its exact rules-language mapping — `DATA_ACCESS_RULES.md`.
- Composite index definitions and the process for adding new ones — `FIRESTORE_INDEXES.md`.
- Retention, archival, and account-deletion cascade behavior — `DATA_RETENTION.md`.
- Conceptual relationships between entities and the reasoning behind embed-vs-subcollection-vs-top-level choices — `DATA_MODEL.md`.
- Non-Firestore static config that MVP schema documents reference by ID (`Exercise`, `ChordDefinition`, `ScaleDefinition` — all live in `packages/music-engine` or app config, per `TYPES_AND_CONTRACTS.md` §4.3/§4.12/§4.13) — noted here only because `practiceSessions.exerciseId` can point to one of these; full ownership is engineering-owned static config, not this document's concern.

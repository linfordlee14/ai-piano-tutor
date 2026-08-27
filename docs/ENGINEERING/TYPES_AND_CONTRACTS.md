# Types & Contracts

## 1. Purpose

This document is the **single canonical source** for every shared data shape in AI Piano Tutor. Every other document in `docs/` — architecture, music, AI, data, security, testing, roadmap — must reference the types defined here rather than redefining them. If a document needs a field this file doesn't have, that's a gap: raise it in `docs/OPEN_QUESTIONS.md` rather than inventing a divergent shape locally.

Source of truth for the product decisions these types encode: `docs/PRODUCT_SPEC.md`, sections 11, 13, 15, 17, 18.

## 2. Scope

Covers the fifteen original shared contracts used across the client, `packages/music-engine`, Cloud Functions, and Firestore: `NoteEvent`, `Lesson`, `Exercise`, `ExerciseAttempt`, `ExerciseVerdict`, `PracticeSession`, `ProgressSummary`, `AIMemorySummary`, `TutorRequest`, `TutorResponse`, `KeyboardMapping`, `ChordDefinition`, `ScaleDefinition`, `Score`, `UserProfile` — plus four added in §4.16–4.18 during the documentation QC pass, to formalize shapes the review agents found already in informal use elsewhere in `docs/` but never given a canonical type: `FunctionResult<T>`, `FunctionErrorCode`, `ExerciseFeedbackPromptInput`, `MotivationNudgePromptInput`. See `docs/DECISIONS.md` for the record of each addition/correction made during that pass.

Out of scope: UI component prop types, internal Cloud Function helper types, and anything not shared across a module boundary — those belong with the code that owns them, not here.

## 3. Conventions

- **MIDI note numbers**, not note-name strings, are the canonical representation of pitch everywhere in this codebase. `60` = Middle C (`C4`). Note names are a *derived display value*, computed by `packages/music-engine`, never stored or compared as the source of truth. (`docs/MUSIC/NOTE_SYSTEM.md` owns the derivation rules.)
- **Timestamps** are milliseconds since epoch (`number`), generated with `performance.now()`-relative offsets for in-session timing and `Date.now()`-equivalent absolute values for persisted records. Firestore-persisted timestamps use `Timestamp` (Firebase's native type) at the storage boundary; in-memory/wire representations use `number`.
- **IDs** are `string`. Firestore auto-IDs unless a stable slug is specified (e.g. `lessonId: "l01-find-middle-c"`).
- All types below are written in TypeScript. `packages/shared-types` is the package that owns these as actual `.ts` exports, imported by both `apps/web` and `functions/`.
- **Required vs. optional** follows TypeScript's `?` convention. A field with no `?` must always be present and non-null.

## 4. Contracts

### 4.1 `NoteEvent`

The atomic unit every input adapter (mouse, touch, computer-keyboard, MIDI) normalizes into. Nothing downstream — Engine Core, Audio Layer, UI Layer — ever inspects the input source's raw event type again after this point.

```typescript
interface NoteEvent {
  midiNote: number;        // 0-127, required. Canonical pitch.
  type: 'noteOn' | 'noteOff'; // required. Chord/interval detection needs on/off pairing, not just "a note happened."
  velocity: number;        // 0-127, required. Non-velocity-sensitive sources (mouse, touch, computer keyboard) default to 100.
  timestamp: number;       // required. High-resolution ms, relative to session/exercise start (not wall-clock) — see NOTE_SYSTEM.md.
  source: 'mouse' | 'touch' | 'keyboard' | 'midi'; // required. Diagnostic/analytics only — never branched on for scoring logic.
}
```

**Validation:** `midiNote` must be an integer in `[0, 127]`; reject (log, don't crash) out-of-range values from any adapter, including MIDI hardware that misbehaves. `velocity` integer in `[0, 127]`.

**Ownership:** created client-side, by input adapters, in `apps/web`. Never persisted directly — it's an ephemeral event; what gets persisted is the *result* of feeding a sequence of `NoteEvent`s into the Engine Core (see `ExerciseVerdict`, `PracticeSession`).

**Client/server boundary:** a `NoteEvent[]` is what the client sends to a Cloud Function as part of an `ExerciseAttempt` for server-side re-validation. The server does not trust client-computed verdicts — only the raw `NoteEvent[]` — and recomputes the verdict itself using the same Engine Core code (`packages/music-engine`, imported by both `apps/web` and `functions/`).

---

### 4.2 `Lesson`

A single structured, sequenced learning unit. Content, not user data — lives in Firestore's public-read `lessons` collection, seeded from static JSON (see `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`).

```typescript
interface Lesson {
  id: string;                    // required, stable slug, e.g. "l03-middle-c"
  title: string;                 // required
  level: number;                 // required, integer, defines linear MVP ordering
  type: 'orientation' | 'noteId' | 'octave' | 'melody' | 'rhythm' | 'scale' | 'chord'; // required
  expectedSequence: ExpectedNote[]; // required, may be empty for pure-orientation lessons (e.g. "find Middle C")
  tempoBpm?: number;              // optional — only meaningful for type: 'melody' | 'rhythm'
  instructions: string;           // required, learner-facing copy
  prerequisiteLessonId: string | null; // required (explicit null for the first lesson, not omitted)
  scoringWeights: ScoringWeights;  // required — see Score (4.15); authored per-lesson, never hardcoded in the engine
  passThreshold: number;          // required, 0-1, the compositeScore (as a fraction) needed to unlock the next lesson — see OPEN_QUESTIONS.md OQ-01, no default value has been ratified yet
  hints?: string[];               // optional, ordered, escalating hint text. Added during the documentation QC pass to close a gap against PR-LESSON-05 (escalating hints is an approved MVP requirement; no field previously existed to represent it). Exact tier count and repeat-on-exhaustion behavior are NOT ratified — see OPEN_QUESTIONS.md OQ-02. A lesson authored with no hints omits the field entirely; UI must handle its absence, not assume it's always present.
}

interface ExpectedNote {
  midiNote: number;      // required
  expectedTimeMs?: number; // optional — omitted for lessons where order matters but timing doesn't (e.g. simple note-finding)
}
```

**Validation:** `level` and `prerequisiteLessonId` together must form a single acyclic chain for MVP (no branching paths — see `docs/PRODUCT_SPEC.md` §17's cut of `learningPaths` as a separate collection).

**Ownership:** authored as static seed content (`docs/CONTENT/`), loaded into Firestore by a seed script. Not user-editable. Not client-writable under any circumstance (see `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`).

---

### 4.3 `Exercise`

A drill-mode unit (Note ID game, scale drill, chord drill) — **not** persisted in Firestore. Per `docs/PRODUCT_SPEC.md` §17, exercise *definitions* are static application config bundled with the client, because they rarely change and don't need a database round-trip. Only lesson *content* lives in Firestore.

```typescript
interface Exercise {
  id: string;                          // required
  kind: 'noteId' | 'scaleDrill' | 'chordDrill'; // required
  difficulty: number;                   // required, integer, drives adaptive difficulty (docs/MUSIC/SCORING_SYSTEM.md)
  pool: ExercisePoolItem[];             // required — the set of prompts this exercise can draw from
  scoringWeights: ScoringWeights;       // required
}

interface ExercisePoolItem {
  promptMidiNote?: number;   // for noteId
  chordDefinitionId?: string; // for chordDrill — references ChordDefinition
  scaleDefinitionId?: string; // for scaleDrill — references ScaleDefinition
}
```

**Ownership:** static TypeScript/JSON config in `apps/web` (or a shared package), authored by engineering, not content-ops. Distinct from `Lesson`, which is content-ops-owned static JSON loaded into Firestore.

---

### 4.4 `ExerciseAttempt`

The raw, **untrusted** submission a client sends when a learner finishes an exercise or lesson step. This is input to server-side validation — it is never itself the record of what happened.

```typescript
interface ExerciseAttempt {
  lessonId?: string;        // exactly one of lessonId / exerciseId is present
  exerciseId?: string;
  noteEvents: NoteEvent[];  // required — the full raw sequence, client-timestamped
  startedAt: number;        // required, client clock, informational only (server does not trust it for scoring)
  submittedAt: number;      // required, client clock, informational only
  deviceType: 'mobile' | 'tablet' | 'desktop'; // required, analytics only
  attemptNumber: number;    // required, integer ≥1 — how many times this learner has attempted this lesson/exercise
}
```

**Trust boundary — the important part:** the Cloud Function that receives this (`getExerciseFeedback`, see `docs/AI/GEMINI_SPEC.md` and `docs/ARCHITECTURE/AI_ARCHITECTURE.md`) treats every field as attacker-controlled. It does not trust `attemptNumber`, does not trust that `noteEvents` timestamps are honest, and computes the actual `ExerciseVerdict` itself by re-running `packages/music-engine` server-side against the `Lesson`/`Exercise` definition it independently loads from Firestore/config. A client cannot submit a fabricated verdict — only raw events, which the server re-judges.

---

### 4.5 `ExerciseVerdict`

The deterministic, server-computed result of validating an `ExerciseAttempt` against its `Lesson`/`Exercise` definition. This is the object the brief's worked example describes (`docs/PRODUCT_SPEC.md` §13/§30): `missedNotes: ['F'], accuracy: 0.8`.

```typescript
interface ExerciseVerdict {
  correctNotes: string[];    // required, note names, e.g. ["C","D","E"]
  missedNotes: string[];     // required, note names expected but not played
  extraNotes: string[];      // required, note names played but not expected
  accuracy: number;          // required, 0-1 = correctNotes.length / expectedSequence.length
  timingScore: number;       // required, 0-1 — see docs/MUSIC/SCORING_SYSTEM.md for the bucket formula
  completeness: number;      // required, 0-1 — were all required notes attempted at all
  consistency: number;       // required, 0-1 — variance across this + recent attempts of the same exercise
  perNoteTiming: NoteTimingResult[]; // required, may be empty for non-timed exercise types
  compositeScore: Score;     // required — see 4.15
  passed: boolean;           // required — compositeScore.value/100 >= Lesson.passThreshold or Exercise-equivalent
}

interface NoteTimingResult {
  midiNote: number;
  deviationMs: number;
  bucket: 'onTime' | 'slightlyOff' | 'late' | 'early';
}
```

**Ownership:** computed exclusively by `packages/music-engine`, invoked server-side inside a Cloud Function. **Gemini never produces, modifies, or is asked to sanity-check any field on this object** — see `docs/ARCHITECTURE/AI_ARCHITECTURE.md` §Boundary. Gemini receives a completed `ExerciseVerdict` as read-only input and explains it in prose; it does not participate in producing it.

**Client/server boundary:** the client receives this back from the Cloud Function as the source of truth for what happened. Any client-side "preview" verdict computed optimistically (for instant UI feedback before the network round-trip completes) is explicitly provisional and is overwritten by the server's response — see `docs/ARCHITECTURE/DATA_FLOW.md`.

---

### 4.6 `PracticeSession`

The Firestore-persisted record of one completed exercise/lesson attempt. Append-only.

```typescript
interface PracticeSession {
  id: string;                 // Firestore doc ID
  userId: string;             // required
  lessonId?: string;
  exerciseId?: string;
  startedAt: FirebaseFirestore.Timestamp; // required
  completedAt: FirebaseFirestore.Timestamp; // required
  verdict: ExerciseVerdict;   // required, embedded, not referenced
  deviceType: 'mobile' | 'tablet' | 'desktop'; // required
}
```

**Firestore path:** `users/{uid}/practiceSessions/{sessionId}` — see `docs/DATA/FIRESTORE_SCHEMA.md`.

**Ownership:** written exclusively by Cloud Functions (`getExerciseFeedback`), using the Admin SDK. **The client never writes to this collection directly, under any circumstance** — see `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`. This is the specific rule that closes the "client self-reports a perfect score" cheating vector.

---

### 4.7 `ProgressSummary`

The flattened per-user progress map — a field on the `UserProfile` document, not a subcollection (deliberate cost/read-count optimization, see `docs/PRODUCT_SPEC.md` §17).

```typescript
type ProgressSummary = Record<string, LessonProgress>; // keyed by lessonId

interface LessonProgress {
  status: 'locked' | 'unlocked' | 'completed'; // required
  bestScore: number;    // required, 0-100, defaults to 0 before first attempt
  attempts: number;     // required, integer ≥0
  lastAttemptAt: FirebaseFirestore.Timestamp | null; // required, explicit null before first attempt
}
```

**Ownership:** written exclusively by Cloud Functions, as a side effect of `getExerciseFeedback` recomputing a verdict. Same client/server trust boundary as `PracticeSession`.

---

### 4.8 `AIMemorySummary`

The single structured document that stands in for "AI memory" — deliberately **not** a growing conversation transcript (`docs/PRODUCT_SPEC.md` §17/§31, `docs/AI/AI_CONTEXT_MODEL.md`).

```typescript
interface AIMemorySummary {
  skillLevel: 'beginner' | 'returning'; // required — maps to the two personas in docs/PRODUCT/USER_PERSONAS.md
  weakNotes: string[];      // required, may be empty, note names
  weakChords: string[];     // required, may be empty, chord names
  strengths: string[];      // required, may be empty
  currentGoal: string | null; // required, explicit null if none set
  lastUpdated: FirebaseFirestore.Timestamp; // required
}
```

**Firestore path:** `users/{uid}/aiMemory/summary` — a single document, rewritten in place, never appended to.

**Ownership:** written exclusively by Cloud Functions, recomputed from recent `PracticeSession` records (aggregation logic owned by `functions/src/ai/`). Read server-side by the Tutor Q&A Cloud Function to assemble `TutorRequest` context, **and directly readable client-side, owner-only** (`docs/SECURITY/FIRESTORE_SECURITY_RULES.md` §5.4) — e.g. to power a "what the app has learned about you" display. **Correction made during the documentation QC pass:** this line previously read "never read or written directly by the client," which contradicted both `docs/PRODUCT_SPEC.md` §19 and the shipped `FIRESTORE_SECURITY_RULES.md`, both of which grant owner-only client read on this path. Write access remains Cloud-Functions-only under all circumstances — only the read path was wrong. See `docs/DECISIONS.md` D-01.

---

### 4.9 `TutorRequest`

The **server-assembled** input to the Gemini Tutor Q&A call. The client supplies only `learnerQuestion`; every other field is assembled server-side from Firestore, specifically so a learner cannot fabricate their own performance context to manipulate the tutor's answer.

```typescript
interface TutorRequest {
  learnerQuestion: string;           // required, client-supplied, length-capped (see AI_GUARDRAILS.md)
  recentExerciseContext: ExerciseVerdict | null; // required, server-assembled from the most recent PracticeSession, null if none exists yet
  learnerMemorySummary: AIMemorySummary; // required, server-assembled
  skillLevel: 'beginner' | 'returning'; // required, server-assembled (redundant with learnerMemorySummary.skillLevel for prompt-template convenience)
}
```

**Client/server boundary:** the client-facing Cloud Function signature accepts only `{ learnerQuestion: string }`. `TutorRequest` as a whole is constructed inside the Function, never accepted as a full object from the client — see `docs/AI/AI_GUARDRAILS.md`.

---

### 4.10 `TutorResponse`

```typescript
interface TutorResponse {
  text: string;              // required, length-capped per template (docs/AI/AI_PROMPTS.md)
  suggestedAction?: string;  // optional, e.g. "practice-c-major-scale" — a deep-link hint, not free text
  tier: 'cheapest' | 'mid';  // required, which model tier served this — for cost monitoring, not shown to the user
  cached: boolean;           // required — true if served from the theory-explanation cache (docs/AI/AI_COST_CONTROL.md) rather than a live call
}
```

**Ownership:** returned by the Cloud Function to the client. Never persisted as a growing log (see `docs/AI/AI_CONTEXT_MODEL.md` for what *is* retained — nothing beyond the `AIMemorySummary` aggregate).

---

### 4.11 `KeyboardMapping`

```typescript
type KeyboardMapping = Record<number, string>; // keyed by semitone offset from the mapping's root (0-12), value = physical key label, e.g. { 0: "A", 1: "W", 2: "S", ... }
```

**Default value:** defined in `docs/MUSIC/KEYBOARD_MAPPING.md`. **Ownership:** stored at `users/{uid}.settings.keyboardMapping`, client-writable (it's a preference, not a scored fact) — the one exception to "everything under users/{uid} beyond settings is Function-written only."

---

### 4.12 `ChordDefinition`

```typescript
interface ChordDefinition {
  id: string;               // required, e.g. "major-triad"
  name: string;              // required, display name, e.g. "Major"
  intervals: number[];       // required, semitone offsets from root, e.g. [0, 4, 7] for major
  quality: 'major' | 'minor' | 'diminished' | 'augmented'; // required — MVP ships major + minor only, others defined but unused until Phase 2
}
```

**Ownership:** static config in `packages/music-engine`, not Firestore (this is engine logic, not content). See `docs/MUSIC/CHORD_SYSTEM.md` for the full interval table.

---

### 4.13 `ScaleDefinition`

```typescript
interface ScaleDefinition {
  id: string;                // required, e.g. "major"
  name: string;               // required, e.g. "Major"
  intervals: number[];        // required, e.g. [0,2,4,5,7,9,11] for major
}
```

**Ownership:** same as `ChordDefinition` — static engine config, see `docs/MUSIC/SCALE_SYSTEM.md`.

---

### 4.14 `Score`

The composite scoring shape, embedded inside `ExerciseVerdict` and reused wherever a single scalar-plus-breakdown score is needed (dashboard display, `PracticeSession.verdict.compositeScore`).

```typescript
interface Score {
  value: number;   // required, 0-100
  breakdown: {
    correctness: number; // 0-1, weighted contribution
    timing: number;      // 0-1
    consistency: number; // 0-1
  };
}

interface ScoringWeights {
  correctness: number; // required, 0-1, sums with timing+consistency to 1.0
  timing: number;       // required, 0-1
  consistency: number;  // required, 0-1
}
```

**Validation:** `ScoringWeights` fields **must** sum to `1.0` (±0.001 float tolerance) — a Lesson/Exercise seed file that fails this check should fail content validation at build/seed time, not at runtime. See `docs/MUSIC/SCORING_SYSTEM.md` for the formula and `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` for the validation step.

---

### 4.15 `UserProfile`

```typescript
interface UserProfile {
  uid: string;                 // required, Firebase Auth UID, doc ID
  displayName: string | null;   // required, explicit null for anonymous users pre-linking
  email: string | null;         // required, explicit null for anonymous users
  isAnonymous: boolean;         // required
  createdAt: FirebaseFirestore.Timestamp; // required
  currentLevel: number;          // required, integer, mirrors furthest-unlocked Lesson.level
  currentLessonId: string;       // required
  streak: {
    count: number;               // required, integer ≥0
    lastPracticeDate: string;    // required, ISO date (YYYY-MM-DD), not a full timestamp — streak logic is calendar-day based
  };
  xp: number;                    // required, integer ≥0
  settings: {
    keyboardMapping: KeyboardMapping; // required
    volume: number;                    // required, 0-1
    sustainDefault: boolean;           // required
  };
  progressSummary: ProgressSummary;    // required — see 4.7; embedded on this same document, not a separate read
}
```

**Firestore path:** `users/{uid}` — see `docs/DATA/FIRESTORE_SCHEMA.md`.

**Field-level ownership** (this is the table every security-rule and Cloud Function author should check first):

| Field | Client-writable? | Written by |
|---|---|---|
| `settings.*` | Yes | Client (own uid only) |
| everything else | No | Cloud Functions (Admin SDK) only |

---

### 4.16 `FunctionResult<T>`

The canonical wire-format envelope for every callable Cloud Function response. `docs/PRODUCT_SPEC.md` §20 establishes the `{ ok: true, data }` / `{ ok: false, error }` shape, and `docs/ENGINEERING/ERROR_HANDLING.md` §4 works out the full error-code taxonomy in prose — but until this QC pass, no document exported it as one named, importable type. **The type and every code name below are taken verbatim from `docs/ENGINEERING/ERROR_HANDLING.md` §4.1/§4.3, which is the canonical source for this taxonomy; this entry only formalizes it as an actual `packages/shared-types` export.**

```typescript
type FunctionResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: FunctionErrorCode; message: string } };

type FunctionErrorCode =
  | 'UNAUTHENTICATED'
  | 'VALIDATION_FAILED'
  | 'RATE_LIMITED'
  | 'GEMINI_UNAVAILABLE'
  | 'NOT_FOUND'
  | 'INTERNAL';
```

**Ownership:** every callable Cloud Function (`getExerciseFeedback`, `getTutorResponse`, `deleteAccount`, `exportUserData`, and any future callable) returns `FunctionResult<T>` — never a bare payload, and never a thrown HTTPS error for an *expected* failure mode (rate limit, validation). Thrown errors are reserved for genuinely unexpected/internal failures the client can't act on specifically. `FunctionErrorCode` is a closed union; a Function must not invent an ad hoc string code outside this list. **This list is a first pass, not exhaustive** — an implementation that needs a code not listed here should extend this union (in both this file and `ERROR_HANDLING.md` §4.3 together, never one without the other), not stringly-type around it. See `docs/ENGINEERING/ERROR_HANDLING.md` §6 for the full code → UI-state mapping and client-side handling per code. **Naming correction made during the QC pass:** an earlier draft of this entry used `INVALID_ARGUMENT` and `AI_UPSTREAM_ERROR`, which didn't match `ERROR_HANDLING.md`'s already-established `VALIDATION_FAILED` and `GEMINI_UNAVAILABLE` for the same two concepts — renamed to match exactly, and `docs/AI/AI_ERROR_HANDLING.md`'s independently-proposed `RATE_LIMIT_EXCEEDED` was renamed to `RATE_LIMITED` for the same reason. See `docs/DECISIONS.md` D-04.

---

### 4.17 `ExerciseFeedbackPromptInput`

The Exercise Feedback template's rendered-prompt input shape (`docs/AI/AI_PROMPTS.md` §2, `docs/AI/GEMINI_SPEC.md` §5). Previously existed only as an identical JSON shape copy-pasted across `docs/PRODUCT_SPEC.md` §15, `docs/ARCHITECTURE/DATA_FLOW.md`, `AI_PROMPTS.md`, and `GEMINI_SPEC.md` — formalized here per `GEMINI_SPEC.md` §12's explicit recommendation.

```typescript
interface ExerciseFeedbackPromptInput {
  exerciseType: string;      // Lesson.type or Exercise.kind of the definition just scored
  expected: string[];        // note names, derived from the Lesson/Exercise definition
  played: string[];          // note names, derived from the attempt's noteEvents, in played order
  accuracyScore: number;     // = ExerciseVerdict.accuracy — renamed for prompt-input clarity, same value
  missedNotes: string[];     // = ExerciseVerdict.missedNotes, verbatim
  attemptNumber: number;     // = ExerciseAttempt.attemptNumber, informational only
  skillLevel: 'beginner' | 'returning'; // = AIMemorySummary.skillLevel. Canonical name is skillLevel, not learnerLevel — docs authored before this pass use learnerLevel for the same field; treat the two as synonyms for this one canonical field, and prefer skillLevel in new code.
}
```

**Note on the brief's illustrative naming:** `AI_PROMPTS.md` §2.3's worked example uses `exercise`/`accuracy` (the original product brief's own field names) rather than `exerciseType`/`accuracyScore` — both name the same values; the discrepancy is cosmetic and pre-dates this type's formalization. Implementers should use `ExerciseFeedbackPromptInput`'s field names going forward.

**Ownership:** assembled server-side, inside `getExerciseFeedback`, immediately after `packages/music-engine` computes the `ExerciseVerdict` for the same attempt — never accepted from the client.

---

### 4.18 `MotivationNudgePromptInput`

The Motivation/Nudge template's input shape (`docs/AI/AI_PROMPTS.md` §5, `docs/AI/GEMINI_SPEC.md` §8) — formalized here for the same reason as 4.17.

```typescript
interface MotivationNudgePromptInput {
  streak: number;                  // = UserProfile.streak.count
  recentMilestone: string | null;  // e.g. "level-2-complete"; null if the trigger was a streak-count change with no named milestone
}
```

**Ownership:** assembled server-side, on the rare pool-miss path only (`GEMINI_SPEC.md` §8) — most nudges never reach this type at all, served instead from the pre-written pool.

## 5. Cross-Cutting Rules

1. Every contract above that is **server-written-only** is enforced two ways, redundantly: Firestore Security Rules deny the write path entirely (`docs/SECURITY/FIRESTORE_SECURITY_RULES.md`), and Cloud Functions independently re-validate/recompute rather than trusting any client-supplied verdict. Neither mechanism alone is sufficient — rules-only leaves the Function itself as an unverified trust boundary if compromised; Function-only leaves a gap if a rules change regresses. Both must hold.
2. `packages/music-engine` (which owns `NoteEvent`, `ChordDefinition`, `ScaleDefinition`, and the verdict-computation logic behind `ExerciseVerdict`) must have **zero import** of `firebase/*`, `react`, `next`, or any Gemini/AI SDK. It is a pure function of `(NoteEvent[], Lesson | Exercise) → ExerciseVerdict`. See `docs/MUSIC/MUSIC_ENGINE_SPEC.md`.
3. Gemini-facing types (`TutorRequest`, `TutorResponse`) never contain a raw `NoteEvent[]` — only already-summarized `ExerciseVerdict`/`AIMemorySummary` data. This is deliberate: Gemini should never be in a position to "recompute" anything from raw note data.

## 6. Out of Scope

- Component prop types, internal Cloud Function request/response wire formats beyond what's specified here, and Firestore document types not listed above (there are none planned for MVP beyond what's in `docs/DATA/FIRESTORE_SCHEMA.md`).
- Exact validation library/schema-enforcement mechanism (Zod, io-ts, etc.) — a choice for `docs/ENGINEERING/CODING_STANDARDS.md` to make, not this document.

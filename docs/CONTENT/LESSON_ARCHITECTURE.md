# Lesson Architecture

**Status:** Draft v1.0 — content documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §9, §10, §17; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 (`Lesson`), §4.7 (`ProgressSummary`/`LessonProgress`), §4.4 (`ExerciseAttempt`)
**Companion documents:** `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` (the JSON seed-file schema implementing this architecture), `docs/CONTENT/BEGINNER_CURRICULUM.md` (the actual MVP lesson list), `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`, `docs/MUSIC/MUSIC_ENGINE_SPEC.md` §5.2 (the empty-`expectedSequence` orientation-lesson case this document's completion model depends on), `docs/ARCHITECTURE/DATA_FLOW.md` §4 (server-side write path for progress), `docs/ARCHITECTURE/AUDIO_ARCHITECTURE.md` (demonstration playback)

---

## 1. Purpose

Define how a `Lesson` moves a learner from locked to unlocked to completed: the exact unlocking algorithm, how demonstrations and hints fit into the core interaction loop, and how attempts are tracked. This document is the architectural companion to `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` (which specifies the JSON shape) and `docs/CONTENT/BEGINNER_CURRICULUM.md` (which specifies the actual MVP lesson sequence) — this document answers "how does a lesson behave," those answer "what does a lesson JSON file look like" and "which 14 lessons exist."

## 2. Scope

In scope: the `Lesson` type as consumed by the Lesson Player, lesson states and the unlocking algorithm, prerequisite chaining, the completion requirement, demonstration playback, the hint system's architecture, and attempt tracking.

Out of scope: the JSON schema and validation rules for lesson seed files (`docs/CONTENT/LESSON_CONTENT_SCHEMA.md`), the specific MVP lesson list (`docs/CONTENT/BEGINNER_CURRICULUM.md`), authoring guidance/tone (`docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`), the scoring math a lesson attempt produces (`docs/MUSIC/SCORING_SYSTEM.md`), the Lesson Player's exact screen layout (owned by whichever UX document specifies screens in full detail).

## 3. Requirements

- A lesson has three states — locked, unlocked, completed (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7: `LessonProgress.status`) — and a learner sees which state every lesson in the path is in (`docs/PRODUCT_SPEC.md` §10: "Linear list/tree of levels with locked/unlocked/completed states").
- MVP uses a single linear prerequisite chain, no branching paths (`docs/PRODUCT_SPEC.md` §17's deliberate cut of `learningPaths` as a separate collection — `Lesson.level` + `Lesson.prerequisiteLessonId` already encode the one MVP path).
- A lesson unlocks only after its prerequisite's deterministic pass threshold is met (`docs/PRODUCT_SPEC.md` §10).
- The piano demonstrates before the learner attempts (`docs/PRODUCT_SPEC.md` §9's core loop: Learn → **Demonstrate** → Practice → Feedback → Repeat → Progress).
- The Lesson Player has a hint button (`docs/PRODUCT_SPEC.md` §10), and hints escalate on retry, never producing a hard stop (`docs/PRODUCT_SPEC.md` §9's core loop: "Repeat (retry with escalating hints, never a hard stop)").
- Attempts are tracked (`ExerciseAttempt.attemptNumber`, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4; `LessonProgress.attempts`, §4.7).

⚠ **Open question (BLOCKING):** `Lesson.passThreshold` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2) has **no ratified default value** — flagged there as `OQ-01` in `docs/OPEN_QUESTIONS.md`, and independently flagged as "blocking for lesson-content authoring" in `docs/PRODUCT/PRODUCT_REQUIREMENTS.md` (PR-LESSON-06). This document inherits that same blocking status: the unlocking algorithm in §4.1 below is fully specified as a *mechanism*, but cannot be exercised with real numbers until a `passThreshold` value (or per-lesson-type set of values) is ratified. `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` and `docs/CONTENT/BEGINNER_CURRICULUM.md` both use an explicit, clearly-labeled illustrative placeholder (`0.8`, matching the value already used consistently elsewhere in this documentation set — `docs/TESTING/UNIT_TESTING.md` §6.5, `docs/MUSIC/SCORING_SYSTEM.md` §9) rather than inventing a different number or leaving the field unfillable.

## 4. Architecture

### 4.1 Lesson states and the unlocking algorithm

A lesson's `LessonProgress.status` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7) is derived, not independently set — it is a function of the lesson's own attempt history and its prerequisite's status:

```
status(lesson) =
    if lesson.prerequisiteLessonId is null:
        // the first lesson in the chain — always at least unlocked
        (lesson has a passing PracticeSession) ? 'completed' : 'unlocked'
    else:
        prereqStatus = status(lesson.prerequisiteLessonId)
        if prereqStatus != 'completed':
            'locked'
        else if lesson has a passing PracticeSession (verdict.passed == true):
            'completed'
        else:
            'unlocked'
```

**"A passing `PracticeSession`"** means: at least one `PracticeSession` document for that `lessonId` under the learner's `practiceSessions` subcollection whose embedded `verdict.passed === true` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5, §4.6) — equivalently, `LessonProgress.bestScore / 100 >= Lesson.passThreshold` for that lesson, since `bestScore` (§4.7) already tracks the learner's best `compositeScore.value` across attempts.

**Where this is computed:** server-side, inside `getExerciseFeedback`, as the direct consequence of writing a new `PracticeSession` and updating `ProgressSummary` (`docs/ARCHITECTURE/DATA_FLOW.md` §4.2, step 7) — never client-computed as authoritative, consistent with `progressSummary` being a Cloud-Function-written-only field (`docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-08). The client reads the *result* (`ProgressSummary`, embedded on `UserProfile`) via `onSnapshot` to render locked/unlocked/completed states in the Learn tab (`docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §4.2) — it never derives lesson state itself from raw session history.

### 4.2 `level` vs. `prerequisiteLessonId` — two fields, two jobs

⚠ **Clarification (NON-BLOCKING, a reasonable resolved interpretation, not stated explicitly in the source documents):** `Lesson` carries both a `level: number` and a `prerequisiteLessonId: string | null` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2). This document treats them as serving **different purposes**, not redundant copies of the same ordering:

- **`prerequisiteLessonId` is the actual gating mechanism** — the unlocking algorithm in §4.1 depends on it exclusively.
- **`level` is a strict, per-lesson sequence number** (1 through however many MVP lessons exist — `docs/CONTENT/BEGINNER_CURRICULUM.md` assigns exactly one `level` value per lesson, not one per curriculum "stage"), used for display/grouping purposes and to populate `UserProfile.currentLevel` ("mirrors furthest-unlocked `Lesson.level`," §4.15) — a single comparable integer a dashboard can show ("Level 7 of 15") without walking the prerequisite chain.

This distinction matters because `docs/PRODUCT_SPEC.md` occasionally uses "Level" informally to mean a coarser, roughly-six-stage grouping (e.g. §7: "brief's Levels 1–6, roughly"), which is a *curriculum-narrative* concept, not the same thing as the `Lesson.level` *field*. `docs/CONTENT/BEGINNER_CURRICULUM.md` keeps both meanings distinct: it organizes its 14 MVP lessons into seven named sections for human readability (Orientation, Note Names, Octaves, Melodies, Rhythm, Scales, Triads — PRODUCT_SPEC.md §7's six narrative topics plus the separately-scoped Scales bullet) while assigning `Lesson.level` as one strictly increasing integer per lesson across the whole sequence, so `currentLevel`'s "furthest unlocked" semantics stay unambiguous even with multiple lessons inside one narrative section.

### 4.3 Demonstration

Per the core loop's **Demonstrate** stage (`docs/PRODUCT_SPEC.md` §9): before the learner attempts a lesson, the piano plays the target sequence itself — "the same instrument the learner will use" — both audibly and visually (keys highlighting in sync with playback).

**Mechanism:** demonstration playback reads `Lesson.expectedSequence: ExpectedNote[]` directly and schedules each note's Audio Layer trigger via Tone.js Transport (`docs/ARCHITECTURE/AUDIO_ARCHITECTURE.md`) at its `expectedTimeMs` offset from the demonstration's start, while the UI Layer highlights the corresponding key at the same offset. No Input Layer or Engine Core involvement during demonstration — this is playback, not scoring (`docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §3.3, step 1).

⚠ **PROPOSED, needs validation:** for a lesson whose `ExpectedNote`s omit `expectedTimeMs` entirely (a pure note-order lesson where timing isn't the point — `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2's comment on `expectedTimeMs`), demonstration notes are spaced at a fixed **600ms** interval each — not specified anywhere in `docs/PRODUCT_SPEC.md`, chosen as a comfortable, unhurried tempo for a complete beginner watching a demonstration for the first time. This constant is independent of, and should not be confused with, `Lesson.tempoBpm` (which is meaningful only for `type: 'melody' | 'rhythm'` lessons that actually have timed `ExpectedNote`s).

For a pure-orientation lesson with an **empty** `expectedSequence` (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §5.2 — e.g. "get oriented to the keyboard layout" with no specific target note at all), there may be nothing to demonstrate in this note-sequence sense at all; such a lesson's "demonstration" is whatever static/animated orientation content `instructions` and the lesson's own UI describe, not a Tone.js Transport playback — this is a content-authoring concern (`docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`), not an engine one.

### 4.4 Hints — escalation architecture, and a real gap in the canonical types

Per the core loop's **Repeat** stage (`docs/PRODUCT_SPEC.md` §9): on an incorrect attempt, the learner retries with escalating hints, never a hard stop. `docs/PRODUCT_SPEC.md` §10 additionally requires a dedicated hint button on the Lesson Player screen.

⚠ **PROPOSED, needs validation:** a **3-tier escalation** — attempt 1's incorrect result shows hint tier 1 (a gentle, general nudge — e.g., pointing at the general area of the keyboard); attempt 2's incorrect result shows tier 2 (more specific — e.g., naming the note that's needed); attempt 3 and beyond show tier 3 (near-explicit — e.g., visually highlighting the exact key). `docs/PRODUCT_SPEC.md` does not specify an exact tier count or content; 3 tiers is a common, reasonable pattern proposed here, not extracted fact. The actual hint *text* per lesson is content-authoring work (`docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`), not something this architecture document invents.

**Which `attemptNumber` drives hint escalation:** the client-local retry counter for the *current* lesson-player session (tracked in Zustand, incrementing on each incorrect result within one sitting), not the server-trusted, cross-session `LessonProgress.attempts` total. This is a deliberate, lower-stakes use of attempt-counting than the server's: hint-tier selection is a pure UX affordance with no scoring or security implication, so it is safe (and simpler) to drive it from client-local state, unlike `ExerciseAttempt.attemptNumber`, which the server explicitly treats as informational-only and never trusts for anything it computes (`docs/ARCHITECTURE/DATA_FLOW.md` §4.4). A learner who leaves mid-lesson and returns later reasonably starts back at hint tier 1 for a fresh attempt sequence under this design — a deliberate simplicity choice, not a gap, since the alternative (persisting hint-tier state across sessions) adds real complexity for a purely cosmetic escalation.

⚠ **Open question (BLOCKING):** `Lesson` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2) has **no field to store hint content at all** — no `hints: string[]` or equivalent. This is a genuine, unaddressed gap: `docs/PRODUCT_SPEC.md` §9 and §10 both explicitly require escalating hints and a hint button as core-loop, MVP-scope features, and multiple parallel-batch documents already assume hint escalation works (`docs/PRODUCT/USER_STORIES.md`: "Repeated incorrect attempts escalate the hint level"; `docs/PRODUCT/USER_FLOWS.md`; `docs/PRODUCT/PRODUCT_REQUIREMENTS.md` PR-LESSON-05; `docs/UX/UX_PRINCIPLES.md`) — but none of them, nor `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` itself, defines where hint text actually lives. Flagged **BLOCKING** because the hint button and escalating-hints requirement literally cannot be implemented without some field to read hint content from, and `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` (this batch) needs somewhere to put it *now* to produce a usable seed-file schema.

**This document's proposed resolution**, stated so it is a confirmable proposal rather than a silent local invention: add `hints: string[]` to `Lesson`, an ordered array where index `0` is tier 1's hint text, index `1` is tier 2's, and so on, with the **last** array element repeating for any attempt number beyond the array's length (so a lesson author isn't forced to write a unique hint for an arbitrarily high retry count). `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` adopts this field in its proposed JSON schema, flagged consistently there as pending confirmation against the canonical `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`, which this document does not have authority to edit directly.

### 4.5 Attempts tracking

`ExerciseAttempt.attemptNumber` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4) is client-supplied and informational-only — the server derives its own authoritative attempt count from existing `PracticeSession` records if attempt-number-dependent scoring logic needs it (`docs/ARCHITECTURE/DATA_FLOW.md` §4.4), which is exactly the mechanism `docs/MUSIC/SCORING_SYSTEM.md` §4.4's cross-attempt consistency calculation depends on. `LessonProgress.attempts` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7) is the durable, server-written running total per lesson, incremented once per completed `getExerciseFeedback` call for that `lessonId` — this is the number surfaced on the Progress/Stats screen (`docs/PRODUCT_SPEC.md` §10), distinct from the client-local hint-escalation counter in §4.4, which is never persisted at all.

## 5. Data structures

Uses `Lesson`, `ExpectedNote`, `LessonProgress`, `ProgressSummary`, `ExerciseAttempt`, `ExerciseVerdict` — all `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`, none redefined here. §4.4 proposes one field addition (`Lesson.hints: string[]`), explicitly flagged as a proposal pending confirmation, not asserted as already-canonical.

## 6. Interfaces / contracts

The Lesson Player consumes: a `Lesson` (via the public-read `lessons` Firestore collection, `docs/DATA/FIRESTORE_SCHEMA.md` §4.4), the learner's current `LessonProgress` for that lesson (via `UserProfile.progressSummary`), and produces an `ExerciseAttempt` sent to `getExerciseFeedback` on exercise completion (`docs/ARCHITECTURE/DATA_FLOW.md` §4.2) — this document adds no new callable-function surface beyond what `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` §6 already specifies.

## 7. Dependencies

`docs/MUSIC/MUSIC_ENGINE_SPEC.md` (the engine that produces the `ExerciseVerdict` this document's completion model reads), `docs/ARCHITECTURE/AUDIO_ARCHITECTURE.md` (demonstration playback), `docs/ARCHITECTURE/DATA_FLOW.md` (the server-side write path).

## 8. Error handling

- A lesson whose `prerequisiteLessonId` points to a non-existent or malformed lesson document should fail the acyclic-chain validation at seed time (`docs/CONTENT/LESSON_CONTENT_SCHEMA.md` §Validation rules), never surface as a runtime "can't determine lock state" error to a learner.
- If `ProgressSummary` is briefly stale (a realtime-listener propagation delay after `getExerciseFeedback` writes) the Learn tab should show the previous known state, not an error — this is the same degraded-but-never-broken principle applied to a normal sync delay, not an outage (`docs/PRODUCT_SPEC.md` §10, §22).

## 9. Testing requirements

- The unlocking algorithm (§4.1), parameterized: a lesson with no passing session for its prerequisite stays `locked`; one with a passing prerequisite session and no attempts of its own is `unlocked`; one with its own passing session is `completed`.
- The first lesson in the chain (`prerequisiteLessonId: null`) is always at least `unlocked`, never `locked`, regardless of any other state.
- Hint-tier selection (§4.4) against the client-local attempt counter, including the "beyond the array's length, repeat the last hint" behavior.
- The empty-`expectedSequence` orientation-lesson path (§4.3, cross-referencing `docs/MUSIC/MUSIC_ENGINE_SPEC.md` §5.2) reaches `completed` status correctly without going through the standard scored-attempt path.

## 10. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29:
- [ ] A learner can complete the full MVP lesson sequence with no dead end or broken step.
- [ ] Progress persists across sign-out/sign-in and across devices for the same account.
- [ ] A learner can retry an incorrect attempt with escalating hints, and is never hard-stopped.

## 11. Out of scope

- Lesson JSON schema and validation → `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`
- The actual MVP lesson list → `docs/CONTENT/BEGINNER_CURRICULUM.md`
- Tone/voice for instructions and hint text → `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`
- Scoring math → `docs/MUSIC/SCORING_SYSTEM.md`
- Exact Lesson Player screen layout beyond what `docs/PRODUCT_SPEC.md` §10 already states → not yet specified in this documentation batch

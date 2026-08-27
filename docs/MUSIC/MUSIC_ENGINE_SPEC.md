# Music Engine Spec

**Status:** Draft v1.0 — music engine documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §11, §13, §26, §27, §29; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1–§4.5, §4.12–§4.14, §5
**Companion documents:** `docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` (the architecture-level companion — owns module *boundaries and dependency rules*; this document owns the *algorithms and the master contract*, and is the detailed spec that architecture document explicitly defers to), `docs/MUSIC/NOTE_SYSTEM.md`, `docs/MUSIC/CHORD_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/SCORING_SYSTEM.md` (the four per-module specs this document indexes), `docs/MUSIC/INPUT_SYSTEM.md`, `docs/MUSIC/KEYBOARD_MAPPING.md` (adjacent Input Layer concerns, architecturally outside this engine — see §2), `docs/ENGINEERING/PROJECT_STRUCTURE.md` §3–§4, `docs/ENGINEERING/CODING_STANDARDS.md` §3.3, `docs/TESTING/UNIT_TESTING.md`

This is the master specification for `packages/music-engine`. Every other document in `docs/MUSIC/` is a detailed module-level spec this document indexes; read this one first.

---

## 1. Purpose

State, once and unambiguously, what `packages/music-engine` is, what it is responsible for, what it must never depend on, and its single public contract. `docs/PRODUCT_SPEC.md` §11 introduces the Engine Core as "plain TypeScript, zero React/Firebase/DOM dependency... This is what actually gets unit-tested in isolation" — this document turns that sentence into a complete, implementation-ready specification.

## 2. Scope

**In scope:** the engine's zero-framework-dependency rule, its full list of responsibilities, its internal module breakdown, and its single external public contract.

**Out of scope, owned by sibling documents in this directory:**
- MIDI note representation, pitch classes, note names, octaves, frequency calculation → `docs/MUSIC/NOTE_SYSTEM.md`
- Chord detection, the `ChordDefinition` interval table, inversion identification → `docs/MUSIC/CHORD_SYSTEM.md`
- Scale detection, the `ScaleDefinition` interval table, ordered/unordered matching → `docs/MUSIC/SCALE_SYSTEM.md`
- Timing, accuracy, completeness, consistency, the composite scoring formula → `docs/MUSIC/SCORING_SYSTEM.md`

**Adjacent, but explicitly outside `packages/music-engine` itself:**
- How a `NoteEvent` is produced by an input adapter (mouse/touch, computer-keyboard, MIDI) → `docs/MUSIC/INPUT_SYSTEM.md`
- The default computer-keyboard mapping table → `docs/MUSIC/KEYBOARD_MAPPING.md`

These last two live in `docs/MUSIC/` because they share this cluster's MIDI-note vocabulary and an implementer needs to understand the shape of data arriving at the engine's boundary — but the code they describe lives in `apps/web`'s Input Layer, upstream of and architecturally separate from `packages/music-engine` (`docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` §4.1: `NoteEvent` "fans out to Audio and Engine Core," it does not originate inside the Engine Core). Do not read this document's zero-dependency rule (§3) as applying to `INPUT_SYSTEM.md`/`KEYBOARD_MAPPING.md`'s code — it applies only to `packages/music-engine` itself.

**Fully out of scope, owned elsewhere in `docs/`:** Tone.js/audio triggering (`docs/ARCHITECTURE/AUDIO_ARCHITECTURE.md`), Gemini prompt construction (`docs/AI/`), how the Cloud Function invokes this engine server-side (`docs/ARCHITECTURE/DATA_FLOW.md` §4), Firestore persistence of the engine's output (`docs/DATA/FIRESTORE_SCHEMA.md`).

## 3. Requirements

### 3.1 The zero-framework-dependency rule — non-negotiable

**`packages/music-engine` is framework-independent. It must not import React, Next.js, Firebase, Gemini, or any browser UI code.**

Stated fully, reproduced from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 2 and `docs/ENGINEERING/CODING_STANDARDS.md` §3.3, because this is the single most important sentence in this document: no file under `packages/music-engine/src` may import `react`, `next`, `firebase`, `firebase-admin`, any package under the `firebase/*` subpath family, or a Gemini/AI SDK package. No browser globals (`window`, `document`, `navigator`, `AudioContext`) — the package must run identically under Node.js (inside a Cloud Function) and inside a browser bundle. No network calls of any kind, to Firestore, to Gemini, or to anything else — every piece of data the engine needs is passed in as an argument by its caller. Its only permitted internal workspace dependency is `packages/shared-types`.

**It is a pure function of `(NoteEvent[], Lesson | Exercise) → ExerciseVerdict`** (§6 below is the full contract). Pure means: no side effects, no hidden state carried between calls, identical input always produces identical output. This is what makes the engine's test suite need zero mocking (`docs/TESTING/UNIT_TESTING.md` §3) and what makes the same code usable both for instant client-side provisional feedback and as the one server-side authoritative computation (`docs/ARCHITECTURE/DATA_FLOW.md` §3).

**Gemini is never involved in anything this engine computes.** Not as a fallback, not as a sanity check, not for an edge case the deterministic logic finds ambiguous. Every field of `ExerciseVerdict` — `correctNotes`, `missedNotes`, `extraNotes`, `accuracy`, `timingScore`, `completeness`, `consistency`, `perNoteTiming`, `compositeScore`, `passed` — is produced exclusively by the algorithms in this document and its four siblings, executed as plain TypeScript. Gemini receives a completed `ExerciseVerdict` as **read-only input**, after this engine has already finished, and explains it in prose — it does not participate in producing it (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5; `docs/ARCHITECTURE/AI_ARCHITECTURE.md` §3–§4.2; `docs/MUSIC/SCORING_SYSTEM.md` §4.8). An LLM asked "was this chord correct" will usually get it right — but "occasionally wrong about music theory" is disqualifying for something presenting itself as a piano teacher (`docs/PRODUCT_SPEC.md` §14), which is why this boundary is architectural, not a style preference.

**Enforcement, not just documentation:** this rule is checked mechanically — an import-restriction lint rule or CI grep check (`docs/ENGINEERING/CODING_STANDARDS.md` §3.3), and the fact that `packages/music-engine` is its own workspace package with its own `package.json` dependency list is the physical precondition that makes automated enforcement possible at all (`docs/ENGINEERING/PROJECT_STRUCTURE.md` §4).

### 3.2 Responsibilities

`packages/music-engine` is responsible for:

| Responsibility | Owning sibling document |
|---|---|
| MIDI note representation (canonical `0`–`127` integer pitch) | `docs/MUSIC/NOTE_SYSTEM.md` |
| Pitch classes (`midiNote mod 12`) | `docs/MUSIC/NOTE_SYSTEM.md` |
| Note names (scientific pitch notation, derived display strings) | `docs/MUSIC/NOTE_SYSTEM.md` |
| Octaves (derived from `midiNote`) | `docs/MUSIC/NOTE_SYSTEM.md` |
| Frequency calculation (`f = 440 × 2^((midi−69)/12)`) | `docs/MUSIC/NOTE_SYSTEM.md` |
| Intervals (semitone offset tables for chords and scales) | `docs/MUSIC/CHORD_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md` |
| Chord detection (pitch-class-set matching, inversion identification) | `docs/MUSIC/CHORD_SYSTEM.md` |
| Scale detection (ordered and unordered sequence matching) | `docs/MUSIC/SCALE_SYSTEM.md` |
| Note sequences (melody/scale ordered matching — the same algorithm serves both lesson types) | `docs/MUSIC/SCALE_SYSTEM.md` §4.2 |
| Timing (per-note deviation bucketing) | `docs/MUSIC/SCORING_SYSTEM.md` |
| Accuracy (`correctNotes.length / expectedSequence.length`) | `docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/SCORING_SYSTEM.md` |
| Completeness (attempted-regardless-of-timing coverage) | `docs/MUSIC/SCORING_SYSTEM.md` |
| Consistency (within- and cross-attempt timing variance) | `docs/MUSIC/SCORING_SYSTEM.md` |
| Scoring (the composite score formula and `ScoringWeights` application) | `docs/MUSIC/SCORING_SYSTEM.md` |
| Exercise validation (the orchestration that turns all of the above into one `ExerciseVerdict`) | This document, §5 |

### 3.3 What this engine is explicitly not responsible for

- Acoustic/microphone pitch detection — a different problem entirely, Phase 3, not designed for even at the interface level (`docs/MUSIC/SCORING_SYSTEM.md` §4.7).
- Producing sound — the Audio Layer is triggered directly by the Input Layer, in parallel with this engine, not downstream of it (`docs/ARCHITECTURE/AUDIO_ARCHITECTURE.md` §4.4).
- Rendering anything — the UI Layer owns key visuals, computed from this engine's output; the engine owns no rendering logic of any kind.
- Deciding lesson sequencing, unlock logic, XP, or streaks — those are `docs/CONTENT/LESSON_ARCHITECTURE.md` / Cloud-Function concerns that *consume* this engine's `passed` output, not something this engine computes itself.

## 4. Architecture

### 4.1 Internal module breakdown

The engine has five internal modules. Four map directly onto the sibling documents in this directory; the fifth (`validation`) is the orchestrator that composes them into the single public contract (§6).

| Conceptual module | Owning document | Physical location (`docs/ENGINEERING/PROJECT_STRUCTURE.md` §3) |
|---|---|---|
| **noteSystem** | `docs/MUSIC/NOTE_SYSTEM.md` | `packages/music-engine/src/notes/` |
| **chordSystem** | `docs/MUSIC/CHORD_SYSTEM.md` | `packages/music-engine/src/chords/` |
| **scaleSystem** | `docs/MUSIC/SCALE_SYSTEM.md` | `packages/music-engine/src/scales/` |
| **scoring** | `docs/MUSIC/SCORING_SYSTEM.md` | `packages/music-engine/src/scoring/` |
| **validation** | This document, §5 | Proposed: `packages/music-engine/src/validation/` (or a top-level `src/evaluate.ts`) |

⚠ **Open question (NON-BLOCKING — a naming/structure alignment, not a scope gap):** `docs/ENGINEERING/PROJECT_STRUCTURE.md` §3's folder tree lists `notes/`, `chords/`, `scales/`, `scoring/` under `packages/music-engine/src/` but does not list a fifth `validation/` folder. This document's `validation` module — the orchestrating `evaluate()` entry point (§6) plus the empty-`expectedSequence` special case (§5.2) — is not omitted from that tree by oversight so much as not yet named in it; it most naturally lives directly under `src/` as the composition root the other four modules feed into. Flagged so `docs/ENGINEERING/PROJECT_STRUCTURE.md`'s next revision can fold in an explicit `validation/` entry (or confirm `src/evaluate.ts` at the top level) rather than an implementer guessing where the orchestrator file belongs.

### 4.2 How the modules compose

```
evaluate(noteEvents, definition)
  │
  ├─ 1. Validate input shape (reject/log malformed NoteEvents — §7)
  ├─ 2. If definition.expectedSequence is empty → return the auto-pass verdict (§5.2), skip 3-6 entirely
  ├─ 3. noteSystem: derive note names for every relevant midiNote (feeds every later step's display output)
  ├─ 4. Depending on definition.type:
  │      'chord'                  → chordSystem.detectChord(...) (docs/MUSIC/CHORD_SYSTEM.md §4)
  │      'scale' | 'melody'       → scaleSystem.matchOrdered(...) (docs/MUSIC/SCALE_SYSTEM.md §4.2)
  │      'noteId' (recall-style)  → scaleSystem.matchUnordered(...) (docs/MUSIC/SCALE_SYSTEM.md §4.3)
  │      'orientation' | 'octave' → matchOrdered or matchUnordered depending on the specific lesson's content
  ├─ 5. scoring: timing buckets, completeness, consistency, correctness-for-scoring (docs/MUSIC/SCORING_SYSTEM.md §4.1-§4.4)
  ├─ 6. scoring: composite score + passed determination (docs/MUSIC/SCORING_SYSTEM.md §4.5)
  └─ return fully-populated ExerciseVerdict
```

This composition is `validation`'s entire job — it contains no music-theory logic of its own beyond dispatching to the right sibling module and assembling their outputs into one object.

## 5. Data structures

This document does not redefine any type — every shape referenced below is canonical in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` and reproduced only for orientation.

### 5.1 Primary types

`NoteEvent` (§4.1), `Lesson` (§4.2), `Exercise` (§4.3), `ExerciseVerdict` (§4.5), `ChordDefinition` (§4.12), `ScaleDefinition` (§4.13), `Score`/`ScoringWeights` (§4.14) — all `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`.

### 5.2 The empty-`expectedSequence` case — a real gap, resolved here

`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 explicitly permits an empty `expectedSequence` on `Lesson`: `"required, may be empty for pure-orientation lessons (e.g. 'find Middle C')"`. But every formula in `docs/MUSIC/SCALE_SYSTEM.md` and `docs/MUSIC/SCORING_SYSTEM.md` divides by `expectedSequence.length` at some point (`accuracy = correctNotes.length / expectedSequence.length`, `completeness`'s denominator, `noteCorrectness`'s denominator) — an empty `expectedSequence` makes every one of those a division by zero, producing `NaN` rather than a usable `ExerciseVerdict`.

⚠ **Open question (BLOCKING):** neither `docs/PRODUCT_SPEC.md` nor `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` addresses this case anywhere. It is not a hypothetical edge case — `docs/CONTENT/BEGINNER_CURRICULUM.md`'s very first lesson (`l01-keyboard-orientation`) is exactly this kind of orientation lesson, meaning a naive implementation would produce `NaN` scores on literally the first lesson every learner takes. Flagged **BLOCKING** because it is a correctness defect that surfaces immediately at the M1/M2 milestones (`docs/PRODUCT_SPEC.md` §28), not a nice-to-have — this needs resolution before `validation` is implemented, not discovered during it.

**Resolution, stated as the engine's defined behavior** (per §4.2's composition order, step 2): when `definition.expectedSequence.length === 0`, `evaluate()` skips the entire correctness/timing/completeness/consistency pipeline and returns a fixed **auto-pass verdict** as soon as the exercise-end signal is received (there is nothing to score against):

```typescript
{
  correctNotes: [], missedNotes: [], extraNotes: [],
  accuracy: 1, timingScore: 1, completeness: 1, consistency: 1,
  perNoteTiming: [],
  compositeScore: { value: 100, breakdown: { correctness: 1, timing: 1, consistency: 1 } },
  passed: true,
}
```

This treats a pure-orientation lesson's "completion" as a binary acknowledgment (the learner reached the end of the demonstration/interaction), not a scored attempt — consistent with `docs/CONTENT/LESSON_ARCHITECTURE.md` §Architecture's treatment of orientation-lesson completion, which this resolution is designed to match exactly. `passThreshold` is irrelevant for such a lesson (any threshold `≤ 1.0` is satisfied), sidestepping the `passThreshold`-default open question (`OQ-01`) for this lesson type entirely.

## 6. Interfaces / contracts

**The engine's single external public contract:**

```typescript
function evaluate(noteEvents: NoteEvent[], definition: Lesson | Exercise): ExerciseVerdict
```

- **Pure, synchronous, no side effects** — matches `docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` §6 exactly.
- Called from two places, with the exact same code: client-side in `apps/web` (instant, provisional feedback while a key is being judged) and server-side inside the `getExerciseFeedback` Cloud Function (the one authoritative evaluation, `docs/ARCHITECTURE/DATA_FLOW.md` §3–§4).
- Every other function named across `docs/MUSIC/NOTE_SYSTEM.md`, `docs/MUSIC/CHORD_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/SCORING_SYSTEM.md` (`midiToNoteName`, `detectChord`, `matchOrdered`, `scoreTiming`, etc.) is an internal helper `evaluate()` composes — none of them individually is this contract.

**A stated, flagged exception — cross-attempt consistency:** `docs/MUSIC/SCORING_SYSTEM.md` §4.4 documents a real tension between this exactly-two-argument contract and the scoring rubric's cross-attempt consistency requirement (which needs the learner's prior-attempt history, unavailable to a function of only `(NoteEvent[], Lesson | Exercise)`). That document's proposed resolution — a small, separate, still-pure exported function (`scoreConsistencyAcrossAttempts`) that the calling Cloud Function invokes separately using data it already has — keeps this contract's headline shape exactly as stated here, and is flagged there as needing explicit sign-off. This document's contract statement should not be read as silently contradicted by that resolution; it is the primary, tested, acceptance-criteria-bound entry point, with one small, clearly-scoped, clearly-flagged companion function alongside it.

## 7. Dependencies

`packages/shared-types` only (§3.1). No other internal or external package dependency is permitted. Consumers — never dependencies — are `apps/web` and `functions/` (`docs/ENGINEERING/PROJECT_STRUCTURE.md` §6).

## 8. Error handling

- Malformed `NoteEvent`s (missing a required field, an out-of-range `midiNote`/`velocity`) are rejected — logged, not thrown — at the point they'd otherwise enter this engine's computation; `evaluate()` never crashes on a stray or attacker-controlled input, consistent with `docs/PRODUCT_SPEC.md` §2's "the learner is never blocked" principle applying to the engine's own robustness, not just its network independence.
- An empty `NoteEvent[]` (learner attempted nothing) for a lesson with a non-empty `expectedSequence` returns a well-formed verdict: `accuracy: 0`, `missedNotes` equal to the full expected sequence, `passed: false` — never a thrown error or `undefined` (`docs/TESTING/UNIT_TESTING.md` §8).
- An empty `expectedSequence` (§5.2) is not an error at all — it is the defined, auto-pass orientation-lesson case.
- The engine never throws for "the learner played the wrong note" — that is a normal, expected `ExerciseVerdict` outcome, not an error (`docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` §9).

## 9. Testing requirements

- **Near-100% coverage target, ≥90% CI-enforced floor**, on chord/scale/scoring logic specifically (`docs/PRODUCT_SPEC.md` §29; `docs/TESTING/UNIT_TESTING.md` §7) — measured on `packages/music-engine/src/` alone, never blended into a repo-wide average.
- **The canonical worked example** (`docs/MUSIC/SCALE_SYSTEM.md` §4.2) as the first, required, non-paraphrased test.
- **All 12 major and 12 minor triads**, inversion- and order-independent (`docs/MUSIC/CHORD_SYSTEM.md` §9).
- **The empty-`expectedSequence` auto-pass case** (§5.2) as an explicit fixture — this is now a named, required test given the BLOCKING severity of the gap it resolves.
- **A build/lint step failing on any disallowed import** (§3.1) — `docs/ENGINEERING/CODING_STANDARDS.md` §3.3's import-boundary check, run alongside this suite, not as a separate concern.
- Full per-module testing detail: `docs/MUSIC/NOTE_SYSTEM.md` §9, `docs/MUSIC/CHORD_SYSTEM.md` §9, `docs/MUSIC/SCALE_SYSTEM.md` §9, `docs/MUSIC/SCORING_SYSTEM.md` §9, and the canonical suite-level treatment in `docs/TESTING/UNIT_TESTING.md`.

## 10. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29 (the complete Music Engine acceptance block):
- [ ] All 12 major and 12 minor triads are correctly identified regardless of inversion or the order notes are played.
- [ ] The worked example (expected C-D-E-F-G, played C-D-E-G) returns exactly `{missedNotes: ['F'], accuracy: 0.8}`.
- [ ] Unit test coverage on chord/scale/scoring logic is ≥90%.
- [ ] `packages/music-engine` has zero import of `firebase/*`, `react`, `next`, or any Gemini/AI SDK — verified by a lint/build rule, not just code review.
- [ ] A `Lesson` with an empty `expectedSequence` produces a well-formed, non-`NaN` verdict (§5.2).

## 11. Out of scope

- Everything listed in §2's "owned by sibling documents" and "adjacent, but explicitly outside" tables.
- Cloud Function orchestration around this engine (loading the `Lesson`/`Exercise` definition, writing `PracticeSession`, calling Gemini) → `docs/ARCHITECTURE/DATA_FLOW.md` §4.
- Firestore/security-rule enforcement of the client/server trust boundary → `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`, `docs/ARCHITECTURE/SECURITY_ARCHITECTURE.md`.

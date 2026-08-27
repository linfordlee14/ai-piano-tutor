# M1 — Piano That Makes Sound

## 1. Objective

The instrument works completely standalone, before any lesson wraps around it. By the end of M1, a learner can open Free Play, play any note or triad by mouse, touch, or computer keyboard, hear a real piano sound in well under 50ms, and see it correctly recognized — all with zero lesson content, zero Firestore writes beyond what auth already needs, and zero Gemini involvement.

**A note on this file's name:** per this documentation batch's scoping instruction, this file is named `M1_MUSIC_ENGINE.md` even though its actual scope — matching `docs/PRODUCT_SPEC.md` §28's own milestone title, "Piano that makes sound" — is broader than the engine alone: it covers the full input → audio → engine → UI stack for the standalone instrument. `packages/music-engine` (the deterministic core) is the single most important and highest-risk deliverable in this milestone, which is why it names the file, but M1 is not scoped narrower than what §28 actually describes.

**Source of truth:** `docs/PRODUCT_SPEC.md` §6 (Core Features — Virtual Piano, Explore/Free Play), §11 (Piano/Music Engine Architecture), §12 (Audio Architecture), §13 (Music Theory Engine), §21 (State Management), §22 (Performance), §28 (M1 row), §29 (Acceptance Criteria — Piano & Input, Music Engine sections). Contracts: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1 (`NoteEvent`), §4.12–4.14 (`ChordDefinition`, `ScaleDefinition`, `Score`/`ScoringWeights`).

## 2. Prerequisites

**M0** must be done: the repository skeleton, CI/CD, Firebase provisioning, and default-deny security rules baseline are in place. M1 adds real implementation into the `packages/music-engine` and `apps/web` skeletons M0 created — it does not re-do M0's infrastructure work.

## 3. Exact Scope

**Music Engine Core (`packages/music-engine`)** — the deterministic heart of the product:
- Note identification: MIDI-number-based, with note-name/octave derivation for display (`docs/PRODUCT_SPEC.md` §13; derivation rules owned by `docs/MUSIC/NOTE_SYSTEM.md`, parallel batch).
- Chord detection: pitch-class-set matching against known interval patterns, inversion-tolerant and order-independent by construction, covering **all 12 major and all 12 minor triads** (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.12 — `diminished`/`augmented` are defined as types but unused until Phase 2).
- Scale validation: both ordered comparison ("play it ascending") and unordered comparison ("did you hit all seven notes"), per `docs/PRODUCT_SPEC.md` §13.
- The **full scoring rubric** — note correctness, timing buckets, completeness, consistency, and the composite `Score`/`ScoringWeights` formula (`docs/PRODUCT_SPEC.md` §13, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14) — is implemented and unit-tested in M1 even though **no lesson exists yet to visibly exercise it in the UI**. This is deliberate: `docs/TESTING/TESTING_STRATEGY.md` places the music-engine unit suite at M1, and M2's `getExerciseFeedback` Function needs a complete, already-tested engine to call into on day one rather than building engine logic and Function logic simultaneously. M1's own UI (Free Play) exercises the *detection* half of the engine visibly; the *scoring* half becomes UI-visible starting at M2.
- `Exercise`/`Lesson`-shaped inputs are accepted generically (the engine's public API takes `NoteEvent[]` plus an expected-sequence/scoring-weights shape, per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 2's `(NoteEvent[], Lesson | Exercise) → ExerciseVerdict` signature) — M1 does not need real `Lesson` or `Exercise` content to exist to test this; `UNIT_TESTING.md`'s fixtures are plain data, not real seeded content.

**Input Layer (`apps/web`):**
- Mouse and touch adapters, and a computer-keyboard adapter using the default mapping table below, all normalizing into the canonical `NoteEvent` shape (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1).
- The keyboard mapping is stored as data, not hardcoded per-component (`docs/PRODUCT_SPEC.md` §11), so a later settings screen can remap it — the remapping UI itself is not required in M1, but the data-driven structure enabling it is.

| Piano key | C | C# | D | D# | E | F | F# | G | G# | A | A# | B | C (next 8ve) |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Keyboard | A | W | S | E | D | F | T | G | Y | H | U | J | K |

**Audio Layer:**
- Tone.js, with `Tone.Sampler` loaded against a trimmed **Salamander Grand Piano** sample set (CC BY 3.0 — Alexander Holm), served as static assets from the same CDN as the app, **not** Firebase Storage (`docs/PRODUCT_SPEC.md` §1 pushback, §12, §16).
- Load only the 1–2 octaves the current context needs first (in M1, Free Play's visible range); prefetch the rest during idle time.
- `AudioContext` created/resumed on first user gesture, with a visible "tap to enable sound" affordance — never a silently broken piano (§12).
- The Audio Layer is triggered directly by the Input Layer's `NoteEvent`s, **not** routed through the Engine Core, so sound latency never waits on scoring logic (§11) — this is an architectural requirement, not an optimization to consider later.
- Attribution: a visible credit for Salamander Grand Piano / Alexander Holm and a `NOTICE` file in the repo — legally required by the CC BY 3.0 license (§12). M1 is the natural place to add this since it's the milestone that first ships the samples, though `docs/ROADMAP/M5_HARDENING.md` re-verifies it's actually visible in the shipped UI before launch.

**UI Layer:**
- Virtual piano: white/black keys, toggleable note labels, press/hover animation states (idle/hover/active/correct/incorrect — though correct/incorrect only becomes meaningful once a scored context exists in M2; M1's UI uses idle/hover/active).
- Adjustable visible octave range with pan, sustain toggle, volume control (`docs/PRODUCT_SPEC.md` §6).
- Memoized per-key/per-octave rendering so pressing one key does not re-render the whole keyboard (§22 — verified by `docs/TESTING/PERFORMANCE_TESTING.md` §7).
- **Piano Free Play screen** (`docs/PRODUCT_SPEC.md` §10): full piano, live note/chord recognition surfaced as a toast ("You played: C major"), zero scaffolding — this is also the screen used for the pre-signup "try before signup" flow, so it must work for an anonymous auth session (already wired at M0).

**State Management:**
- A Zustand store for client-only UI/engine state (current visible octave, live note buffer, audio-loading state) — chosen specifically because it's readable/writable from Tone.js event callbacks outside React's render cycle (`docs/PRODUCT_SPEC.md` §21).
- The Engine Core itself does **not** live in React state — it's a plain module called imperatively; only its output (verdicts, key-highlight state) flows into Zustand for rendering (§21).

## 4. Files/Components Expected

```
packages/music-engine/
  src/
    notes.ts            # MIDI <-> note-name derivation, note identification
    chords.ts            # pitch-class-set chord detection, ChordDefinition table
    scales.ts             # ordered/unordered scale validation, ScaleDefinition table
    scoring.ts             # composite scoring rubric (correctness/timing/completeness/consistency)
    index.ts                # public API: evaluate(NoteEvent[], Lesson | Exercise) -> ExerciseVerdict
  src/__tests__/            # the highest-value suite in the repo — see docs/TESTING/UNIT_TESTING.md

apps/web/
  app/(app)/piano/           # Piano Free Play route
  components/piano/          # PianoKeyboard, PianoKey, OctaveControls, SustainToggle, VolumeControl
  components/piano/RecognitionToast.tsx
  hooks/useAudioEngine.ts     # Tone.js wiring, sample loading/preloading, AudioContext gesture handling
  hooks/useInputAdapters.ts    # mouse/touch/keyboard adapters -> NoteEvent
  stores/engineStore.ts         # Zustand store, per docs/PRODUCT_SPEC.md §21
  lib/keyboardMapping.ts         # default mapping table (data, not hardcoded per-component)

public/audio/salamander/         # (or content/audio/ — exact static-asset path owned by
                                  # docs/ENGINEERING/PROJECT_STRUCTURE.md) trimmed sample set
NOTICE                            # Salamander Grand Piano / Alexander Holm CC BY 3.0 attribution
```

## 5. Technical Requirements

- `packages/music-engine` has **zero** `firebase/*`, `react`, `next`, or AI-SDK imports — enforced as a lint/dependency-boundary check, not just a convention (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 2). This is the single most important technical constraint in this milestone.
- All pitch representation inside the engine uses MIDI note numbers (integers 0–127); note names are derived for display only, never compared as the source of truth (`docs/PRODUCT_SPEC.md` §13, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §3).
- `midiNote` and `velocity` are validated as integers in `[0, 127]` at the Input Layer; out-of-range values are rejected (logged, not thrown) rather than crashing the engine (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1).
- Non-velocity-sensitive input sources (mouse, touch, computer keyboard) default `velocity` to `100` (§4.1).
- MIDI input itself is explicitly **not** built in M1 (see §8) — but because the `source` field on `NoteEvent` is diagnostic-only and never branched on for scoring (§4.1), nothing in this milestone's engine or scoring code should special-case any input source. This is what makes a future `MIDIAdapter` a pure addition later, not a refactor.
- Key-press-to-sound latency target: under ~50ms on a mid-range Android device and a current iPhone (`docs/PRODUCT_SPEC.md` §12, §29) — cross-reference `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md` (parallel batch) for the authoritative figure and measurement method.

## 6. Data Structures / Contracts Touched

- `NoteEvent` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1) — created here, by input adapters, for the first time in the codebase.
- `ChordDefinition`, `ScaleDefinition` (§4.12–4.13) — static engine config, authored in this milestone.
- `Score` / `ScoringWeights` (§4.14) — implemented here; note that `ScoringWeights` are metadata supplied by whatever calls the engine (a `Lesson` or `Exercise`), never hardcoded inside `packages/music-engine` itself.
- `ExerciseVerdict` (§4.5) — the engine's return type, fully implemented and tested here, first *consumed* by a real Cloud Function in M2.

## 7. Tests Required

- **`docs/TESTING/UNIT_TESTING.md` — the primary gate for this milestone.** Every requirement in that document applies in full at M1: the canonical worked example (expected C-D-E-F-G, played C-D-E-G → `{missedNotes: ['F'], accuracy: 0.8}`), all 12 major and 12 minor triads across all roots and at least one inversion each, ordered/unordered scale checks, every scoring-rubric component tested in isolation, and ≥90% coverage on chord/scale/scoring logic specifically.
- **`docs/TESTING/PERFORMANCE_TESTING.md` §5** — the manual Android/iPhone latency spot-check is first meaningful at M1, since this is the first milestone with real audio playback. It does not need to be repeated on every subsequent milestone unless the Audio Layer changes, but it must happen at least once here, not deferred to M5.
- **`docs/TESTING/ACCESSIBILITY_TESTING.md`** baseline: full keyboard-only operability of the piano itself (§3 of that document's second commitment) is testable starting now, since the piano exists. The broader manual screen-reader pass across the whole app is still M5 scope.
- **Note recognition/chord recognition correctness in the actual Free Play UI** (not just the engine in isolation) belongs to `docs/TESTING/E2E_TESTING.md`'s computer-keyboard-input scenario (§5.2 of that document) and is exercised further once M2 adds the reference E2E scenario — a light Playwright smoke test of Free Play's recognition toast is reasonable to add in M1 itself, ahead of the full E2E suite.

## 8. Acceptance Criteria

Reproduced and expanded from `docs/PRODUCT_SPEC.md` §28 (M1 row) and §29:

- [ ] A learner can play any of the 12 major triads and see it correctly recognized (§28's literal M1 acceptance line).
- [ ] All 12 major **and** all 12 minor triads are correctly identified regardless of inversion or the order notes are played (§29 — broader than §28's M1 row, but an explicit named acceptance criterion this milestone is the natural place to satisfy, since chord detection is entirely engine scope).
- [ ] `music-engine` unit tests pass, with ≥90% coverage on chord/scale/scoring logic (§28, §29).
- [ ] The canonical worked example returns exactly `{missedNotes: ['F'], accuracy: 0.8}` (§29, §13).
- [ ] Key press (mouse, touch, or mapped computer key) produces sound in under ~50ms on a mid-range Android device and a current iPhone (§29).
- [ ] Browsers that block audio until a gesture show a "tap to enable sound" prompt — never a silently broken piano (§29).
- [ ] Mouse, touch, and computer-keyboard input all produce identical engine and audio behavior for the same note (§29).
- [ ] Salamander Grand Piano attribution is visible in the shipped UI and a `NOTICE` file exists in the repo (§12 licensing requirement).

## 9. Definition of Done

Checked against `docs/ROADMAP/DEFINITION_OF_DONE.md`. Item 5 ("security rules not weakened") is not yet load-bearing at M1 — no new Firestore paths are introduced by this milestone beyond what M0 already covers (Free Play does not write any exercise results; there is no scored exercise yet to persist). Item 6 ("no secrets exposed") remains vacuous until M4, same as M0.

## 10. Out of Scope for This Milestone

- **MIDI keyboard input** (`MIDIAdapter`, `navigator.requestMIDIAccess()`) — explicitly Phase 2 (`docs/PRODUCT_SPEC.md` §7, §8, §11). The `NoteEvent` shape and the source-agnostic engine design in this milestone are deliberately built to make adding a `MIDIAdapter` later a pure addition, not a refactor — but no MIDI code, feature-detection stub, or MIDI-specific UI messaging ships in M1.
- **Any lesson content, the Lesson Player screen, or lesson-driven scoring UI** — M2. M1's UI only exercises detection (Free Play's recognition toast); no `Lesson` document, real or seeded, exists yet.
- **Any Firestore write** beyond what M0's anonymous-auth session already implies — Free Play in M1 does not persist anything; there is no `PracticeSession` yet to write, because there is no scored exercise yet.
- **The Note Identification game, scale/chord drill lessons, streak/XP** — M3.
- **Identifying/labeling which specific chord inversion was played** (first inversion vs. second inversion, as distinct pedagogical content) — the detection algorithm is inversion-*tolerant* by construction and must correctly recognize a triad's quality regardless of inversion (that's the §29 acceptance criterion above), but *labeling* a specific inversion as a distinct teaching concept is Phase 2 content (`docs/PRODUCT_SPEC.md` §7's deferred table lists "inversions" under Phase 2). These are not the same requirement — see `docs/TESTING/UNIT_TESTING.md` §6.2 for the same distinction stated on the testing side.
- **Any Gemini/AI involvement** — M4.
- **Additional scales beyond what the engine's generic validation supports being exercised by real content** — the *engine* supports arbitrary `ScaleDefinition`s generically in M1, but *which* scales actually ship as lesson content (C major + 3–4 more) is M3's decision, grounded in `docs/CONTENT/BEGINNER_CURRICULUM.md`.

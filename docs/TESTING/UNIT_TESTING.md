# Unit Testing — `packages/music-engine`

## 1. Purpose

Defines the unit testing approach for `packages/music-engine`, the single highest-value test suite in the repository (`docs/TESTING/TESTING_STRATEGY.md` §3). This engine is the entire proof that the deterministic/AI boundary in `docs/PRODUCT_SPEC.md` §14 is real and not just an architectural intention — Gemini is trustworthy specifically *because* it never touches correctness logic, and that claim only holds if the engine itself is provably correct.

**Source of truth:** `docs/PRODUCT_SPEC.md` §11 (module boundaries), §13 (Music Theory Engine — detection algorithms and scoring rubric), §26 (testing strategy), §29 (acceptance criteria). Contract shapes: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1 (`NoteEvent`), §4.5 (`ExerciseVerdict`), §4.12–4.14 (`ChordDefinition`, `ScaleDefinition`, `Score`/`ScoringWeights`). Engine implementation detail: `docs/MUSIC/MUSIC_ENGINE_SPEC.md` (parallel batch — this document tests the contract that spec implements, not its internals).

## 2. Scope

Covers unit tests for note identification, chord detection (major/minor triads, inversion-tolerant), scale validation (ordered and unordered), and the composite scoring formula — everything inside `packages/music-engine`. Does not cover: Cloud Function server-side re-validation of an `ExerciseAttempt` (that's an integration concern once Firestore/Auth are involved — see `INTEGRATION_TESTING.md`), UI rendering of engine output, or audio playback (`Audio Layer` is a separate module per §11 and is triggered directly by the Input Layer, not by the Engine Core).

## 3. Why Zero Mocking Is Possible Here

`packages/music-engine` is plain TypeScript with **zero** dependency on React, Firebase, DOM, or any Gemini/AI SDK (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5, rule 2 — this is enforced, not aspirational: the package cannot import `firebase/*`, `react`, `next`, or an AI SDK at all). It is a pure function of `(NoteEvent[], Lesson | Exercise) → ExerciseVerdict`. That means every test in this suite:

- Constructs plain-data fixtures (`NoteEvent[]` arrays, a `Lesson` or `Exercise` definition) directly, with no test doubles, no emulator, no network.
- Asserts on the returned `ExerciseVerdict` object's fields directly.
- Runs in milliseconds, in-process, with no setup/teardown beyond constructing the fixture.

If a test in this suite ever needs a mock, that is itself a signal something has leaked a dependency it shouldn't have — treat it as a build-time violation to fix (per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 2), not a testing-pattern question to work around.

## 4. Test Fixture Shape

All fixtures are plain data conforming to the canonical contracts:

```typescript
// Input fixture — a NoteEvent[], per TYPES_AND_CONTRACTS.md §4.1
const playedNotes: NoteEvent[] = [
  { midiNote: 60, type: 'noteOn', velocity: 100, timestamp: 0,    source: 'mouse' }, // C4
  { midiNote: 62, type: 'noteOn', velocity: 100, timestamp: 500,  source: 'mouse' }, // D4
  { midiNote: 64, type: 'noteOn', velocity: 100, timestamp: 1000, source: 'mouse' }, // E4
  { midiNote: 67, type: 'noteOn', velocity: 100, timestamp: 1500, source: 'mouse' }, // G4
];

// Expected-sequence fixture — an ExpectedNote[], per TYPES_AND_CONTRACTS.md §4.2
const expectedSequence: ExpectedNote[] = [
  { midiNote: 60 }, // C
  { midiNote: 62 }, // D
  { midiNote: 64 }, // E
  { midiNote: 65 }, // F
  { midiNote: 67 }, // G
];

// Output fixture — the returned ExerciseVerdict, per TYPES_AND_CONTRACTS.md §4.5
```

No fixture ever needs a `Timestamp` (Firebase's type) — engine-internal timing is plain `number` milliseconds relative to exercise start (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §3). Firestore's `Timestamp` type only appears at the persistence boundary, which is outside this package entirely.

## 5. The Canonical Test Case (Required, Non-Negotiable)

This exact case is quoted in `docs/PRODUCT_SPEC.md` §13 and §29, and restated as the client/server contract in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5. It must exist in the suite verbatim, not paraphrased, and it must be the first test anyone points to when asked "does the engine work":

> **Given** an expected sequence of C, D, E, F, G
> **And** the learner plays C, D, E, G (F omitted)
> **Then** the engine returns `missedNotes: ['F']` and `accuracy: 0.8`

```typescript
describe('scale/melody scoring — canonical example', () => {
  it('reports the single missed note and 0.8 accuracy when one note in a 5-note sequence is skipped', () => {
    const expectedSequence: ExpectedNote[] = [
      { midiNote: 60 }, // C4
      { midiNote: 62 }, // D4
      { midiNote: 64 }, // E4
      { midiNote: 65 }, // F4
      { midiNote: 67 }, // G4
    ];
    const playedEvents: NoteEvent[] = [
      { midiNote: 60, type: 'noteOn', velocity: 100, timestamp: 0,    source: 'mouse' },
      { midiNote: 62, type: 'noteOn', velocity: 100, timestamp: 500,  source: 'mouse' },
      { midiNote: 64, type: 'noteOn', velocity: 100, timestamp: 1000, source: 'mouse' },
      { midiNote: 67, type: 'noteOn', velocity: 100, timestamp: 1500, source: 'mouse' },
    ];

    const verdict = evaluateSequence(playedEvents, expectedSequence, /* ...scoringWeights */);

    expect(verdict.missedNotes).toEqual(['F']);
    expect(verdict.accuracy).toBeCloseTo(0.8, 5);
    expect(verdict.correctNotes).toEqual(['C', 'D', 'E', 'G']);
    expect(verdict.extraNotes).toEqual([]);
  });
});
```

Note the reasoning that makes this assertion exact, worth stating explicitly since it's easy to get subtly wrong: G is in *both* the expected sequence and the played sequence, so it counts as correct, not extra — `extraNotes` is for notes played that were **not** expected at all. `accuracy = correctNotes.length / expectedSequence.length = 4/5 = 0.8` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5). A test that asserts `extraNotes: ['G']` or `accuracy: 4/4` is testing the wrong semantics and should fail review even if it happens to pass against a buggy implementation.

## 6. Required Test Groups

### 6.1 Note identification
- Single note played matches single expected note (exact MIDI number match).
- Note name derivation from MIDI number is correct across at least one full octave (`docs/MUSIC/NOTE_SYSTEM.md` owns the derivation table; this suite tests the engine's use of it, e.g. `60 → "C4"`).
- Out-of-range MIDI values (`< 0` or `> 127`) from a misbehaving adapter are rejected (logged, not thrown) per the validation rule in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1 — assert the engine does not crash and does not count the invalid event toward the verdict.

### 6.2 Chord detection — the §29 acceptance criterion
- **All 12 major triads** and **all 12 minor triads** are correctly identified as their quality (`major`/`minor` per `ChordDefinition.quality`), parameterized across all 12 roots — this is a direct, named acceptance criterion (`docs/PRODUCT_SPEC.md` §29: "All 12 major and 12 minor triads are correctly identified regardless of inversion or the order notes are played").
- **Inversion tolerance is inherent, not a separate feature to test as an afterthought:** because detection is pitch-class-set matching (`(note mod 12)` values compared order-independently against `{0, 4, 7}` for major, tested against all 12 roots — `docs/PRODUCT_SPEC.md` §13), a root-position C major (`C4, E4, G4`) and a first-inversion C major (`E4, G4, C5`) must both resolve to `{ quality: 'major', root: 'C' }`. Include at least one root-position and one inverted fixture per triad quality to make this explicit in the suite, not just implicit in the algorithm.
- **Order independence:** the same three pitch classes played in any order (e.g., `G4, C4, E4` vs. `C4, E4, G4`) must produce an identical chord-detection result.
- **Non-chord input:** an arbitrary set of notes that doesn't match any known interval pattern returns a "no chord recognized" result, not a false positive or a thrown error.
- **Note:** the engine's chord *recognition* capability covers major and minor for MVP (`ChordDefinition.quality` includes `diminished`/`augmented` as defined-but-unused types per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.12). This is broader than the MVP *lesson content* scope, which teaches major triads in root position only (`docs/PRODUCT_SPEC.md` §6) — see `docs/ROADMAP/M1_MUSIC_ENGINE.md` §7 for why both statements are true at once and neither contradicts the other.

### 6.3 Scale validation
- **Ordered comparison** ("play it ascending"): a scale played out of sequence fails validation even if all correct pitch classes were eventually played — test at least one case where all 7 notes of C major are played but in a shuffled order, asserting the ordered check fails while an unordered check on the same input would pass.
- **Unordered comparison** ("did you hit all seven notes"): the complementary case — same shuffled-order input, asserting the unordered check passes.
- **C major plus the 3–4 additional MVP keys** (`docs/PRODUCT_SPEC.md` §7 — exact key list owned by `docs/MUSIC/SCALE_SYSTEM.md` and `docs/CONTENT/BEGINNER_CURRICULUM.md`) each get at least one full-scale-correct and one missed-note fixture.

### 6.4 Scoring rubric — each component tested in isolation, then composed
Per `docs/PRODUCT_SPEC.md` §13's explicit rubric:
- **Note correctness:** `correctNotes.length / expectedSequence.length`, with wrong/extra notes penalized (assert a fixture with extra unexpected notes played produces a lower composite score than an otherwise-identical fixture without them — extra notes are not free).
- **Timing buckets:** per-note deviation from `expectedTimeMs` bucketed as `onTime` (≤100ms), `slightlyOff` (100–250ms), `late`/`early` (>250ms) — test one fixture landing in each bucket, and assert `perNoteTiming` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5) reports every note's individual bucket, not just an averaged score.
- **Completeness:** independent of timing — a note played wildly off-time still counts as "attempted." Test that a very-late note contributes to `completeness` even when it tanks `timingScore`.
- **Consistency:** variance in timing deviation, both within a single attempt and across the learner's last N attempts of the same exercise — this requires a fixture that supplies prior-attempt history; test both the single-attempt variance case and (if the engine's public API accepts prior attempts) the cross-attempt case.
- **Composite score:** `docs/PRODUCT_SPEC.md` §13 gives an illustrative weighting (`0.6 × noteCorrectness + 0.25 × timingScore + 0.15 × consistencyScore`) — but `ScoringWeights` are exercise/lesson metadata, not hardcoded (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14). Test the composite formula generically against **injected** weights (e.g. assert `compositeScore.value` for a set of known component scores and a set of weights matches the documented formula exactly, clamped to `0–100`), plus a specific case using a note-ID-style weighting (`correctness: 1.0, timing: 0, consistency: 0`) to confirm timing is fully ignorable when a lesson's metadata says so.
- **Weights validation:** a `ScoringWeights` fixture whose `correctness + timing + consistency` does not sum to `1.0` (±0.001) should fail validation at the point content is loaded/seeded, not silently produce a skewed score at runtime — test that this is caught (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14; validation ownership is `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`, but the engine-side guard, if any, belongs in this suite).

### 6.5 `passed` determination
- `ExerciseVerdict.passed` is `true` when `compositeScore.value / 100 >= passThreshold` and `false` otherwise — test both sides of the boundary with an exact-equal case included.
- **Open question:** `Lesson.passThreshold` has no ratified default value yet (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2, flagged there as OQ-01 in `docs/OPEN_QUESTIONS.md`). Until that's resolved, write this test group parameterized on an explicit, clearly-labeled placeholder threshold (e.g. `passThreshold: 0.8` used only as an illustrative test fixture value, not a claimed product default) so the *mechanism* is proven correct independent of what the eventual default turns out to be.

## 7. Coverage Target

**≥90% unit test coverage on chord/scale/scoring logic specifically** — this is a named acceptance criterion, not a suggestion (`docs/PRODUCT_SPEC.md` §29: "Unit test coverage on chord/scale/scoring logic is ≥90%"). `docs/TESTING/TESTING_STRATEGY.md` §3 additionally frames this suite's aspiration as "near-100%" given how much of the product's trust argument depends on it — treat 90% as the CI-enforced floor, not the target to stop at. Coverage is measured on `packages/music-engine/src/` specifically; it is not a repo-wide blended number, and a high repo-wide average must never be used to paper over a gap in this package.

## 8. Error Handling in the Engine Itself

- Malformed input (e.g. a `NoteEvent` missing a required field, or a `noteOff` with no matching prior `noteOn`) must not throw an unhandled exception — the engine returns a verdict reflecting what it *could* validate, and logs the anomaly. A piano app whose scoring engine crashes on a stray input is a worse failure mode than a slightly wrong score, given the "learner is never blocked" principle (`docs/PRODUCT_SPEC.md` §2) applies to the engine's own robustness, not just its network independence.
- Empty `NoteEvent[]` (learner attempted nothing) returns a well-formed verdict with `accuracy: 0`, `missedNotes` equal to the full expected sequence, and `passed: false` — not a thrown error or `undefined`.

## 9. Acceptance Criteria (Restated From `docs/PRODUCT_SPEC.md` §29)

- [ ] All 12 major and 12 minor triads are correctly identified regardless of inversion or the order notes are played.
- [ ] The canonical worked example (expected C-D-E-F-G, played C-D-E-G) returns exactly `{missedNotes: ['F'], accuracy: 0.8}`.
- [ ] Unit test coverage on chord/scale/scoring logic is ≥90%.
- [ ] `packages/music-engine` has zero `firebase/*`, `react`, `next`, or AI-SDK imports — enforceable as a lint/dependency-boundary check run alongside this suite (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 2).

## 10. Out of Scope

- Server-side re-validation wiring (the Cloud Function calling into this same engine) — that's `INTEGRATION_TESTING.md`.
- Audio-triggering correctness (does a `NoteEvent` actually produce sound) — that's `E2E_TESTING.md`/`PERFORMANCE_TESTING.md`; the Audio Layer is triggered directly by the Input Layer, not by the Engine Core (`docs/PRODUCT_SPEC.md` §11), so it is architecturally untestable *from within* this suite and shouldn't be forced in.
- MIDI-sourced `NoteEvent`s specifically — the engine is source-agnostic by design (`source` is diagnostic-only, never branched on for scoring per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1), so no MIDI-specific engine test is needed even once MIDI ships in Phase 2; existing fixtures already prove source-independence.

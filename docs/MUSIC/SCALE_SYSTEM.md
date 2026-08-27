# Scale System

**Status:** Draft v1.0 — music engine documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §6, §7, §13, §29, §30; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 (`ExpectedNote`), §4.5 (`ExerciseVerdict`), §4.13 (`ScaleDefinition`)
**Companion documents:** `docs/MUSIC/MUSIC_ENGINE_SPEC.md` (master spec), `docs/MUSIC/NOTE_SYSTEM.md` (derivations this document builds on), `docs/MUSIC/CHORD_SYSTEM.md` (sibling detection algorithm, contrasting octave-invariant vs. this document's octave-specific matching), `docs/MUSIC/SCORING_SYSTEM.md` (how a scale match's correctness feeds `ExerciseVerdict`), `docs/CONTENT/BEGINNER_CURRICULUM.md` (which specific keys are taught), `docs/TESTING/UNIT_TESTING.md` §6.3

---

## 1. Purpose

Define `ScaleDefinition`'s interval table, the transposition method for playing the same scale shape from any root, and — the highest-precision requirement in this document — the exact ordered- and unordered-matching algorithms that produce `ExerciseVerdict`. This document also owns **the canonical worked example the whole scoring spec is built around**: expected `C-D-E-F-G`, played `C-D-E-G`, must produce exactly `missedNotes: ['F']`, `accuracy: 0.8` (`docs/PRODUCT_SPEC.md` §13, §29; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5). Every detail below is written so that number is reproducible by construction, not by coincidence.

## 2. Scope

In scope: the major-scale interval table, transposition to any root, the ordered-sequence matching algorithm (with the canonical worked example derived step by step), the unordered/coverage matching algorithm, and how the two differ and when each applies.

Out of scope: note-name/frequency derivation (`NOTE_SYSTEM.md`), chord detection (`CHORD_SYSTEM.md`), the timing/consistency/composite-score math a scale match's `correctNotes`/`missedNotes`/`extraNotes` feed into (`SCORING_SYSTEM.md`), which specific keys the MVP curriculum teaches beyond what's needed to justify this document's note-naming convention (`docs/CONTENT/BEGINNER_CURRICULUM.md` is authoritative on the lesson list).

## 3. Requirements

- **Ordered comparison when sequence matters** ("play it ascending") — a scale played with the right pitches in the wrong order must be scored differently from the same pitches in the right order (`docs/PRODUCT_SPEC.md` §13).
- **Unordered comparison when it doesn't** ("did you hit all seven notes") — a coverage check, independent of order (§13).
- **The canonical worked example is exact, not approximate:** expected `[C,D,E,F,G]`, played `[C,D,E,G]` → `correctNotes: ['C','D','E','G']`, `missedNotes: ['F']`, `extraNotes: []`, `accuracy: 4/5 = 0.8`. This is quoted directly in `docs/PRODUCT_SPEC.md` §13 and §29 and restated as the client/server contract in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5 — it is not this document's example to alter, only to make precisely reproducible.
- **`accuracy` formula is fixed by `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5** — `accuracy: number = correctNotes.length / expectedSequence.length` — and this document's matching algorithm must produce a `correctNotes` array whose length, divided by the expected sequence's length, equals exactly `0.8` for the canonical example. This constrains the algorithm design directly (§4.2 below).

## 4. Architecture

### 4.1 The `ScaleDefinition` interval table and transposition

Shape from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.13, reproduced for reference:

```typescript
interface ScaleDefinition {
  id: string;
  name: string;
  intervals: number[];
}
```

**Major scale**, fully defined:

```
majorScale.intervals = [0, 2, 4, 5, 7, 9, 11]
```

This is the diatonic major (Ionian) scale's whole-step/half-step pattern (W-W-H-W-W-W-H), expressed as cumulative semitone offsets from the root: `0` (root), `+2` (whole step), `+2` (whole step, cumulative `4`), `+1` (half step, cumulative `5`), `+2` (cumulative `7`), `+2` (cumulative `9`), `+2` (cumulative `11`), then `+1` back to the octave (`12`, not included in `intervals` — the octave note is the next lesson's/loop's root).

**Transposition to any root** — the method `docs/PRODUCT_SPEC.md` §13 gestures at directly: add the target root's `midiNote` to each interval.

```
scaleNotes(rootMidiNote, scaleDefinition) = scaleDefinition.intervals.map(i => rootMidiNote + i)
```

**Worked example — G major** (root = G4 = `67`):

```
scaleNotes(67, majorScale) = [67+0, 67+2, 67+4, 67+5, 67+7, 67+9, 67+11]
                            = [67, 69, 71, 72, 74, 76, 78]
                            = [G4, A4, B4, C5, D5, E5, F#5]
```

This matches the conventional G major scale (`G, A, B, C, D, E, F#`) exactly — the transposed formula produces `F#`, not `F`, which is the entire pedagogical point of a major scale's *interval pattern* (not a fixed set of white keys) being what makes it "major" from any starting note. This is also why `docs/MUSIC/NOTE_SYSTEM.md` §4.4's sharps-only spelling convention was constrained to sharp-side keys: G major's one sharp (`F#`) spells correctly under that convention with no compromise.

**Worked example — D major** (root = D4 = `62`): `[62, 64, 66, 67, 69, 71, 73] = [D4, E4, F#4, G4, A4, B4, C#5]` — two sharps (`F#`, `C#`), again spelling correctly under the sharps-only convention.

**Worked example — A major** (root = A3 = `57`, chosen so the scale doesn't run past a typical visible range): `[57, 59, 61, 62, 64, 66, 68] = [A3, B3, C#4, D4, E4, F#4, G#4]` — three sharps (`C#`, `F#`, `G#`).

These three transpositions (G, D, A) are exactly `docs/CONTENT/BEGINNER_CURRICULUM.md`'s proposed "3 more common keys" beyond C major.

### 4.2 Ordered-sequence matching — the algorithm behind the canonical example

**Precise algorithm** (a two-pointer match with explicit consumption tracking — chosen specifically because a naive "walk both arrays in lockstep" or "greedy played-driven" algorithm produces the *wrong* answer on the canonical example; both wrong alternatives are shown after the correct one, so an implementer who reaches for the obvious approach first can see why it fails):

```
function matchOrdered(expectedSequence: ExpectedNote[], playedNotes: NoteEvent[]):
    consumed = array of false, length = playedNotes.length
    playedPointer = 0
    correctNotes = []
    missedNotes = []

    for expectedNote in expectedSequence (in order):
        found = false
        for i = playedPointer to playedNotes.length - 1:
            if playedNotes[i].midiNote == expectedNote.midiNote and not consumed[i]:
                correctNotes.push(noteName(expectedNote.midiNote))
                consumed[i] = true
                playedPointer = i + 1     // advance past this match (and anything skipped before it)
                found = true
                break
        if not found:
            missedNotes.push(noteName(expectedNote.midiNote))
            // playedPointer does NOT advance — the next expected note still searches from here

    extraNotes = [ noteName(playedNotes[i].midiNote) for i where not consumed[i] ]  // in played order
    accuracy = correctNotes.length / expectedSequence.length

    return { correctNotes, missedNotes, extraNotes, accuracy }
```

**Step-by-step trace of the canonical example** — expected `[C(60), D(62), E(64), F(65), G(67)]`, played `[C(60), D(62), E(64), G(67)]` (F omitted):

| Step | Expected note | Search from `playedPointer` | Result | `playedPointer` after |
|---|---|---|---|---|
| 1 | C (60) | `played[0]=60` → match | `correctNotes = [C]`, `consumed[0]=true` | 1 |
| 2 | D (62) | `played[1]=62` → match | `correctNotes = [C,D]`, `consumed[1]=true` | 2 |
| 3 | E (64) | `played[2]=64` → match | `correctNotes = [C,D,E]`, `consumed[2]=true` | 3 |
| 4 | F (65) | `played[3]=67` ≠ 65; no more played notes | not found | `missedNotes = [F]`; pointer stays at 3 |
| 5 | G (67) | `played[3]=67` → match | `correctNotes = [C,D,E,G]`, `consumed[3]=true` | 4 |

End of expected sequence. `consumed = [true, true, true, true]` → `extraNotes = []`.

**Result:** `correctNotes = ['C','D','E','G']`, `missedNotes = ['F']`, `extraNotes = []`, `accuracy = 4/5 = 0.8`. **Exact match to the canonical example.**

**Why the pointer does not advance on a miss (step 4):** if the pointer advanced past the "failed search" position anyway, the subsequent search for G would start *after* G's actual position and miss it too, incorrectly turning this into a 2-correct/2-missed result instead of the required 4-correct/1-missed result. Not advancing the pointer on a miss is what lets a single skipped note in the middle of a sequence cost exactly one `missedNotes` entry, not a cascade of downstream false misses.

**A second trace, showing extra-note handling** (not in `docs/PRODUCT_SPEC.md`, included here to fully specify the algorithm's behavior for the case the canonical example doesn't exercise) — expected `[C(60), D(62), E(64)]`, played `[C(60), X(61, an unexpected note), D(62), E(64)]`:

- Expected C: `played[0]=60` → match, `consumed[0]=true`, pointer → 1.
- Expected D: search from pointer 1 — `played[1]=61(X)` ≠ 62, not consumed, scan continues — `played[2]=62(D)` → match, `consumed[2]=true`, pointer → 3.
- Expected E: `played[3]=64` → match, `consumed[3]=true`, pointer → 4.
- `consumed = [true, false, true, true]` → index `1` (X) unconsumed → `extraNotes = ['C#4']` (61's note name).
- Result: `correctNotes = ['C','D','E']`, `missedNotes = []`, `extraNotes = ['C#4']`, `accuracy = 3/3 = 1.0`.

This shows extra notes reduce `extraNotes` (and, per `docs/MUSIC/SCORING_SYSTEM.md` §4.1, penalize the *scoring-rubric* correctness component) without reducing `accuracy` itself, which stays a pure `correctNotes.length / expectedSequence.length` ratio per the fixed formula in §3.

**Two algorithms that look plausible but produce the wrong answer on the canonical example** (documented explicitly so an implementer doesn't independently rediscover and ship one of these):

1. **Lockstep index comparison** (`expected[i].midiNote === played[i].midiNote` for each `i`): fails as soon as `played` and `expected` have different lengths at all — comparing `expected[3]=F(65)` against `played[3]=G(67)` would mark index 3 wrong and never correctly recognize that `played[3]` actually matches `expected[4]`. This produces cascading false mismatches for the entire remainder of the sequence, not just one missed note.
2. **Greedy played-driven walk** (iterate `playedNotes` in order; for each played note, check only whether it equals the *next* unconsumed expected note; if not, mark it extra): at the point `played[3]=G(67)` is reached, the "next" unconsumed expected note is `F(65)` — since `67 ≠ 65`, this algorithm marks G as an *extra* note and leaves *both* F and G in `missedNotes`, producing `missedNotes: ['F','G']`, `accuracy: 3/5 = 0.6` — wrong. This algorithm fails because it never looks ahead past the immediately-next expected note before giving up on a played note.

The two-pointer algorithm above avoids both failure modes because it searches *forward through the played array* for each expected note (rather than only checking one position), while still processing the expected sequence strictly in order (preserving the "ordered" semantics §3 requires).

### 4.3 Unordered (coverage) matching

For exercises where order doesn't matter ("did you hit all seven notes" — §13), reduce both sides to sets and compare:

```
function matchUnordered(expectedSequence: ExpectedNote[], playedNotes: NoteEvent[]):
    expectedSet = unique({ n.midiNote for n in expectedSequence })
    playedSet = unique({ e.midiNote for e in playedNotes })

    correctNotes = [ noteName(m) for m in expectedSet if m in playedSet ]
    missedNotes  = [ noteName(m) for m in expectedSet if m not in playedSet ]
    extraNotes   = [ noteName(m) for m in playedSet if m not in expectedSet ]
    accuracy = correctNotes.length / expectedSet.length
```

**Differentiating worked example** — expected `[C(60), D(62), E(64)]` (order matters), played `[E(64), C(60), D(62)]` (right notes, wrong order):

- **Ordered match** (§4.2): expected C → search played from pointer 0 for `60` → `played[0]=64`(no), `played[1]=60` → match, pointer→2. Expected D → search from pointer 2: `played[2]=62` → match, pointer→3. Expected E → search from pointer 3: no more played notes → not found → `missedNotes=['E']`. Final sweep: `played[0]=64` never consumed → `extraNotes=['E4']`. Result: `correctNotes=['C','D']`, `missedNotes=['E']`, `extraNotes=['E4']`, `accuracy = 2/3 ≈ 0.667`.
- **Unordered match** (§4.3), same input: `expectedSet={60,62,64}`, `playedSet={64,60,62}` — identical sets. `correctNotes=['C','D','E']` (all three), `missedNotes=[]`, `extraNotes=[]`, `accuracy=1.0`.

This is the concrete illustration of why the two algorithms are genuinely different tools, not interchangeable: the ordered check correctly penalizes "right notes, wrong order" for an ascending-sequence exercise, while the unordered check correctly rewards "you know all the right notes" for a pure-recall exercise — `docs/MUSIC/SCORING_SYSTEM.md` §4.3's `completeness` metric is conceptually adjacent to this unordered check (see that document for the exact relationship).

### 4.4 Which algorithm applies when

- **Ordered** (§4.2): any lesson/exercise where the instruction is "play it ascending/descending" — this includes every scale lesson in `docs/CONTENT/BEGINNER_CURRICULUM.md` (scales are inherently sequence-shaped) and every melody-type lesson. This is the algorithm behind the canonical worked example.
- **Unordered** (§4.3): a drill/exercise explicitly framed as "find/hit all N notes" without a required order — e.g., a Note ID game round covering a set of notes, or a "review all seven natural notes" style exercise (`docs/CONTENT/BEGINNER_CURRICULUM.md`'s note-identification lessons use this).
- **`type: 'octave'`** — not one of `docs/PRODUCT_SPEC.md`'s own two named cases above, and resolved here rather than left to silent per-lesson guessing: `docs/CONTENT/BEGINNER_CURRICULUM.md`'s one MVP octave lesson (`l06-octaves`) asks the learner to find a note, then the same note an octave away, in that stated order — pedagogically an ordered task, not a coverage check — so `'octave'`-type lessons use **ordered** matching (§4.2), joining `'scale'`/`'melody'` on that side of the split. This is also the specific case that needs `docs/MUSIC/NOTE_SYSTEM.md` §4.5's octave-qualified-name exception, since `l06-octaves`'s expected sequence repeats a pitch class across two octaves.
- **`type: 'orientation'`** — only ever paired with an empty `expectedSequence` in MVP content (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §5.2's auto-pass case), so neither matcher in this document is ever invoked for it; the ordered-vs-unordered question does not arise.
- **`type: 'chord'`** — uses neither matcher in this document; see `docs/MUSIC/CHORD_SYSTEM.md` §6.1 for chord-lesson correctness, which is determined by chord detection, not `ExpectedNote`-by-`ExpectedNote` comparison.
- Which one a given `Lesson`/`Exercise` uses is determined by its `type` and content, not a separate boolean field on `Lesson` — `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` does not currently define an explicit ordering flag on `Lesson`.

⚠ **Open question (NON-BLOCKING):** `Lesson` has no explicit field stating whether a given lesson uses ordered or unordered matching — this document infers it from `Lesson.type` (`'scale'`/`'melody'` → ordered; a subset of `'noteId'` → unordered), which is a reasonable default but not something `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` states outright. If a lesson type ever needs to mix behaviors (e.g., a note-ID lesson that *does* care about order), an explicit `matchMode: 'ordered' | 'unordered'` field on `Lesson`/`Exercise` would remove the inference entirely — flagged as a small, low-risk type addition worth considering rather than silently baking the `type`-based inference in as unstated behavior.

## 5. Data structures

Uses `NoteEvent` and `ExpectedNote[]` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1, §4.2) as input, produces the `correctNotes`/`missedNotes`/`extraNotes`/`accuracy` fields of `ExerciseVerdict` (§4.5) directly — this document's algorithms are the reference implementation for those four fields specifically, for any sequence-shaped (scale or melody) lesson.

## 6. Interfaces / contracts

**Proposed internal function signatures** (`packages/music-engine`'s `scales/` module, `docs/ENGINEERING/PROJECT_STRUCTURE.md` §3 — melody-type lessons reuse `matchOrdered` directly, since a melody is structurally an ordered note sequence with no scale-specific logic beyond that):

```typescript
function matchOrdered(expectedSequence: ExpectedNote[], playedNotes: NoteEvent[]): {
  correctNotes: string[]; missedNotes: string[]; extraNotes: string[]; accuracy: number;
};
function matchUnordered(expectedSequence: ExpectedNote[], playedNotes: NoteEvent[]): {
  correctNotes: string[]; missedNotes: string[]; extraNotes: string[]; accuracy: number;
};
function transposeScale(rootMidiNote: number, scale: ScaleDefinition): number[];
```

Neither is the engine's single external public contract (`evaluate`, `docs/MUSIC/MUSIC_ENGINE_SPEC.md` §6) — both are composed by `validation` to build the full `ExerciseVerdict`.

## 7. Dependencies

`docs/MUSIC/NOTE_SYSTEM.md` for note-name derivation; `packages/shared-types` for `NoteEvent`/`ExpectedNote`/`ScaleDefinition`. No network, no framework.

## 8. Error handling

- An empty `expectedSequence` (only relevant to non-scale lesson types — see `docs/CONTENT/LESSON_ARCHITECTURE.md` §Architecture for the orientation-lesson zero-length case; a scale lesson's `expectedSequence` is never legitimately empty) is not this document's concern to special-case beyond returning `accuracy: correctNotes.length / expectedSequence.length` cleanly — `docs/MUSIC/MUSIC_ENGINE_SPEC.md` §Error handling owns the general empty-sequence guard applied before either matcher here is invoked.
- Empty `playedNotes` (learner attempted nothing before the exercise timed out or was submitted) returns `correctNotes: []`, `missedNotes` equal to the full expected sequence's note names, `extraNotes: []`, `accuracy: 0` — a well-formed verdict, not a thrown error (`docs/TESTING/UNIT_TESTING.md` §8).

## 9. Testing requirements

- The canonical worked example (§4.2), asserted exactly, as the first and most important test in this suite (`docs/TESTING/UNIT_TESTING.md` §5 — required, non-negotiable, "must exist in the suite verbatim, not paraphrased").
- The extra-note trace (§4.2) and the ordered-vs-unordered differentiating example (§4.3), both as explicit fixtures.
- G, D, and A major transposition (§4.1), each asserted against its fully-spelled-out note-name sequence.
- Both documented wrong algorithms (§4.2) are worth a regression-style comment in the actual test file pointing back to this document, so a future refactor that accidentally reintroduces lockstep or greedy matching is caught by the canonical test failing, not silently shipped.

## 10. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §13, §29 (exact, non-negotiable):
- [ ] Expected `C-D-E-F-G`, played `C-D-E-G` → exactly `{missedNotes: ['F'], accuracy: 0.8}` (and, per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5's fuller shape, `correctNotes: ['C','D','E','G']`, `extraNotes: []`).
- [ ] A shuffled-order full scale fails ordered validation but passes unordered validation on the same input (§4.3).

## 11. Out of scope

- Note-name/frequency derivation → `docs/MUSIC/NOTE_SYSTEM.md`
- Chord detection → `docs/MUSIC/CHORD_SYSTEM.md`
- Timing buckets, completeness, consistency, composite score → `docs/MUSIC/SCORING_SYSTEM.md`
- Which specific keys/lessons teach which scale → `docs/CONTENT/BEGINNER_CURRICULUM.md`

# Note System

**Status:** Draft v1.0 — music engine documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §11, §13; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §3 (Conventions), §4.1 (`NoteEvent`)
**Companion documents:** `docs/MUSIC/MUSIC_ENGINE_SPEC.md` (the master spec this document is a module of), `docs/MUSIC/CHORD_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/SCORING_SYSTEM.md` (siblings, all consume the derivations defined here), `docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` (module boundaries — this document owns the derivation rules it explicitly defers to `docs/MUSIC/`)

---

## 1. Purpose

Define the single, canonical representation of pitch used everywhere in AI Piano Tutor — the MIDI note number — and the deterministic rules for deriving everything else (note name, octave, frequency) from it. Every chord/scale detection algorithm (`CHORD_SYSTEM.md`, `SCALE_SYSTEM.md`) and the scoring rubric (`SCORING_SYSTEM.md`) build directly on the conventions fixed here. Nothing in this document is a matter of taste — `docs/PRODUCT_SPEC.md` §13 states the canonical representation and formula directly; this document expands them into implementation-ready derivations with worked examples.

## 2. Scope

In scope: the MIDI note number range and what it represents, the frequency formula and worked examples, note-name and octave derivation, and the rule that comparison always happens on the integer `midiNote`, never on a derived string.

Out of scope: chord/scale pitch-class-set matching (`CHORD_SYSTEM.md`, `SCALE_SYSTEM.md` — those documents consume this one's derivations but own their own algorithms), timing/scoring math (`SCORING_SYSTEM.md`), how a `NoteEvent` is produced by an input adapter (`INPUT_SYSTEM.md`), the computer-keyboard mapping table (`KEYBOARD_MAPPING.md`).

## 3. Requirements

- **MIDI note numbers are the sole canonical representation of pitch**, everywhere in this codebase — client, server, and every shared contract (`docs/PRODUCT_SPEC.md` §13: "MIDI note numbers (0–127) as the single internal source of truth"; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §3: "MIDI note numbers, not note-name strings, are the canonical representation of pitch everywhere in this codebase").
- **Note name and octave are derived display values, never stored or compared as source of truth.** They exist only for rendering to the learner (key labels, feedback text, `ExerciseVerdict.correctNotes`/`missedNotes`/`extraNotes` — which are note-name strings *for display*, computed once, at the moment the verdict is built, from the underlying `midiNote` comparison that already happened).
- **Note comparison is always done on the integer `midiNote`, never on a derived name string.** Comparing `"C#4" === "Db4"` (the same pitch, spelled two different ways) is a real, avoidable bug class that string-based comparison invites and integer comparison makes structurally impossible. Every algorithm in `CHORD_SYSTEM.md` and `SCALE_SYSTEM.md` operates on `midiNote` integers (or `midiNote mod 12` pitch classes); note names are computed only at the point of producing output for a human to read.

## 4. Architecture

### 4.1 MIDI note number range

`midiNote` is an integer in `[0, 127]` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1). Middle C is `60`, scientific pitch notation `C4`. This anchor value is load-bearing for every derivation below and is referenced throughout the product (`docs/PRODUCT_SPEC.md` §5's "show me where Middle C is" user story; `docs/CONTENT/BEGINNER_CURRICULUM.md`'s first lessons).

The full MIDI range spans `C-1` (MIDI `0`) to `G9` (MIDI `127`) in scientific pitch notation — most acoustic and virtual pianos cover only a subset of this (a standard 88-key piano spans MIDI `21`–`108`, `A0`–`C8`); AI Piano Tutor's virtual keyboard shows a much smaller visible slice at any one time (`docs/PRODUCT_SPEC.md` §9: "1–1.5 octaves at a time with horizontal pan" on phones), but the underlying representation is never restricted below the full `[0, 127]` range — a `NoteEvent` from any adapter, including a future MIDI keyboard, must be accepted anywhere in that full range.

### 4.2 Frequency derivation

**Formula** (`docs/PRODUCT_SPEC.md` §13, restated verbatim): 

```
frequency(midiNote) = 440 × 2^((midiNote − 69) / 12)
```

This is the standard 12-tone-equal-temperament formula, anchored on `A4 = 69 = 440 Hz` (concert pitch A). Every other frequency is a power-of-two multiple or fraction of 440 Hz, spaced geometrically across the 12 semitones of an octave.

**Worked examples** (computed to 5 decimal places, displayed rounded to 2 for a UI):

| `midiNote` | Note name | Exponent `(midi−69)/12` | `frequency` (Hz) |
|---|---|---|---|
| 69 | A4 | `0/12 = 0` | `440 × 2^0 = 440.00000` — exact |
| 60 | C4 (Middle C) | `−9/12 = −0.75` | `440 × 2^−0.75 = 261.62557` → **261.63 Hz** |
| 57 | A3 | `−12/12 = −1` | `440 × 2^−1 = 220.00000` — exact (one octave below A4; frequency exactly halves per octave down, a useful sanity check for any implementation) |
| 64 | E4 | `−5/12 ≈ −0.41667` | `440 × 2^−0.41667 = 329.62756` → **329.63 Hz** |

The A3 example is deliberately included alongside A4 to make the octave-doubling relationship explicit and testable: `frequency(midiNote − 12) = frequency(midiNote) / 2` exactly, for any `midiNote`, independent of the exponential formula's rounding — a cheap unit-test invariant (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §Testing requirements).

**Reference table — one full octave (C4–C5)**, for implementers who want copy-paste-verifiable fixture values rather than re-deriving them:

| `midiNote` | Note | `frequency` (Hz, rounded 2dp) |
|---|---|---|
| 60 | C4 | 261.63 |
| 61 | C#4 | 277.18 |
| 62 | D4 | 293.66 |
| 63 | D#4 | 311.13 |
| 64 | E4 | 329.63 |
| 65 | F4 | 349.23 |
| 66 | F#4 | 369.99 |
| 67 | G4 | 392.00 |
| 68 | G#4 | 415.30 |
| 69 | A4 | 440.00 |
| 70 | A#4 | 466.16 |
| 71 | B4 | 493.88 |
| 72 | C5 | 523.25 |

**Where frequency is actually used:** the Audio Layer (Tone.js + sampled piano) does not need this formula to produce sound — `Tone.Sampler` maps `midiNote`-derived note names directly to sample files and handles pitch-shifting between sample points internally (`docs/ARCHITECTURE/AUDIO_ARCHITECTURE.md`). This formula's role in the product is for anywhere `packages/music-engine` or a UI component needs a numeric frequency value directly (e.g., a possible future visualization, or Gemini-facing theory-explanation content that states a frequency fact) — it is defined here as the single canonical source so no second, possibly-inconsistent implementation appears elsewhere.

### 4.3 Note name derivation

**Pitch class** (`0`–`11`) is `midiNote mod 12`, mapped to letter names using sharps only:

```
pitchClassNames = ["C", "C#", "D", "D#", "E", "F", "F#", "G", "G#", "A", "A#", "B"]
//                  0     1    2    3    4    5    6    7    8    9   10    11
```

**Octave** is `Math.floor(midiNote / 12) − 1` (integer division, then subtract one — this `−1` offset is what makes MIDI `60` come out as octave `4`, matching the historical convention that scientific pitch notation's octave numbering starts at `−1` for the lowest MIDI octave).

**Note name** = `pitchClassNames[midiNote mod 12] + octave`.

**Worked examples:**

| `midiNote` | `midiNote mod 12` | pitch class name | `floor(midiNote/12) − 1` | Note name |
|---|---|---|---|---|
| 60 | 0 | C | `5 − 1 = 4` | `C4` |
| 61 | 1 | C# | 4 | `C#4` |
| 69 | 9 | A | `5 − 1 = 4` | `A4` |
| 0 | 0 | C | `0 − 1 = −1` | `C-1` (lowest possible MIDI note) |
| 127 | 7 | G | `10 − 1 = 9` | `G9` (highest possible MIDI note) |

### 4.4 Sharps-only spelling — a stated convention, PROPOSED

This document defines note names using **sharps only** (`C#`, `D#`, `F#`, `G#`, `A#`) — there is no flat spelling (`Db`, `Eb`, `Gb`, `Ab`, `Bb`) anywhere in the canonical derivation. `docs/PRODUCT_SPEC.md` does not state this explicitly; it is a necessary implementation decision this document makes and flags clearly rather than leaving ambiguous, since `ExerciseVerdict.correctNotes`/`missedNotes`/`extraNotes` are note-name **strings** (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5) and some single, deterministic spelling has to be chosen for every pitch class.

⚠ **Open question (NON-BLOCKING):** sharps-only spelling is musically *correct* for C major and every sharp-side key (G, D, A, E, B, F# major), but musically *incorrect* by convention for flat-side keys (F, B♭, E♭, A♭, D♭, G♭ major — e.g. F major's fourth scale degree is conventionally spelled `B♭`, not `A#`, even though they're the same pitch class). This document resolves the immediate MVP conflict by construction: `docs/CONTENT/BEGINNER_CURRICULUM.md` deliberately selects G, D, and A major as the MVP's "3–4 more common keys" (`docs/PRODUCT_SPEC.md` §7) — all sharp-side keys — specifically so sharps-only spelling is never musically wrong for any MVP lesson. This is a real constraint on curriculum key selection, driven by this naming decision, not an unrelated coincidence — noted explicitly in both documents so a future contributor doesn't add an F-major or B♭-major lesson without first deciding how flat-key spelling should work (a small but real Phase 2 addition: either a key-signature-aware spelling table, or an accepted, documented simplification that the app always shows sharps regardless of key).

### 4.5 Octave-suffix convention for `ExerciseVerdict` note-name arrays — a real gap, resolved here

A second naming question sits alongside §4.4's sharp/flat question and is easy to conflate with it but is actually independent: **does a note-name string in `ExerciseVerdict.correctNotes`/`missedNotes`/`extraNotes` include the octave digit (`"C4"`) or not (`"C"`)?** Neither `docs/PRODUCT_SPEC.md` nor `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` states this in general form. It matters concretely because the two are **not interchangeable**: the mandatory canonical worked example (`docs/PRODUCT_SPEC.md` §13, §29) requires the literal array `missedNotes: ['F']` — no octave suffix — and that exact literal is restated as a non-negotiable acceptance criterion in `docs/PRODUCT_SPEC.md` §29 and `docs/MUSIC/SCALE_SYSTEM.md` §3, §10. `midiToNoteName` (§4.3 above) is not, by itself, the function used to populate these three arrays — it always includes the octave (`60 → "C4"`), which would make the canonical example's required output `['F4']`, contradicting the source spec's own literal acceptance criterion.

**Resolution — the convention every sibling document in this cluster already uses, made explicit and binding here:**

- **`correctNotes` and `missedNotes`** — both anchored to a specific position in the lesson's own `expectedSequence` — render the **bare pitch-class name, no octave suffix** (`"C"`, `"F#"`), by default. This is what makes the canonical example's `missedNotes: ['F']` correct by construction, not by coincidence, and matches every worked trace in `docs/MUSIC/SCALE_SYSTEM.md` §4.2–§4.3.
- **`extraNotes`** — a note with no corresponding expected-sequence entry to borrow context from — renders the **full octave-qualified name** (`"C#4"`), by default. An unexpected note has no lesson-context to make its octave implicit, so leaving it out would make the feedback ("you additionally played C#... somewhere") less actionable than it should be. This matches `docs/MUSIC/SCALE_SYSTEM.md` §4.2's extra-note trace (`extraNotes = ['C#4']`) exactly.
- **Exception — disambiguation within one verdict:** if a single exercise's `expectedSequence` contains the *same pitch class at more than one octave* (relevant only to `Lesson.type: 'octave'` lessons, e.g. `docs/CONTENT/BEGINNER_CURRICULUM.md`'s `l06-octaves`, whose `expectedSequence` is `[C4, C5]`), bare pitch-class names would make two genuinely different expected notes indistinguishable in the verdict (`missedNotes: ['C', 'C']` tells a learner nothing about *which* C). For such an exercise, **every** `correctNotes`/`missedNotes` entry for that verdict is rendered octave-qualified instead (`"C4"`, `"C5"`) — the switch applies to the whole verdict's arrays uniformly, never a per-note mix within the same result, so a learner never sees `['C', 'C5']`.

⚠ **Open question / recommendation (IMPORTANT, not extracted from the spec):** this is this document's own inferred rule, not a statement `docs/PRODUCT_SPEC.md` or `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` makes directly — inferred from the one literal example both source documents commit to, plus the pattern already used consistently (if not previously stated as a rule) throughout `docs/MUSIC/SCALE_SYSTEM.md` and `docs/MUSIC/CHORD_SYSTEM.md`. Flagged for implementer/founder sign-off precisely because it is a real, binding implementation choice with no single canonical source stating it outright — but it is not left silently ambiguous: `docs/MUSIC/SCORING_SYSTEM.md` and `docs/CONTENT/BEGINNER_CURRICULUM.md` both build on exactly this rule.

## 5. Data structures

This document does not define new types. It specifies the pure derivation functions that operate on `midiNote: number` (part of `NoteEvent`, `ExpectedNote` — `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1, §4.2) and produce the note-name `string` values used in `ExerciseVerdict.correctNotes`/`missedNotes`/`extraNotes` (§4.5) and `NoteTimingResult.midiNote`-adjacent display logic.

**Proposed internal function signatures** (implementation detail, not a contract other packages depend on — `packages/music-engine` may name these differently as long as the derivation logic matches this document exactly):

```typescript
function midiToPitchClass(midiNote: number): number;   // midiNote mod 12, result in [0, 11]
function midiToOctave(midiNote: number): number;        // Math.floor(midiNote / 12) - 1
function midiToNoteName(midiNote: number): string;      // e.g. 60 -> "C4"
function midiToFrequency(midiNote: number): number;     // 440 * 2 ** ((midiNote - 69) / 12)
```

## 6. Interfaces / contracts

These four functions are internal to `packages/music-engine`'s `notes/` module (`docs/ENGINEERING/PROJECT_STRUCTURE.md` §3) and are not part of the engine's single external public contract (`evaluate(noteEvents, definition): ExerciseVerdict` — `docs/MUSIC/MUSIC_ENGINE_SPEC.md` §6). They are used internally by `chords/`, `scales/`, and `scoring/` (`docs/MUSIC/CHORD_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/SCORING_SYSTEM.md`) whenever a note name needs to be produced for an `ExerciseVerdict` field, and may reasonably be exported from the package for UI-layer use (e.g., rendering a piano key's accessible name, `docs/PRODUCT_SPEC.md` §24: `"C4, white key"`) — the UI Layer calling into `packages/music-engine` for a pure display-derivation function does not violate the framework-independence rule, since the dependency direction (UI depends on the engine) is exactly the direction already permitted (`docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` §4.1).

## 7. Dependencies

None beyond `packages/shared-types` for the `NoteEvent`/`ExpectedNote` field types being read. No network, no browser globals, no framework — pure arithmetic (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §3).

## 8. Error handling

- `midiNote` values outside `[0, 127]` are rejected at the `NoteEvent` construction boundary (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1: "reject (log, don't crash) out-of-range values from any adapter") — the derivation functions in this document should defensively guard against an out-of-range input reaching them (return a clearly-invalid sentinel or throw a caught-and-logged internal error) rather than silently producing a nonsensical note name (e.g., a negative-octave string) for a value that should never have reached this layer, but must never crash the process.
- Non-integer `midiNote` (a defensive case — the type is `number`, not narrowed to integer at the TypeScript level) should be treated the same as out-of-range: rejected, logged, not silently floored or rounded, since silently coercing `60.5` to `60` would mask an upstream bug in whichever adapter produced it.

## 9. Testing requirements

- All four worked frequency examples (§4.2) as exact unit-test assertions, with a tolerance appropriate to floating-point comparison (e.g., `toBeCloseTo(261.63, 2)`), not exact equality for the non-exact values — A4 and A3 should assert exact equality (`440` and `220`), since those are exact under the formula.
- The octave-doubling invariant — `frequency(midiNote − 12) === frequency(midiNote) / 2` — as a property-style test across a range of `midiNote` values, not just the one worked pair.
- Note-name derivation across at least one full octave (the C4–C5 reference table in §4.2), plus the two boundary cases (`midiNote: 0` → `"C-1"`, `midiNote: 127` → `"G9"`).
- A rejected/logged (not thrown) result for `midiNote: -1`, `midiNote: 128`, and a non-integer input, per §8.

## 10. Acceptance criteria

- [ ] `midiToFrequency(69)` returns exactly `440`.
- [ ] `midiToFrequency(60)` returns `261.63` within a `0.01` tolerance.
- [ ] `midiToNoteName(60)` returns exactly `"C4"`.
- [ ] Every comparison used for chord/scale/melody correctness in `packages/music-engine` operates on `midiNote` (or `midiNote mod 12`), never on a derived note-name string — verifiable by code review of `chords/`, `scales/`, `scoring/` for any `string ===` comparison against a note name (there should be none in a correctness-determining code path).

## 11. Out of scope

- Chord/scale pitch-class-set matching algorithms → `docs/MUSIC/CHORD_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md`
- Timing bucket / scoring math → `docs/MUSIC/SCORING_SYSTEM.md`
- How a `NoteEvent`'s `midiNote` is populated by an input adapter → `docs/MUSIC/INPUT_SYSTEM.md`
- Flat-key spelling / key-signature-aware note naming → explicitly deferred, see §4.4's open question

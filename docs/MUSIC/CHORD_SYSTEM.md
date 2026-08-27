# Chord System

**Status:** Draft v1.0 — music engine documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §6, §7, §13, §29; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.12 (`ChordDefinition`)
**Companion documents:** `docs/MUSIC/MUSIC_ENGINE_SPEC.md` (master spec), `docs/MUSIC/NOTE_SYSTEM.md` (pitch-class/note-name derivations this document builds on), `docs/MUSIC/SCALE_SYSTEM.md` (sibling detection algorithm), `docs/MUSIC/SCORING_SYSTEM.md` (how a chord match feeds into `ExerciseVerdict`), `docs/TESTING/UNIT_TESTING.md` §6.2 (the test suite for this algorithm)

---

## 1. Purpose

Define the deterministic algorithm `packages/music-engine` uses to recognize which chord (if any) was played from a set of simultaneous `NoteEvent`s, and to identify its inversion. This is the algorithm `docs/PRODUCT_SPEC.md` §13 describes at a summary level ("pitch-class set matching... inherently inversion-tolerant since it's order-independent"); this document makes it precise enough to implement without clarification.

## 2. Scope

In scope: the simultaneity window that defines "a chord was played," the pitch-class-set matching algorithm against `ChordDefinition`, the full interval table (including Phase-2-unused entries), inversion identification, and how chord detection is used both for Free Play recognition and for chord-lesson/drill scoring.

Out of scope: note-name/frequency derivation (`NOTE_SYSTEM.md`), scale detection (`SCALE_SYSTEM.md`), the composite scoring formula a chord match feeds into (`SCORING_SYSTEM.md`), how `NoteEvent`s arrive from an input adapter (`INPUT_SYSTEM.md`).

## 3. Requirements

- **Order-independent:** the three (or more) notes of a chord may be played in any order and must still be recognized as the same chord (`docs/PRODUCT_SPEC.md` §13, §29).
- **Octave-doubling-tolerant:** a chord voiced across octaves (e.g., a doubled root) must still be recognized (§13: "the set of `(note mod 12)` values played").
- **Inversion-tolerant for detection, inversion-*identifying* as a secondary step:** recognizing *that* a chord was played does not depend on which note is lowest; identifying *which inversion* is an additional, separate computation (§13).
- **MVP engine capability is major *and* minor triad detection — not major-only.** This is a precise, verified reading of the source spec, stated explicitly here because it is easy to get wrong (see the open question below): `docs/PRODUCT_SPEC.md` §7's MVP bullet list names "major/minor triad detection" directly, and §29's acceptance criteria require, verbatim, "All 12 major and 12 minor triads are correctly identified regardless of inversion or the order notes are played." Both major and minor triad *detection* are therefore MVP engine scope. What is **not** MVP is chord *lesson content*: `docs/PRODUCT_SPEC.md` §6/§7 scope taught chord lessons to "major triads, root position only" (`docs/CONTENT/BEGINNER_CURRICULUM.md` teaches no minor-triad lesson). The engine can correctly recognize a minor triad in MVP; the curriculum simply never asks a learner to play one on purpose yet.

  ⚠ **Open question (NON-BLOCKING — clarifying the task brief against the primary source):** the brief for this documentation batch summarized MVP chord scope as "major triads only," implying minor-triad detection itself was Phase 2. Cross-checking directly against `docs/PRODUCT_SPEC.md` §7 and §29 (both cited above) shows minor-triad *detection* is explicitly required in MVP; only minor-triad *lesson content* is Phase 2. This document follows `docs/PRODUCT_SPEC.md` (the authoritative source per this task's own instructions) rather than the summarized brief, and this reading is independently corroborated by `docs/TESTING/UNIT_TESTING.md` §6.2 and `docs/PRODUCT/PRODUCT_REQUIREMENTS.md` (PR-ENGINE-04 / PR-LESSON-14), both written in a parallel batch and reaching the identical conclusion. Flagged so the distinction (detection-capability scope vs. taught-content scope) is never silently lost.

## 4. Architecture

### 4.1 Step 1 — Simultaneity window: what counts as "played together"

`docs/PRODUCT_SPEC.md` §13 says a chord is recognized from "the set of `(note mod 12)` values played within a short time window" but does not give an exact number.

⚠ **PROPOSED, needs validation:** all `NoteEvent`s of `type: 'noteOn'` whose `timestamp` falls within **150ms** of the *first* `noteOn` event in the cluster are considered simultaneous for chord-detection purposes.

**Clustering algorithm, precisely:**

```
cluster = []
for each NoteEvent e (type 'noteOn', in ascending timestamp order):
    if cluster is empty:
        cluster = [e]
    else if e.timestamp - cluster[0].timestamp <= 150:
        cluster.append(e)
    else:
        // cluster is complete — run chord matching (§4.2) on it now
        cluster = [e]  // start a new cluster with this note
```

**Why anchor to the first note in the cluster, not the previous note:** anchoring to the *previous* note ("within 150ms of the last note added") would let a slow arpeggio drift into a single cluster indefinitely — note 1 at `t=0`, note 2 at `t=140`, note 3 at `t=280`, note 4 at `t=420`... each consecutive pair is within 150ms, but the first and last are 420ms apart, which no longer resembles "played together" as a chord. Anchoring to the cluster's first note bounds the entire cluster to a fixed 150ms window regardless of how many notes join it, which better matches the intuitive meaning of "struck as a chord."

**Scope of the window:** only `noteOn` events participate in clustering. `noteOff` events matter for other purposes (sustain, note duration) but not for deciding *which* notes were struck together — a chord is defined by its onsets, not its releases.

**Why 150ms specifically:** a beginner pressing three keys with two hands (or an unpracticed single-hand stretch) rarely lands all three onsets within much less than 50–100ms of each other; 150ms gives comfortable room above that without being so wide that two genuinely separate, deliberately-sequential single-note presses could accidentally cluster into a false chord detection. This is a reasoned starting point, not measured data — `docs/PRODUCT_SPEC.md` §30 itself flags "deterministic scoring feels too rigid (or too loose)" as a risk to watch in early usage and tune post-launch, and this window is exactly the kind of tunable constant that risk applies to.

### 4.2 Step 2 — Pitch-class-set matching against `ChordDefinition`

Once a cluster of simultaneous `noteOn` events is formed (§4.1), reduce it to a **set** of pitch classes (`midiNote mod 12`, deduplicated — octave-doubled notes collapse to one pitch class, per `NOTE_SYSTEM.md` §4.3):

```
playedPitchClasses = unique({ e.midiNote mod 12 for e in cluster })
```

To identify the chord, try every possible root `r` in `[0, 11]`: normalize the played set relative to that candidate root, and check whether the result exactly equals a known `ChordDefinition.intervals` set (§4.3's table).

```
for r in 0..11:
    normalized = unique({ (pc - r + 12) mod 12 for pc in playedPitchClasses })
    for each ChordDefinition def in the table:
        if normalized == def.intervals (as sets):
            return { root: r, definition: def }   // match found
// no match across any root/definition combination -> "no chord recognized"
```

This is inherently **order-independent** (the input is a set) and **octave-doubling-tolerant** (pitch classes, not raw MIDI numbers, are compared) — both properties fall directly out of reducing to a pitch-class set before comparing, exactly as `docs/PRODUCT_SPEC.md` §13 describes.

### 4.3 The `ChordDefinition` interval table

Shape from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.12, reproduced here for reference, not redefined:

```typescript
interface ChordDefinition {
  id: string;
  name: string;
  intervals: number[];   // semitone offsets from root
  quality: 'major' | 'minor' | 'diminished' | 'augmented';
}
```

| `id` | `name` | `intervals` | `quality` | MVP status |
|---|---|---|---|---|
| `major-triad` | Major | `[0, 4, 7]` | `major` | **Active MVP detection.** Also the only MVP *lesson content* chord (`docs/CONTENT/BEGINNER_CURRICULUM.md`). |
| `minor-triad` | Minor | `[0, 3, 7]` | `minor` | **Active MVP detection** (§3's open question). Not taught as lesson content until Phase 2. |
| `diminished-triad` | Diminished | `[0, 3, 6]` | `diminished` | Defined, unused until Phase 2 (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.12: "others defined but unused until Phase 2"). |
| `augmented-triad` | Augmented | `[0, 4, 8]` | `augmented` | Defined, unused until Phase 2. |

**Phase 2 extension chords — interval tables defined now since `ChordDefinition` is designed to hold them, explicitly unused until Phase 2 lesson content and detection are scoped:**

| `id` | `name` | `intervals` | Notes |
|---|---|---|---|
| `dominant-seventh` | Dominant 7th | `[0, 4, 7, 10]` | See the open question immediately below — `quality` has no clean value for this chord. |
| `major-seventh` | Major 7th | `[0, 4, 7, 11]` | Same gap. |
| `minor-seventh` | Minor 7th | `[0, 3, 7, 10]` | Same gap. |

⚠ **Open question (IMPORTANT):** `ChordDefinition.quality` is a closed union of `'major' | 'minor' | 'diminished' | 'augmented'` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.12) — a triad-quality vocabulary. A 7th chord is not cleanly any one of those four (a dominant 7th, for instance, is built on a major triad *plus* a minor-7th extension; it is conventionally its own category, not "major"). As written, `ChordDefinition` has no field that correctly tags a 7th-chord entry. This does not block MVP (7th chords are Phase 2 for both detection and content), but it should be resolved — most likely by widening `quality` to a richer union (e.g. adding `'dominant7' | 'major7' | 'minor7'`) or by splitting "triad quality" and "extension" into two separate fields — before any Phase 2 work seeds a 7th-chord `ChordDefinition` for real. Flagged here rather than silently forcing a 7th-chord entry into one of the four existing quality values, which would misrepresent it.

### 4.4 Step 3 — Inversion identification

Once a chord (root + quality) is matched (§4.2), identifying the inversion is a small, separate step: compare the **lowest-sounding note in the cluster** (by actual `midiNote`, not pitch class — this is the one place in chord detection where absolute octave placement matters) to the matched pitch-class set.

```
lowestNote = min(cluster, by e.midiNote)
lowestPitchClass = lowestNote.midiNote mod 12

if lowestPitchClass == root:              inversion = "root position"
else if lowestPitchClass == (root + intervals[1]) mod 12:   inversion = "first inversion"   // third in bass
else if lowestPitchClass == (root + intervals[2]) mod 12:   inversion = "second inversion"  // fifth in bass
```

MVP lesson content only teaches root-position major triads (`docs/PRODUCT_SPEC.md` §6, §7), so inversion identification's practical use in MVP is limited to Free Play's live recognition (a learner might voice a chord in first inversion during free exploration and the toast should still correctly say "C major" — `docs/PRODUCT_SPEC.md` §10) and, potentially, `LessonProgress`/`AIMemorySummary` feedback that a chord-lesson attempt used the right notes but the wrong voicing. The full inversion vocabulary (root/1st/2nd) exists in the engine now specifically because `ChordDefinition` and the detection algorithm already support it at no extra cost, and Phase 2 inversion-aware lesson content (`docs/PRODUCT_SPEC.md` §8) can build on it directly rather than needing new engine work.

### 4.5 Full worked examples (canonical for this document) — a correct match and an incorrect one

**Played:** `{60, 64, 67}` — C4, E4, G4, struck within 150ms of each other, in any order.

1. **Cluster** (§4.1): all three `noteOn` events fall within the 150ms window → one cluster of 3 notes.
2. **Pitch classes** (§4.2): `{60 mod 12, 64 mod 12, 67 mod 12} = {0, 4, 7}`.
3. **Try each root `r` in `[0, 11]`:**
   - `r = 0`: normalize `{0, 4, 7}` relative to `0` → `{0, 4, 7}`. Compare against every `ChordDefinition.intervals`: matches `major-triad`'s `[0, 4, 7]` exactly. **Match found: root = 0 (C), quality = major.**
4. **Result: "C Major."**
5. **Inversion** (§4.4): lowest note played = `60` (C) → pitch class `0` = the root itself → **root position.**

**Variation 1 — reordered, octave-doubled:** played `{67, 60, 64, 72}` (G4, C4, E4, C5 — root doubled an octave up, played in a scrambled order). Pitch classes: `{67 mod 12, 60 mod 12, 64 mod 12, 72 mod 12} = {7, 0, 4, 0} → unique {0, 4, 7}` — identical set. Same match: **C Major, root position** (lowest note is still `60`, pitch class `0`).

**Variation 2 — first inversion:** played `{64, 67, 72}` (E4, G4, C5). Pitch classes: `{4, 7, 0}` — same set `{0, 4, 7}`, same match (root = C, quality = major). Lowest note = `64` (E), pitch class `4` = `(root 0 + intervals[1]=4) mod 12` → **first inversion.**

**Example 2 — an incorrect chord match**, in a lesson-scoring context (`docs/CONTENT/BEGINNER_CURRICULUM.md`'s `l14-intro-to-major-triads`, target: C major triad, `expectedSequence = [C4(60), E4(64), G4(67)]`): the learner intends a C major triad but plays the third a semitone flat, landing on a C **minor** triad instead.

**Input NoteEvents:** `noteOn` events for `{60, 63, 67}` — C4, D#4/E♭4, G4 (i.e. `E4` replaced by `D#4`), struck within 150ms of each other.

1. **Cluster** (§4.1): all three fall within the window → one cluster.
2. **Pitch classes** (§4.2): `{60 mod 12, 63 mod 12, 67 mod 12} = {0, 3, 7}`.
3. **Try each root:** `r = 0`: normalize `{0, 3, 7}` relative to `0` → `{0, 3, 7}`. This matches `minor-triad`'s `[0, 3, 7]` exactly (§4.3's table) — **not** `major-triad`'s `[0, 4, 7]`. **Match found: root = 0 (C), quality = minor.**
4. **Result: a real, validly-detected chord — "C Minor" — which is not the lesson's target chord** (target: root 0, quality `major`; detected: root 0, quality `minor`). Per §6.1, the chord-level match against the lesson's target **fails** (`root` matches, `quality` does not) — this attempt does not count as "the target chord was played."

**Resolving the failed chord match into `ExerciseVerdict.correctNotes`/`missedNotes`/`extraNotes` — a necessary extension of §6.1, stated explicitly here because §6.1 specifies the pass/fail chord comparison but not this finer-grained breakdown:** when the detected chord (or lack of one) does not match the lesson's target, the engine falls back to a plain unordered `midiNote` comparison (`docs/MUSIC/SCALE_SYSTEM.md` §4.3) between the target's concrete `expectedSequence` and the notes actually played, so a learner gets granular, note-level feedback rather than an undifferentiated "wrong chord":

| Field | Value |
|---|---|
| `correctNotes` | `['C', 'G']` — both present in both the expected set (`{60, 64, 67}`) and the played set (`{60, 63, 67}`) |
| `missedNotes` | `['E']` — `64` was expected, not played |
| `extraNotes` | `['D#4']` — `63` was played, not expected (octave-qualified, `docs/MUSIC/NOTE_SYSTEM.md` §4.5) |
| `accuracy` | `0.667` (`2/3`) |

This is a deliberate, pedagogically important design point: the learner played a real, valid, recognizable chord (C minor is not "random noise" — it is confidently, correctly detected as a chord in its own right, §4.2), and the feedback correctly isolates exactly the one note that needs to change (the third: flatten instead of natural, i.e. "you played D#/E♭ where E was expected") rather than treating the whole attempt as an undifferentiated failure. This is the concrete difference between "wrong" and "here's specifically what to fix" that `docs/PRODUCT_SPEC.md` §5's user stories require of the engine generally (`docs/MUSIC/SCORING_SYSTEM.md` §4.3 makes the identical point for scale attempts).

⚠ **Open question (IMPORTANT — a necessary extension of §6.1's open question, not previously specified anywhere):** this fallback-to-per-note-comparison behavior for a failed chord match is this document's own proposed resolution, not stated in `docs/PRODUCT_SPEC.md` or `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`. The alternative — treating a failed chord match as all-or-nothing (`missedNotes` = the full expected set, `extraNotes` = the full played set, regardless of any pitch-class overlap) — is simpler to implement but throws away exactly the granular "you got two of three right" signal PR-LESSON-10's spirit (named for scales, but reads as a general product principle) argues for. Flagged for the same implementer/founder sign-off as §6.1's open question, since both concern the same underlying "how does a `'chord'`-type lesson populate the field-level parts of `ExerciseVerdict`" decision and should be ratified together, not independently.

**A second, simpler incorrect case, for completeness:** a cluster whose pitch-class set matches **no** `ChordDefinition` at any root (e.g., played `{60, 61, 63}` — C4, C#4, D#4, a tight cluster with no triadic structure) returns "no chord recognized" (§8) — for a `'chord'`-type lesson, this is treated identically to the C-minor case above: the chord-level match fails, and the same per-note fallback applies (`correctNotes: ['C']`, `missedNotes: ['E', 'G']`, `extraNotes: ['C#4', 'D#4']`, `accuracy: 0.333`) — there is no third code path for "no chord at all" versus "the wrong chord": both are simply "the detected chord (or lack of one) did not match the target."

## 5. Data structures

Uses `NoteEvent` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1) as input and `ChordDefinition` (§4.12, reproduced in §4.3 above) as the static match table. Produces a chord-match result consumed by `docs/MUSIC/SCORING_SYSTEM.md` (for chord-lesson `ExerciseVerdict` computation) and, for Free Play, a lightweight `{ chordName: string, inversion: string }`-shaped result surfaced directly to the UI as a toast (`docs/PRODUCT_SPEC.md` §10) — this Free Play result shape is UI-facing convenience, not one of the fifteen canonical contracts, and is not defined in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`.

## 6. Interfaces / contracts

**Proposed internal function signature** (implementation detail of `packages/music-engine`'s `chords/` module, `docs/ENGINEERING/PROJECT_STRUCTURE.md` §3):

```typescript
interface ChordMatch {
  definition: ChordDefinition;
  root: number;              // pitch class, 0-11
  inversion: 'root' | 'first' | 'second';
  lowestMidiNote: number;
}

function detectChord(cluster: NoteEvent[]): ChordMatch | null;
function clusterSimultaneousNoteOns(noteEvents: NoteEvent[]): NoteEvent[][];
```

Neither function is part of the engine's single external public contract (`evaluate(noteEvents, definition): ExerciseVerdict`, `docs/MUSIC/MUSIC_ENGINE_SPEC.md` §6) — they are internal helpers `scoring/` and `validation` compose, and (per `docs/MUSIC/NOTE_SYSTEM.md` §6's precedent) may reasonably be exported for the Free Play UI's live-recognition feature to call directly.

### 6.1 How chord detection is used for lesson/drill scoring

For a `Lesson`/`Exercise` of `type: 'chord'`, this document's recommended (and, absent contrary information in `docs/PRODUCT_SPEC.md`, most textually consistent) design is: correctness is determined by **chord detection** (this document's algorithm — pitch-class-set match against the lesson's target `ChordDefinition` + root), not by exact `midiNote`-by-`midiNote` comparison against `Lesson.expectedSequence`. Concretely: cluster the learner's `noteOn` events (§4.1), run chord detection (§4.2), and compare the *result* (root + quality) against the lesson's target chord — a learner who plays the right three pitch classes in a different octave than the demonstration is correct, per §13's explicit framing of chord recognition as inherently octave/order-tolerant; a learner who plays the right pitch classes in an unexpected register is not penalized as if they'd played entirely different notes.

`Lesson.expectedSequence` for a chord-type lesson still lists concrete `ExpectedNote`s (e.g. `[{midiNote: 60}, {midiNote: 64}, {midiNote: 67}]` for a "C Major Triad" lesson) — this anchors the demonstration playback (`docs/CONTENT/LESSON_ARCHITECTURE.md`) to one specific voicing the learner hears and sees, and gives `correctNotes`/`missedNotes`/`extraNotes` note-name strings to report even though the underlying match is octave-invariant.

⚠ **Open question (IMPORTANT — flagged for implementer sign-off):** neither `docs/PRODUCT_SPEC.md` nor `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` states explicitly whether chord-type lesson scoring uses chord-detection matching (this document's recommendation) or plain `ExpectedNote`-by-`midiNote` matching (the same mechanism used for melody/scale lessons, `docs/MUSIC/SCALE_SYSTEM.md` §4). The two produce different results whenever a learner plays the right chord in the wrong octave. This document's recommendation is stated above and is the pedagogically defensible default (rewarding "you built the right chord" over "you hit these exact key positions," consistent with §13's own framing of chord detection as octave-tolerant by design) — but it is a real scoring-behavior decision worth explicit confirmation before `packages/music-engine`'s `validation` module is implemented, not something to leave two different implementers to resolve differently.

## 7. Dependencies

`docs/MUSIC/NOTE_SYSTEM.md` for pitch-class derivation; `packages/shared-types` for `NoteEvent`/`ChordDefinition`. No network, no framework (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §3).

## 8. Error handling

- An empty cluster (no `noteOn` events at all in an attempt) returns "no chord recognized," not an error.
- A cluster whose pitch-class set matches no `ChordDefinition` at any root returns "no chord recognized" cleanly — this is an expected, common outcome (e.g., a learner mashing random keys during Free Play), not a failure state.
- A cluster containing more than 4 distinct pitch classes with no subset matching any defined chord still returns "no chord recognized" — the algorithm does not attempt partial/fuzzy matching in MVP.

## 9. Testing requirements

Restated and detailed from `docs/TESTING/UNIT_TESTING.md` §6.2:
- All 12 major triads and all 12 minor triads, each correctly identified, parameterized across all 12 roots.
- At least one root-position and one inverted fixture per triad quality (the §4.5 worked example's three variations are a direct template for this).
- Order independence: the same pitch-class set in different playback orders produces identical results.
- Octave-doubling tolerance: a doubled root/third/fifth an octave away does not change the detected chord or (for a non-lowest doubling) the detected inversion.
- A non-chord input (arbitrary unmatched pitch classes) returns "no chord recognized," not a false positive or a thrown error.
- The 150ms clustering boundary itself: two `noteOn` events exactly 150ms apart cluster together; two events 151ms apart do not (an explicit boundary-value test for the PROPOSED constant in §4.1, so the exact threshold is provably what the implementation uses, not an approximation).
- **Example 2's incorrect-match case (§4.5), verbatim** — target C major, played `{60, 63, 67}`, asserting the full `{correctNotes: ['C','G'], missedNotes: ['E'], extraNotes: ['D#4'], accuracy: 0.667}` shape, not just "detection returns C minor" in isolation — this is the fixture that proves the per-note fallback (§4.5's flagged open question) is actually implemented, not just documented.
- A confusable-but-different-quality pair (major vs. minor on the same root, as in Example 2) for every root, not just C — the semitone-third distinction is the single easiest chord-detection bug to introduce (an off-by-one in the interval table) and the easiest to miss if only C is tested.

## 10. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29:
- [ ] All 12 major and 12 minor triads are correctly identified regardless of inversion or the order notes are played.
- [ ] The full worked example (§4.5): played `{60, 64, 67}` in any order/octave-doubling → detected "C Major," root position.
- [ ] Example 2's incorrect-match case (§4.5): played `{60, 63, 67}` against a C-major target → detected chord is "C Minor" (not the target), and `ExerciseVerdict` reports exactly `{correctNotes: ['C','G'], missedNotes: ['E'], extraNotes: ['D#4'], accuracy: 0.667}`.

## 11. Out of scope

- Note-name/frequency derivation → `docs/MUSIC/NOTE_SYSTEM.md`
- Scale detection → `docs/MUSIC/SCALE_SYSTEM.md`
- How a chord match's correctness feeds into the composite score → `docs/MUSIC/SCORING_SYSTEM.md`
- 7th chords, diminished/augmented lesson content, non-root-position lesson content — all Phase 2 (`docs/PRODUCT_SPEC.md` §8)

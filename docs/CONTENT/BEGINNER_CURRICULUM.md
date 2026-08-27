# Beginner Curriculum

**Status:** Draft v1.0 — content documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §5, §6, §7, §34; `docs/PRODUCT/MVP_SCOPE.md` §2, §6 (the authoritative MVP boundary this document is bound by); `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 (`Lesson`)
**Companion documents:** `docs/CONTENT/LESSON_ARCHITECTURE.md` (how a lesson behaves — unlocking, hints, completion; this document is the content that architecture operates on), `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` (the JSON shape each lesson below is authored in — see that document for `l09-basic-rhythm`'s full worked file), `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`, `docs/MUSIC/NOTE_SYSTEM.md` §4.4–§4.5 (the sharps-only spelling and note-name-format conventions this curriculum's key selection is deliberately built to satisfy), `docs/MUSIC/SCALE_SYSTEM.md` §4.1 (the G/D/A transpositions this document commits to as the MVP's scale keys), `docs/MUSIC/CHORD_SYSTEM.md` §4.5 (the C-major-triad example this curriculum's final lesson reuses), `docs/MUSIC/SCORING_SYSTEM.md` §4.9 (the five canonical worked examples, all built on `l07-first-five-notes` below)

---

## 1. Purpose

Enumerate the actual MVP lesson sequence — every lesson `docs/PRODUCT_SPEC.md` §7 and `docs/PRODUCT/MVP_SCOPE.md` §2 commit to building, in the order a learner takes them, each with a stable `lessonId`, its `level`, its learning objective, and the exercise mechanics (`Lesson.type`, matching mode, note content) it uses. This document is the single place a content author, a seed-script implementer, or an engine implementer can find "what are the 14 MVP lessons, specifically" without re-deriving it from `docs/PRODUCT_SPEC.md`'s narrative prose.

**This document does not invent curriculum content freely — every lesson below is a direct, traceable instance of something `docs/PRODUCT_SPEC.md` §7 or `docs/PRODUCT/MVP_SCOPE.md` §2 already names as in scope.** Where a specific note sequence, key, or grouping had to be chosen and the source documents don't dictate the exact choice (there is no world in which they would — "12–15 lessons covering keyboard orientation → note names → octaves → simple melodies → basic rhythm → intro to major triads" is a topic list, not a lesson-by-lesson script), this document makes that choice explicitly and states the reasoning, consistent with every other document in this batch's practice of never silently inventing scope.

## 2. Scope

In scope: the ordered list of all 14 MVP lessons, their `lessonId`/`level`/`type`/`prerequisiteLessonId`/learning objective/`expectedSequence` content, the seven narrative sections they're grouped into for readability, and the explicit MVP/Phase-2 scope boundary as it applies to curriculum content specifically.

Out of scope: the full JSON file shape for any one lesson (`docs/CONTENT/LESSON_CONTENT_SCHEMA.md`), the unlocking/hint/completion mechanics every lesson runs through (`docs/CONTENT/LESSON_ARCHITECTURE.md`), authored instruction/hint *text* and tone guidance (`docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`), the scoring math a lesson attempt is judged by (`docs/MUSIC/SCORING_SYSTEM.md`), Note ID/scale/chord *drill* content (`Exercise` pool config, static app config, not `Lesson` content — `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.3; not curriculum, and not this document's concern).

## 3. Requirements

- **~12–15 lessons, single linear sequence, no branching** (`docs/PRODUCT_SPEC.md` §7; `docs/PRODUCT/MVP_SCOPE.md` §2). This document lands on **14**.
- **Topic coverage, in order:** keyboard orientation → note names → octaves → simple melodies → basic rhythm → intro to major triads (`docs/PRODUCT_SPEC.md` §7's "brief's Levels 1–6, roughly"; `docs/PRODUCT/PRODUCT_REQUIREMENTS.md` PR-LESSON-01) — **plus** guided scale lessons, C major plus 3–4 more common keys, which `docs/PRODUCT_SPEC.md` §7 scopes as MVP via a separate bullet, not folded into the six-topic narrative list.
- **Chord content is major triads only, root position only** (`docs/PRODUCT/MVP_SCOPE.md` §2; PR-LESSON-11) — no minor, 7th, diminished, augmented, or inverted chords anywhere in this curriculum, even though the *engine* can detect minor triads (`docs/MUSIC/CHORD_SYSTEM.md` §3 — an engine-capability/lesson-content split this document respects by construction: nothing below asks a learner to play a minor triad).
- **Chord instruction builds gradually** from single notes toward a full triad, not introducing the whole triad immediately (`docs/PRODUCT_SPEC.md` §5; PR-LESSON-12).
- **Scale lessons are demonstrated before attempted** (PR-LESSON-09), and on a scale attempt the learner is told which specific note(s) were missed, not just "wrong" (PR-LESSON-05, PR-LESSON-10) — both are `docs/CONTENT/LESSON_ARCHITECTURE.md`/`docs/MUSIC/SCORING_SYSTEM.md` mechanics this document's content triggers, not something this document itself implements.
- **The path ends with a recognizable simple melody and an introduction to major triads** (`docs/PRODUCT/MVP_SCOPE.md` §6, the MVP test's exact closing clause) — melody content appears mid-path (§4.4 below) and the triad lesson is deliberately the curriculum's final lesson, satisfying this clause literally, not just in spirit.
- **Nothing below is Phase 2+ content.** No minor/7th/diminished/augmented chords, no inversions, no scale keys beyond C/G/D/A, no song-learning content, no branching "rusty returner" path — every one of those is explicitly `docs/PRODUCT/MVP_SCOPE.md` §3 deferred territory and appears nowhere in this document, including as a "coming soon" placeholder lesson.

## 4. Architecture

### 4.1 The 14 MVP lessons, in order

| # | `lessonId` | `level` | Section | `type` | Matching | Learning objective |
|---|---|---|---|---|---|---|
| 1 | `l01-keyboard-orientation` | 1 | Orientation | `orientation` | n/a (empty sequence, auto-pass — §4.2 below) | Get oriented to the keyboard's white/black key pattern; learn to spot the groups of 2 and 3 black keys as a visual landmark for finding any note by sight. |
| 2 | `l02-find-middle-c` | 2 | Orientation | `noteId` | unordered (single note) | Find and confirm Middle C using the black-key-group landmark from L1 — a glow-and-confirm interaction (PR-LESSON-02) — establishing the learner's first anchor point on the keyboard. |
| 3 | `l03-note-names-c-d-e` | 3 | Note Names | `noteId` | unordered | Learn and recall the note names C, D, E by matching name to key. |
| 4 | `l04-note-names-f-g-a-b` | 4 | Note Names | `noteId` | unordered | Learn and recall F, G, A, B — completing all seven natural note names within one octave. |
| 5 | `l05-note-recall-review` | 5 | Note Names | `noteId` | unordered | Consolidate all seven natural notes in one review drill — "did you hit all seven notes," independent of order. |
| 6 | `l06-octaves` | 6 | Octaves | `octave` | ordered (`docs/MUSIC/SCALE_SYSTEM.md` §4.4) | Understand that a note name repeats every 12 semitones; find Middle C, then the C one octave higher. |
| 7 | `l07-first-five-notes` | 7 | Melodies | `melody` | ordered | Play a short 5-note ascending phrase, C-D-E-F-G, in order — the learner's first melody, and the exact sequence used as the canonical scoring example throughout `docs/MUSIC/` (§4.4 below). |
| 8 | `l08-simple-melody-two` | 8 | Melodies | `melody` | ordered | Play a slightly longer original up-and-down phrase, reinforcing note-order reading beyond a single ascending run. |
| 9 | `l09-basic-rhythm` | 9 | Rhythm | `rhythm` | ordered, timed | Play an already-familiar short phrase in steady quarter-note time — the first lesson where *timing*, not just note order, is scored. |
| 10 | `l10-c-major-scale` | 10 | Scales | `scale` | ordered, timed | Play the C major scale ascending, one octave, root position — the first scale lesson, demonstrated before attempted. |
| 11 | `l11-g-major-scale` | 11 | Scales | `scale` | ordered, timed | Play the G major scale ascending from a new root — reinforcing that "major" is a transposable interval pattern, not a fixed set of white keys. |
| 12 | `l12-d-major-scale` | 12 | Scales | `scale` | ordered, timed | Play the D major scale ascending (two sharps). |
| 13 | `l13-a-major-scale` | 13 | Scales | `scale` | ordered, timed | Play the A major scale ascending (three sharps) — the last of the MVP's four scale keys. |
| 14 | `l14-intro-to-major-triads` | 14 | Triads | `chord` | chord detection (`docs/MUSIC/CHORD_SYSTEM.md` §6.1) | Having heard root, third, and fifth demonstrated individually, build and play a C major triad in root position — the curriculum's capstone lesson. |

`prerequisiteLessonId` is the straight chain `l01 ← null`, `l02 ← l01`, `l03 ← l02`, … `l14 ← l13` — no branches, per §3's requirement and `docs/CONTENT/LESSON_ARCHITECTURE.md` §4.1's unlocking algorithm.

**Seven narrative sections**, for human readability only (not a `Lesson` field — see `docs/CONTENT/LESSON_ARCHITECTURE.md` §4.2 on `level` vs. this grouping): Orientation (L1–2), Note Names (L3–5), Octaves (L6), Melodies (L7–8), Rhythm (L9), Scales (L10–13), Triads (L14). This maps onto `docs/PRODUCT_SPEC.md` §7's six narrative topics (keyboard orientation, note names, octaves, simple melodies, basic rhythm, intro to major triads) with Scales inserted as the seventh, separately-scoped topic, positioned between Rhythm and Triads — after the learner already knows note names/octaves/order (needed to read a scale) and before triads (a scale's notes are a natural on-ramp to hearing a triad as "some of the same notes, played together" rather than an unrelated new concept).

### 4.2 `l01-keyboard-orientation` — the empty-sequence case, by design

`l01`'s `expectedSequence` is deliberately **empty**. This is not an oversight — it is the specific case `docs/MUSIC/MUSIC_ENGINE_SPEC.md` §5.2 documents as the engine's auto-pass path (every scoring field defaults to a fixed full-credit verdict; there is nothing to play or miss). A learner's first-ever screen in the app should carry zero risk of "getting it wrong" before they've touched a single key — L1 is oriented purely toward *looking* at the keyboard and noticing its landmark pattern; the first actual note press (Middle C, scored) is L2, seconds later. Both lessons together are well within `docs/PRODUCT_SPEC.md` §2's north-star window ("play their first correctly-identified note within two minutes").

### 4.3 `l06-octaves` — the octave-qualified-name case, by design

`l06`'s `expectedSequence` — `[C4 (60), C5 (72)]` — deliberately repeats one pitch class at two octaves within a single lesson. This is the one MVP lesson that exercises `docs/MUSIC/NOTE_SYSTEM.md` §4.5's octave-qualified-name exception: because both expected notes are "C," a verdict reporting bare pitch-class names (`missedNotes: ['C', 'C']`, if both were missed) would be genuinely ambiguous about which C the learner missed, so this lesson's verdict renders both `correctNotes`/`missedNotes` entries octave-qualified (`"C4"`, `"C5"`) instead — the one exception case `docs/MUSIC/NOTE_SYSTEM.md` §4.5 anticipates, made concrete here.

### 4.4 `l07-first-five-notes` — the canonical scoring example, by design

`l07`'s `expectedSequence` — `[C4, D4, E4, F4, G4]`, no `expectedTimeMs` — **is**, deliberately and exactly, the canonical worked example quoted throughout this documentation set: `docs/PRODUCT_SPEC.md` §13/§29 ("expected C-D-E-F-G, played C-D-E-G... `{missedNotes: ['F'], accuracy: 0.8}`"), `docs/MUSIC/SCALE_SYSTEM.md` §1/§3/§4.2 (the algorithm's own derivation), and `docs/MUSIC/SCORING_SYSTEM.md` §4.9 (all five of that document's worked examples, including the mandatory one, are built on this exact lesson's expected sequence). This is not a coincidence engineered after the fact — placing the abstract canonical example inside one specific, real, numbered MVP lesson means the example an implementer tests against and the content a learner actually plays are the same thing, not two things that happen to look similar.

### 4.5 Chord content: the gradual build, and why it needs no new `Lesson` mechanics

PR-LESSON-12 requires chord instruction to build "gradually from single notes to simple triads," not introduce a full triad immediately. This curriculum satisfies that requirement through `l14`'s **instructional content and demonstration**, not through additional `Lesson` types or a chain of separate single-note lessons: `l14`'s demonstration plays and highlights the root (C4) alone, then the third (E4) alone, then the fifth (G4) alone, then all three together — narrated by `instructions`/hint text that walks the learner through "this is the bottom note, this is the middle note, this is the top note, now play all three at once" before the scored attempt begins. `Lesson.expectedSequence` for the scored attempt itself is still just the three simultaneous target notes (`[C4, E4, G4]`, matching `docs/MUSIC/CHORD_SYSTEM.md` §4.5's own canonical worked example exactly), because correctness for a `'chord'`-type lesson is judged by chord detection (`docs/MUSIC/CHORD_SYSTEM.md` §6.1), not by the demonstration's internal build-up sequence. The "gradual" requirement is authored pacing within one lesson, not a scoring mechanic — `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` gives the authoring guidance for writing this kind of staged demonstration.

### 4.6 Scale key selection: not a free choice

The MVP's four scale keys — C, G, D, A major — are not chosen independently by this document; they are a direct, binding consequence of two upstream decisions already committed to elsewhere in this batch:

1. `docs/PRODUCT_SPEC.md` §7 requires "C major plus 3–4 more common keys."
2. `docs/MUSIC/NOTE_SYSTEM.md` §4.4 commits, as its stated rationale for the sharps-only note-spelling convention, that the MVP's additional keys must all be **sharp-side** keys (G, D, A, E, B, or F♯ major) — because sharps-only spelling is musically wrong for any flat-side key, and this curriculum is the only thing that decides which keys actually appear.

G, D, and A major are the three sharp-side keys nearest to C on the circle of fifths (one, two, and three sharps respectively) — the gentlest possible transposition step at each stage, and the exact three keys `docs/MUSIC/SCALE_SYSTEM.md` §4.1 already shows fully worked out (`scaleNotes(67, majorScale)`, `scaleNotes(62, majorScale)`, `scaleNotes(57, majorScale)`). This document's `l11`/`l12`/`l13` reuse those exact note sequences rather than re-deriving them, so the curriculum and the engine's own worked examples never drift apart.

⚠ **Open question (NON-BLOCKING):** `docs/PRODUCT_SPEC.md` §7 says "3–4 more common keys" — a range, not a fixed count. This document commits to exactly **3** (G, D, A), landing the curriculum at 14 lessons total, comfortably inside the "~12–15" budget. A 4th scale key (e.g., E major, four sharps) would be a mechanical, low-risk addition — the engine, note-naming convention, and lesson-content schema already support it identically to G/D/A — but would push the total to 15 lessons (still in range) and is left as a content decision for whoever finalizes the curriculum, not preemptively added here without a stated reason to prefer 4 over 3.

### 4.7 What is explicitly not in this curriculum

Restated from §3, made concrete against the actual lesson list above so a future contributor checking "is X already here" gets a direct answer, not an inference: no minor triads, no 7th/diminished/augmented chords, no chord inversions, no scale key beyond C/G/D/A, no two-hand technique, no sight-reading, no song content (public-domain or otherwise — `docs/PRODUCT/MVP_SCOPE.md` §3, gated on a licensing conversation that hasn't happened), and no second/branching path for the "rusty returner" persona (`docs/PRODUCT/USER_PERSONAS.md`) — a returning learner uses the onboarding skill-check (`docs/PRODUCT_SPEC.md` §10) to start further along this same single sequence, not a different one.

## 5. Data structures

Every lesson in §4.1 is an instance of `Lesson` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2), authored as JSON per `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`'s schema — this document specifies *content* (which 14 lessons, what each one's `expectedSequence`/objective is), not the JSON file shape itself or full field-by-field values (`instructions` prose, `hints`, exact `ScoringWeights`) for every lesson; `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` §4.3 shows one lesson (`l09-basic-rhythm`) with every field fully populated as the canonical worked example of the file format.

**`ScoringWeights` by lesson, for completeness** (full reasoning: `docs/MUSIC/SCORING_SYSTEM.md` §4.2, §4.5; `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` §How to choose ScoringWeights):

| Lessons | `ScoringWeights` | Why |
|---|---|---|
| L1–L8 (orientation, noteId, octave, melody — no `expectedTimeMs` on any note) | `{correctness: 1, timing: 0, consistency: 0}` | Timing is never tested when no note carries an `expectedTimeMs` — `docs/MUSIC/SCORING_SYSTEM.md` §4.2 requires `timing`/`consistency` be authored `0` in this case, not left at a nonzero value the engine's default would never actually score against. |
| L9 (`rhythm`) | `{correctness: 0.4, timing: 0.5, consistency: 0.1}` | The one MVP lesson whose entire point is timing — weighted accordingly, per `docs/PRODUCT_SPEC.md` §13's own framing ("a rhythm exercise shifts weight toward timing"). |
| L10–L13 (`scale`) | `{correctness: 0.6, timing: 0.25, consistency: 0.15}` | `docs/PRODUCT_SPEC.md` §13's own default/example weighting — scales are its paradigm case: real notes to get right, evenly-timed ascent as a secondary, real signal. |
| L14 (`chord`) | `{correctness: 1, timing: 0, consistency: 0}` | Chord onset simultaneity is judged by `docs/MUSIC/CHORD_SYSTEM.md` §4.1's 150ms clustering window, not per-note `expectedTimeMs` deviation — this lesson's `ExpectedNote`s carry no `expectedTimeMs`, so the same "no timed notes → timing/consistency = 0" rule as L1–L8 applies (`docs/MUSIC/SCORING_SYSTEM.md` §4.2). |

**`passThreshold`:** every lesson above uses the same value, `0.8`. See §Requirements below for why this is stated as a recommendation, not silently assumed.

⚠ **Recommendation (IMPORTANT — proposed project decision, `OQ-01`):** `Lesson.passThreshold` has no ratified default (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2; tracked as `OQ-01`). This document recommends **`0.8`** as the actual MVP default, uniformly, for all 14 lessons above — not merely as the illustrative placeholder every sibling document in this batch already uses for consistency (`docs/MUSIC/SCORING_SYSTEM.md` §9, §4.9; `docs/TESTING/UNIT_TESTING.md` §6.5; `docs/CONTENT/LESSON_ARCHITECTURE.md` §3), but as a considered proposal: 80% is strict enough that "passing" means something real (a learner who missed half the notes should not unlock the next lesson), while forgiving enough not to wall off a complete beginner who has the right idea but isn't yet precise — consistent with `docs/PRODUCT_SPEC.md` §5's "wrong key press is a normal part of learning, not a failure state" framing, and with `docs/PRODUCT_SPEC.md` §30's own flagged risk ("deterministic scoring feels too rigid (or too loose)") cutting in the forgiving direction for a beginner-facing MVP rather than the strict one. This is a **recommendation requiring explicit ratification**, not a fact extracted from `docs/PRODUCT_SPEC.md` — flag it in any project-decision review rather than treating its presence here as already-decided. A single uniform value across all 14 lessons (rather than a per-lesson-type value) is also this document's recommendation, for the same reason a single global default is simpler to reason about and tune post-launch (`docs/PRODUCT_SPEC.md` §30) than 14 independently-tunable thresholds with no usage data yet to justify varying any of them.

## 6. Interfaces / contracts

This document produces no code-facing interface of its own — it is the authoring source `content/lessons/*.json` (`docs/ENGINEERING/PROJECT_STRUCTURE.md` §3) is written against, loaded by `scripts/seedLessons.ts` into the `lessons` Firestore collection (`docs/DATA/FIRESTORE_SCHEMA.md` §4.4) exactly as `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` specifies.

## 7. Dependencies

`docs/MUSIC/NOTE_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/CHORD_SYSTEM.md` (the note/scale/chord content every `expectedSequence` above is built from), `docs/CONTENT/LESSON_ARCHITECTURE.md` (the mechanics this content runs through), `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` (the file format).

## 8. Error handling

Not applicable in the runtime sense — this document is authoring content, not code. The one integrity requirement: the 14-lesson chain in §4.1 must validate as a single acyclic sequence with strictly increasing `level` at seed time (`docs/CONTENT/LESSON_CONTENT_SCHEMA.md` §Validation rules), exactly like any other seeded lesson content.

## 9. Testing requirements

- A seed-time (not runtime) validation pass over all 14 lessons: unique `lessonId`s, strictly increasing unique `level`s, a single acyclic `prerequisiteLessonId` chain with `l01`'s `null` as the sole root, every `ScoringWeights` summing to `1.0` (§5's table).
- `l07-first-five-notes`'s `expectedSequence` asserted, byte-for-byte, against `docs/MUSIC/SCORING_SYSTEM.md` §4.9's five worked examples' shared input — a content-drift regression test: if someone edits this lesson's notes later, the canonical scoring examples silently stop matching real content, and this is the test that catches it.
- `l06-octaves`'s verdict fields asserted as octave-qualified (`"C4"`, `"C5"`), not bare, per `docs/MUSIC/NOTE_SYSTEM.md` §4.5's exception.
- `l01-keyboard-orientation` asserted to reach `completed` status via the auto-pass path (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §5.2) without going through scored comparison at all.

## 10. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29, §34; `docs/PRODUCT/MVP_SCOPE.md` §6:
- [ ] A learner can progress linearly from `l01` through `l14` with no missing or broken step (§29: "A learner can complete the full MVP lesson sequence with no dead end or broken step").
- [ ] The path, taken as a whole, reaches both a recognizable simple melody (`l07`/`l08`) and an introduction to major triads (`l14`, the final lesson) — `docs/PRODUCT/MVP_SCOPE.md` §6's closing test, satisfied literally by this ordering.
- [ ] Total lesson count is within `docs/PRODUCT_SPEC.md` §7's "~12–15" — this document totals 14.
- [ ] No lesson in this document requires a chord type, scale key, or content topic outside `docs/PRODUCT/MVP_SCOPE.md` §2.

## 11. Out of scope

- Lesson JSON file format → `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`
- Unlocking, hints, demonstration playback, completion mechanics → `docs/CONTENT/LESSON_ARCHITECTURE.md`
- Authored instruction/hint text and tone → `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`
- Note ID game / scale drill / chord drill `Exercise` pool content (static app config, not `Lesson` content) → not this document's concern, per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.3
- Phase 2+ curriculum (full scale set, minor/7th/diminished/augmented chords and inversions, song module, a second learning path) → `docs/PRODUCT/FUTURE_ROADMAP.md`

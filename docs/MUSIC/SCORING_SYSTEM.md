# Scoring System

**Status:** Draft v1.0 — music engine documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §13, §29, §30; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5 (`ExerciseVerdict`), §4.14 (`Score`/`ScoringWeights`)
**Companion documents:** `docs/MUSIC/MUSIC_ENGINE_SPEC.md` (master spec), `docs/MUSIC/NOTE_SYSTEM.md` (§4.5's note-name-format convention, used throughout §4.9), `docs/MUSIC/SCALE_SYSTEM.md` (note-correctness matching this document's correctness component consumes), `docs/MUSIC/CHORD_SYSTEM.md`, `docs/ARCHITECTURE/AI_ARCHITECTURE.md` (the Gemini boundary this document's final rule restates), `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` (how a lesson author chooses `ScoringWeights`), `docs/CONTENT/BEGINNER_CURRICULUM.md` (`l07-first-five-notes`, the real MVP lesson §4.9's five worked examples are all built on), `docs/TESTING/UNIT_TESTING.md` §6.4–§6.5

---

## 1. Purpose

Define the full scoring rubric from `docs/PRODUCT_SPEC.md` §13 in implementation-ready detail: the four components (correctness, timing, completeness, consistency), the composite formula that combines them, and the two rules that bound the entire document — `ScoringWeights` are per-exercise metadata, never hardcoded, and Gemini never computes, invents, or recalculates any number in this document; it only explains one already computed.

**The primary illustration for this whole document is §4.9's set of five worked examples**, anchored on the mandatory canonical case from `docs/PRODUCT_SPEC.md` §13/§29 — expected `C D E F G`, played `C D E G`, verdict `missedNotes: ['F']`, `accuracy: 0.8` — plus four more covering extra notes, a wrong note substituted for an expected one, notes played out of order, and an octave mismatch. Readers who want the exact input/output shape before the derivations may jump there first; §4.1–§4.8 derive the formulas those five examples exercise.

## 2. Scope

In scope: the note-correctness-for-scoring formula (distinct from the raw `accuracy` field — see §4.1), the timing bucket thresholds and per-note deviation classification, the completeness metric, the consistency metric (within-attempt and cross-attempt), the composite score formula and its weighting, and the explicit non-goal of acoustic/microphone pitch detection.

Out of scope: how `correctNotes`/`missedNotes`/`extraNotes` are determined in the first place (`docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/CHORD_SYSTEM.md` — this document consumes their output), note-name derivation (`docs/MUSIC/NOTE_SYSTEM.md`), how `ScoringWeights` should be *chosen* for a given lesson type (`docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` — this document defines the mechanism, that one gives authoring guidance).

## 3. Requirements

- **Note correctness** = correct notes played ÷ required notes, with wrong/extra notes penalized rather than ignored (`docs/PRODUCT_SPEC.md` §13).
- **Timing** = per-note deviation from its expected time, bucketed: `≤100ms` onTime, `100–250ms` slightlyOff, `>250ms` late/early — each bucket surfaced per-note, not just averaged away (§13).
- **Completeness** = were all required notes attempted at all, independent of timing (§13).
- **Consistency** = variance in timing deviation, both within one attempt and across the learner's last N attempts of that exercise (§13).
- **Composite score** = `0.6 × correctness + 0.25 × timing + 0.15 × consistency`, clamped `0–100`. Weights are exercise metadata (`ScoringWeights`), never hardcoded (§13).
- **Gemini never invents or recalculates this number — it only explains it** (§13, §14; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5: "Gemini never produces, modifies, or is asked to sanity-check any field on this object").
- **Explicit non-goal:** this engine does not do acoustic/microphone pitch detection. That is a fundamentally different problem — signal processing on audio, not comparison of symbolic input events — and is scoped as Phase 3 (`docs/PRODUCT_SPEC.md` §8, §13), not something that sneaks into this rubric because it sounds related.

## 4. Architecture

### 4.1 Note correctness — two distinct numbers, not one

`ExerciseVerdict.accuracy` is fixed by `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5 as exactly `correctNotes.length / expectedSequence.length` — a raw, unpenalized ratio. This is the number the canonical worked example (`docs/MUSIC/SCALE_SYSTEM.md` §4.2) reports as `0.8`, and it must never be altered by extra/wrong notes, because the canonical example has zero extra notes and its required result (`0.8`) is exactly `correctNotes.length / expectedSequence.length` with no penalty term.

But `docs/PRODUCT_SPEC.md` §13 also requires "wrong/extra notes penalized rather than ignored" — which `accuracy` alone, as defined, cannot express (it has no term for extra notes at all). This document resolves the apparent tension by treating them as **two different numbers with two different jobs**:

- **`ExerciseVerdict.accuracy`** — a simple, unpenalized ratio, used for direct reporting (the "you got 4 out of 5 notes" framing) and required to match the canonical example exactly.
- **`Score.breakdown.correctness`** (inside `compositeScore`) — the *penalized* correctness input to the composite score formula, distinct from `accuracy`.

⚠ **PROPOSED, needs validation** (not specified by `docs/PRODUCT_SPEC.md` beyond "penalized rather than ignored" — this document proposes a concrete formula):

```
noteCorrectness = max(0, (correctNotes.length − extraNotes.length) / expectedSequence.length)
```

Each extra/wrong note played subtracts one full "correct note" worth of credit from the numerator, floored at zero so a chaotic attempt can't produce a negative score. **Design constraint this formula must satisfy, stated explicitly so any alternative proposal is checked against it:** when `extraNotes.length === 0` (as in the canonical example), `noteCorrectness` must equal `accuracy` exactly — `(4 − 0)/5 = 0.8 = accuracy`. The two numbers only diverge when extra notes are present, which is exactly the case `accuracy`'s fixed formula is silent on and `docs/PRODUCT_SPEC.md` §13 says must be penalized.

**Worked example showing the divergence:** expected `[C,D,E,F,G]`, played `[C,D,E,G,X]` (all four correct notes plus one extra, unexpected note `X`). Using `docs/MUSIC/SCALE_SYSTEM.md`'s ordered matcher: `correctNotes=['C','D','E','G']`, `missedNotes=['F']`, `extraNotes=['X']`. `accuracy = 4/5 = 0.8` (unchanged — extras don't affect it). `noteCorrectness = max(0, (4−1)/5) = 3/5 = 0.6` (penalized — this is the number that flows into the composite score, §4.5).

### 4.2 Timing — bucket thresholds and sign convention

Per-note `deviationMs = actualTimestamp − expectedTimeMs` (positive = played after/late, negative = played before/early). Buckets, resolving the boundary ambiguity in `docs/PRODUCT_SPEC.md` §13's prose (which writes "≤100ms" and "100–250ms," leaving the exact boundary at 100 and 250 needing a non-overlapping definition) as non-overlapping, closed-on-the-tighter-side intervals:

| `bucket` | Condition on `deviationMs` | Meaning |
|---|---|---|
| `onTime` | `−100 ≤ deviationMs ≤ 100` | within a tenth of a second either way |
| `slightlyOff` | `100 < |deviationMs| ≤ 250` | noticeably off, not disqualifying |
| `late` | `deviationMs > 250` | played meaningfully after the expected time |
| `early` | `deviationMs < −250` | played meaningfully before the expected time |

This matches `NoteTimingResult.bucket: 'onTime' | 'slightlyOff' | 'late' | 'early'` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5) exactly — four values, where `onTime` and `slightlyOff` are direction-agnostic (a note 80ms early and a note 80ms late are both `onTime`) and only the far buckets split by direction.

**Worked examples:**

| `expectedTimeMs` | `actualTimestamp` | `deviationMs` | `bucket` |
|---|---|---|---|
| 500 | 540 | `+40` | `onTime` (`|40| ≤ 100`) |
| 1000 | 1180 | `+180` | `slightlyOff` (`100 < 180 ≤ 250`) |
| 2000 | 1700 | `−300` | `early` (`−300 < −250`) |
| 3000 | 3400 | `+400` | `late` (`400 > 250`) |

**Notes with no `expectedTimeMs`** (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2: "omitted for lessons where order matters but timing doesn't") are excluded from timing scoring entirely — they contribute no `NoteTimingResult` entry and are not counted in the `timingScore` average below.

**`timingScore` aggregation** — ⚠ PROPOSED, needs validation (no formula given in `docs/PRODUCT_SPEC.md` beyond the bucket definitions themselves): assign each bucket a point value and average across every note that *has* an `expectedTimeMs`:

```
bucketPoints = { onTime: 1.0, slightlyOff: 0.5, late: 0.0, early: 0.0 }
timingScore = average(bucketPoints[note.bucket] for note in perNoteTiming)
```

If a lesson has **no** timed notes at all (every `ExpectedNote.expectedTimeMs` omitted), `timingScore` defaults to `1.0` by convention — full credit, since timing was never being tested. The same applies to `consistency` (§4.4): with no `NoteTimingResult` entries at all, there is no `deviationMs` to compute variance over, so `consistency` also defaults to `1.0` for such a lesson, by the identical reasoning. Both defaults are a safety net only; `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` requires such a lesson's `ScoringWeights.timing` **and** `ScoringWeights.consistency` to be authored as `0` regardless (with `correctness: 1.0` absorbing the full weight), so neither default value is ever actually multiplied into a visible score in practice (defense in depth against a `0/0` or an accidentally-nonzero timing/consistency weight on an untimed lesson). This also applies to `'chord'`-type lessons: per `docs/MUSIC/CHORD_SYSTEM.md` §4.1, chord onset simultaneity is judged by the 150ms clustering window, not by per-note `expectedTimeMs` deviation, so an MVP chord lesson's `ExpectedNote`s should likewise omit `expectedTimeMs` and its `ScoringWeights.timing`/`consistency` should likewise be `0` — `docs/CONTENT/BEGINNER_CURRICULUM.md`'s `l14-intro-to-major-triads` follows this pattern.

### 4.3 Completeness — attempted, independent of timing

"Were all required notes attempted at all, independent of timing" (§13) is a coverage check, structurally identical to `docs/MUSIC/SCALE_SYSTEM.md` §4.3's **unordered** matcher, applied regardless of whether the lesson itself uses ordered or unordered *correctness* matching:

```
completeness = |unique expected midiNotes that appear anywhere in playedNotes| / |unique expected midiNotes|
```

**Worked example distinguishing `completeness` from `accuracy`** — expected `[C,D,E]` (order matters for this lesson), played `[E,C,D]` (right notes, wrong order). Per `docs/MUSIC/SCALE_SYSTEM.md` §4.3's identical trace: the **ordered** matcher gives `correctNotes=['C','D']`, `missedNotes=['E']`, `accuracy = 2/3 ≈ 0.667` (E was played too late relative to the sequence position search to count). But **completeness**, which ignores order entirely, sees all three expected pitches somewhere in the played set: `completeness = 3/3 = 1.0`. This is deliberate and pedagogically meaningful: a learner who knows all three correct notes but plays them out of order gets full credit on "did you attempt everything" while still being correctly marked down on sequence accuracy — two different signals for two different things the learner needs to know, consistent with `docs/PRODUCT_SPEC.md` §5's "guide, don't punish" framing (partial credit reflects what was actually right).

A note played wildly late (e.g., 2 full seconds after expected) still counts toward `completeness` even though it tanks `timingScore` — this is the specific "independent of timing" property `docs/PRODUCT_SPEC.md` §13 names directly.

### 4.4 Consistency — within-attempt and cross-attempt, and a real contract gap

`docs/PRODUCT_SPEC.md` §13 requires "variance in timing deviation, both within one attempt and across the learner's last N attempts of that exercise."

⚠ **PROPOSED, needs validation:** `N = 5` (the current attempt plus the 4 most recent prior attempts of the same lesson/exercise). Not specified in `docs/PRODUCT_SPEC.md`; chosen as a small enough window to reflect recent practice, not a learner's entire history. When fewer than 5 prior attempts exist (including the learner's very first attempt, where there are zero), the cross-attempt component uses however many are actually available and should not be penalized for a short history — see the resolution below for how this is handled cleanly.

**Within-attempt consistency:** the standard deviation of `deviationMs` across every timed note in the *current* attempt.

**Cross-attempt consistency:** the standard deviation of each of the last `N` attempts' *mean* deviation, i.e. variance of variances at one level up — how consistent the learner's average timing has been, attempt to attempt, not just note to note within one attempt.

**Combining the two into a single `0–1` score** — ⚠ PROPOSED, needs validation (this is the least-specified part of the entire rubric; `docs/PRODUCT_SPEC.md` gives no numbers here at all, only the concept):

```
consistencyComponent(stdDevMs) = clamp(1 − stdDevMs / 200, 0, 1)   // 200ms stdDev or worse -> 0; 0ms stdDev -> 1
consistency = average(consistencyComponent(withinAttemptStdDev), consistencyComponent(crossAttemptStdDev))
```

When there is no prior-attempt history (first-ever attempt), `consistency` uses only `consistencyComponent(withinAttemptStdDev)` — the cross-attempt term is simply omitted from the average rather than defaulted to a guessed value, so a first attempt is judged on its own internal steadiness only. Per `docs/PRODUCT_SPEC.md` §30's own acknowledgment that "deterministic scoring feels too rigid (or too loose)" is a real UX risk to watch, this entire formula — the `200ms` normalization constant especially — should be treated as a tunable starting point pending real usage data, not a fixed constant.

**A genuine contract gap, flagged prominently:** `docs/MUSIC/MUSIC_ENGINE_SPEC.md`'s public contract is a pure function of exactly `(NoteEvent[], Lesson | Exercise) → ExerciseVerdict` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5, restated verbatim across this documentation set). Cross-attempt consistency, by definition, needs the learner's *prior* attempts — data that does not exist inside a single `NoteEvent[]` and is not part of a `Lesson`/`Exercise` definition. A strictly two-argument pure function cannot compute the cross-attempt component at all.

⚠ **Open question (IMPORTANT — flagged, not silently resolved):** this is a real tension between the engine's stated two-argument public contract and §13's own scoring requirement, independently noticed (and left conditionally worded, not resolved) in `docs/TESTING/UNIT_TESTING.md` §6.4: *"this requires a fixture that supplies prior-attempt history; test... the cross-attempt case (if the engine's public API accepts prior attempts)."* This document's proposed resolution, stated so it can be confirmed or overridden by a single implementer decision rather than guessed at independently by whoever writes `validation` and whoever writes `getExerciseFeedback`:

1. **`evaluate(noteEvents, definition): ExerciseVerdict`** — the one true public entry point, exactly as specified — computes everything **within-attempt**, including the within-attempt consistency component, using only its two arguments. This keeps the primary, tested, acceptance-criteria-bound contract (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §6) completely unchanged.
2. **A second, small, exported pure function**, living alongside `evaluate` in `packages/music-engine`'s `scoring/` module — proposed name `scoreConsistencyAcrossAttempts(recentAttemptDeviations: number[][]): number` — takes plain numeric data (each prior attempt's array of `deviationMs` values, or a pre-reduced per-attempt mean) and returns the cross-attempt component alone. It is still framework-independent, still zero-network, still pure (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §3) — it just accepts different input than `evaluate` does.
3. **The Cloud Function** (`getExerciseFeedback`) — which already has Firestore access and, per `docs/ARCHITECTURE/DATA_FLOW.md` §4.4, already loads recent `practiceSessions` for `AIMemorySummary` recomputation in the same request — calls `evaluate()` for the within-attempt verdict, separately calls `scoreConsistencyAcrossAttempts()` with data from those already-loaded prior sessions, and blends the two into the final `consistency` field before writing `PracticeSession`.

This keeps `packages/music-engine` (not the Cloud Function, and never Gemini) as the exclusive owner of every scoring computation — satisfying `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5's ownership statement that `ExerciseVerdict` is "computed exclusively by `packages/music-engine`" — while resolving the real architectural fact that a pure function of only `(NoteEvent[], Lesson | Exercise)` cannot see attempt history. Flagged as needing explicit implementer/founder sign-off before `validation`'s exact function signature is finalized, since two different implementers could otherwise resolve this two different ways.

### 4.5 Composite score formula

```
compositeScore.value = round(clamp(
    weights.correctness × noteCorrectness (§4.1)
  + weights.timing       × timingScore     (§4.2)
  + weights.consistency  × consistency     (§4.4)
, 0, 1) × 100)
```

`weights` is the `Lesson`/`Exercise`'s own `scoringWeights: ScoringWeights` field (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14) — **never** a constant inside `packages/music-engine`. `docs/PRODUCT_SPEC.md` §13's own weighting (`0.6 / 0.25 / 0.15`) is stated there as "an example weighting," which this document treats as the **recommended default** a lesson author starts from (`docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` §How to choose ScoringWeights), not a hardcoded formula constant.

**`Score.breakdown` fields store the raw, unweighted `0–1` component scores** (`noteCorrectness`, `timingScore`, `consistency` as computed above) — ⚠ PROPOSED clarification: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14's comment `// 0-1, weighted contribution` on each `breakdown` field is read here as "the contribution that *gets* weighted" (i.e., the raw component), not "the contribution *already multiplied* by its weight." This matters concretely for any UI displaying a breakdown (e.g., a post-exercise "correctness: 92%, timing: 78%, consistency: 85%" bar chart) — showing the already-weighted, fractional numbers instead would look wrong to a learner (a `0.6`-weighted `92%` correctness would display as `55%`, which is not what "your correctness was 92%" means). This is a small, easily-verified clarification, not a design fork — flagged as **NON-BLOCKING** since either reading is trivial to implement correctly once stated, but worth confirming against `TYPES_AND_CONTRACTS.md`'s author since the two readings are not numerically interchangeable.

**Full worked example** (illustrative synthetic values, not from `docs/PRODUCT_SPEC.md` — included to demonstrate the formula end to end): `noteCorrectness = 0.8`, `timingScore = 0.7`, `consistency = 0.9`, using the default weights `{correctness: 0.6, timing: 0.25, consistency: 0.15}`:

```
raw = 0.6×0.8 + 0.25×0.7 + 0.15×0.9 = 0.48 + 0.175 + 0.135 = 0.79
compositeScore.value = round(clamp(0.79, 0, 1) × 100) = 79
```

**`passed`** = `compositeScore.value / 100 >= Lesson.passThreshold` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5) — `passThreshold` itself has no ratified default value (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2, tracked as `OQ-01` in `docs/OPEN_QUESTIONS.md`); see `docs/CONTENT/LESSON_ARCHITECTURE.md` §Requirements for the full treatment of that gap.

### 4.6 `ScoringWeights` validation

`correctness + timing + consistency` must sum to `1.0` (±0.001 float tolerance) — `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14: "a Lesson/Exercise seed file that fails this check should fail content validation at build/seed time, not at runtime." Full validation mechanics: `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` §Validation rules. This document only states the invariant the formula in §4.5 depends on: if weights don't sum to `1.0`, `compositeScore.value`'s theoretical maximum silently drifts away from `100`, which is exactly the kind of skew build-time validation exists to catch before it reaches a learner.

### 4.7 Non-goal: acoustic pitch detection

This engine does not, and in MVP scope will never, listen to speaker or microphone audio output and infer what was played. Every input to this entire rubric is a symbolic `NoteEvent` — a discrete, already-known `midiNote` + `timestamp` + `velocity`, produced by an input adapter (`docs/MUSIC/INPUT_SYSTEM.md`) — never raw audio. Acoustic pitch detection is a fundamentally different problem (digital signal processing on a waveform, with all the ambiguity and noise that implies) from comparing symbolic events, and is explicitly scoped to Phase 3, gated on validated demand (`docs/PRODUCT_SPEC.md` §8: "Real acoustic-piano pitch detection is a genuinely hard, novel feature — validate demand first"). Nothing in this rubric should be designed as if microphone input were a lightweight future toggle on the same pipeline — it is a separate feature with a separate architecture, not scoped here even at the interface level (contrast with the MIDI input adapter, which *is* designed-for-but-not-built — `docs/MUSIC/INPUT_SYSTEM.md` §4.3).

### 4.8 Gemini never computes this

Restated because it is the rule this entire document exists in service of: every number in `ExerciseVerdict` and `Score` — `accuracy`, `timingScore`, `completeness`, `consistency`, `compositeScore`, `passed` — is produced exclusively by the formulas in this document, executed inside `packages/music-engine`, invoked server-side inside a Cloud Function (`docs/ARCHITECTURE/AI_ARCHITECTURE.md` §4.2). Gemini receives the finished `ExerciseVerdict` as **read-only input** to the Exercise Feedback prompt template (`docs/PRODUCT_SPEC.md` §15) and turns it into 2–3 sentences of natural-language explanation. It never recalculates, sanity-checks, rounds differently, or overrides any field here — not even in the case of an ambiguous or surprising score. If a score looks wrong, the bug is in this document's formulas or their implementation, never something for Gemini to "correct" at explanation time.

### 4.9 Canonical worked examples — five cases, exact input and exact output

All five examples below share the same expected sequence — `docs/CONTENT/BEGINNER_CURRICULUM.md`'s `l07-first-five-notes`, the lesson whose `expectedSequence` **is** the canonical `C-D-E-F-G` case — so the five results are directly comparable to each other and not an artifact of different inputs. `expectedSequence = [C4(60), D4(62), E4(64), F4(65), G4(67)]`, `type: 'melody'`, ordered matching (`docs/MUSIC/SCALE_SYSTEM.md` §4.2), no `expectedTimeMs` on any note (order matters, timing doesn't, per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 — so `perNoteTiming: []`, `timingScore: 1`, `consistency: 1` in every example below; only `correctNotes`/`missedNotes`/`extraNotes`/`accuracy`/`noteCorrectness` vary). Note-name formatting follows `docs/MUSIC/NOTE_SYSTEM.md` §4.5. `noteCorrectness` uses §4.1's proposed formula; `compositeScore` uses the default weights `{correctness: 0.6, timing: 0.25, consistency: 0.15}` (§4.5) for every example except Example 1, which uses this lesson's actual authored weights (`{correctness: 1, timing: 0, consistency: 0}`, since this lesson has no timed notes — §4.2, `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` §How to choose ScoringWeights) to show a non-default weighting applied correctly.

**Example 0 — the mandatory canonical case (verbatim from `docs/PRODUCT_SPEC.md` §13, §29):**

> Expected: C D E F G / Played: C D E G / Expected verdict: `missedNotes = ['F']`, `accuracy = 0.8`.

**Input:** Expected `[C4, D4, E4, F4, G4]`. Played (all `noteOn`, F omitted) `[C4, D4, E4, G4]`.
**Full resulting `ExerciseVerdict`** (fields this document and `docs/MUSIC/SCALE_SYSTEM.md` §4.2 jointly own; `docs/MUSIC/SCALE_SYSTEM.md` §4.2 has the full step-by-step trace):

| Field | Value |
|---|---|
| `correctNotes` | `['C', 'D', 'E', 'G']` |
| `missedNotes` | `['F']` |
| `extraNotes` | `[]` |
| `accuracy` | `0.8` (`4/5`) |
| `noteCorrectness` (§4.1, feeds `compositeScore`) | `0.8` (`(4−0)/5` — no extra notes, so `noteCorrectness = accuracy` exactly, per §4.1's design constraint) |
| `passed` | depends on `passThreshold` (`OQ-01`, unratified) — `true` under the `0.8` illustrative placeholder used consistently across this documentation set (`≥`, not `>`) |

**Example 1 — extra notes played:** the learner plays every expected note correctly, plus one unrequested note.

**Input:** Expected `[C4, D4, E4, F4, G4]`. Played `[C4, D4, E4, F4, G4, C#4]` — all five expected notes, plus an extra `C#4` (61) at the end.

| Field | Value |
|---|---|
| `correctNotes` | `['C', 'D', 'E', 'F', 'G']` |
| `missedNotes` | `[]` |
| `extraNotes` | `['C#4']` (octave-qualified — no expected-sequence anchor, `docs/MUSIC/NOTE_SYSTEM.md` §4.5) |
| `accuracy` | `1.0` (`5/5`) — extras never reduce `accuracy` itself (§4.1) |
| `noteCorrectness` | `0.8` (`max(0, (5−1)/5) = 4/5`) — the one extra note costs exactly one full "correct note" of credit toward the *scoring* number, even though `accuracy` stays perfect |
| `compositeScore.value` (lesson's actual weights `{1, 0, 0}`) | `round(clamp(1×0.8 + 0×1 + 0×1, 0, 1) × 100) = 80` |
| `passed` | `true` under the `0.8` placeholder threshold (boundary case — see §9's boundary-testing requirement) |

This is the concrete illustration of why `accuracy` and `noteCorrectness` are reported as two different numbers (§4.1): a learner reading "`accuracy: 1.0`" as "you got every note" is correct, while the lower `compositeScore` correctly reflects that an unrequested note was also played.

**Example 2 — a wrong note substituted for an expected one:** the learner plays four of the five expected notes correctly, but at the point where `E4` was expected, plays `F#4` instead, then continues on to the remaining expected notes.

**Input:** Expected `[C4, D4, E4, F4, G4]`. Played `[C4, D4, F#4(66), F4, G4]`.

Trace, per `docs/MUSIC/SCALE_SYSTEM.md` §4.2's two-pointer algorithm: C(60) matches `played[0]`, pointer→1. D(62) matches `played[1]`, pointer→2. E(64): search from pointer 2 — `played[2]=66`≠64, `played[3]=65`≠64, `played[4]=67`≠64 — not found, `missedNotes ← 'E'`, pointer stays at 2 (§4.2's "pointer does not advance on a miss" rule). F(65): search from pointer 2 — `played[2]=66`≠65, `played[3]=65` matches, pointer→4. G(67): `played[4]=67` matches, pointer→5. `played[2]=66` (F#4) is never consumed → `extraNotes ← 'F#4'`.

| Field | Value |
|---|---|
| `correctNotes` | `['C', 'D', 'F', 'G']` |
| `missedNotes` | `['E']` |
| `extraNotes` | `['F#4']` |
| `accuracy` | `0.8` (`4/5`) — **numerically identical to Example 0**, even though this is a substitution, not a clean omission |
| `noteCorrectness` | `0.6` (`max(0, (4−1)/5) = 3/5`) — **diverges from Example 0's `0.8`** even though `accuracy` didn't, because a substitution carries both a miss *and* an extra, and only `noteCorrectness` penalizes the extra |

**This example is the answer to "is there a separate `wrongNotes` field?" — no.** `ExerciseVerdict` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5) has no `wrongNotes` field, and none is needed: a "wrong note" — one note substituted for another — is not a third category, it is simply the co-occurrence of exactly one entry in `missedNotes` (the expected note that didn't arrive) and exactly one entry in `extraNotes` (the note that arrived instead) within the same verdict. Comparing this example to Example 0 side by side is the clearest way to see why reporting only `accuracy` would hide real information: both score `0.8` on `accuracy`, but only one of them (this one) also lowers `noteCorrectness`, because only this one has an extra note to penalize. A UI or a Gemini prompt that wants to say "you played a wrong note" rather than "you skipped a note" derives that framing by checking whether `missedNotes` and `extraNotes` are both non-empty for the same attempt — not from a field that doesn't exist.

**Example 3 — notes played out of order:** the learner plays all five correct pitches, but not in ascending order.

**Input:** Expected `[C4, D4, E4, F4, G4]`. Played `[E4, C4, G4, D4, F4]` (right notes, scrambled order).

Trace: expected C(60) — search from pointer 0: `played[0]=64`(no), `played[1]=60` matches, pointer→2. Expected D(62) — search from pointer 2: `played[2]=67`(no), `played[3]=62` matches, pointer→4. Expected E(64) — search from pointer 4: `played[4]=65`(no) — not found, `missedNotes ← 'E'`, pointer stays 4. Expected F(65) — search from pointer 4: `played[4]=65` matches, pointer→5. Expected G(67) — search from pointer 5: no more played notes in range (`playedNotes.length` is 5, so a search starting at index 5 is empty) — not found, `missedNotes ← 'G'`. Two indices were never consumed: `played[0]=64` (E4) and `played[2]=67` (G4) — the latter specifically because F's match at index 4 advanced the pointer past index 2 before G was ever searched for, orphaning the G4 that was genuinely played, just too early in the sequence to be recognized as "the expected G." In played order, `extraNotes ← ['E4', 'G4']`.

| Field | Value |
|---|---|
| `correctNotes` | `['C', 'D', 'F']` |
| `missedNotes` | `['E', 'G']` |
| `extraNotes` | `['E4', 'G4']` |
| `accuracy` | `0.6` (`3/5`) |
| `noteCorrectness` | `0.2` (`max(0, (3−2)/5) = 1/5`) |
| `completeness` (§4.3, unordered coverage — independent of the above) | `1.0` — all five pitch classes appear somewhere in the played set, regardless of order |

This is the ordered-vs-unordered contrast §4.3 describes in the abstract, made concrete with the same base sequence as every other example here: the **ordered** correctness numbers are the most heavily penalized of all five examples — two `missedNotes` entries *and* two `extraNotes` entries, even though every pitch was eventually played — exactly as `docs/MUSIC/SCALE_SYSTEM.md` §4.2's two-pointer algorithm is designed to score it (order genuinely matters for a `'melody'`-type lesson, and a note played "too early" relative to its expected position is scored as if it were a different, unrequested note, not given credit for showing up eventually) — while **`completeness`** reports full marks, correctly isolating "did you know all five notes" from "did you play them in the right sequence" as two different signals. This is also a useful counter-example to Example 2: not every case with two `missedNotes` also produces exactly two `extraNotes` by coincidence — here it does because reordering, unlike a clean substitution, tends to orphan more than one note at once.

**Example 4 — an octave-mismatch case:** the learner plays the right pitch class for every note, but one note an octave away from where it was expected.

**Input:** Expected `[C4, D4, E4, F4, G4]`. Played `[C4, D4, E5(76), F4, G4]` — `E5`, not `E4`, at the position where E was expected.

Trace: C(60), D(62) match `played[0]`, `played[1]` in order, pointer→2. E(64): search from pointer 2 — `played[2]=76`≠64, `played[3]=65`≠64, `played[4]=67`≠64 — not found, `missedNotes ← 'E'`, pointer stays 2. F(65): `played[2]=76`≠65, `played[3]=65` matches, pointer→4. G(67): `played[4]=67` matches, pointer→5. `played[2]=76` (E5) unconsumed → `extraNotes ← 'E5'` (octave-qualified, since there is no octave ambiguity to resolve here — it is simply not the pitch class the note-name convention would abbreviate).

| Field | Value |
|---|---|
| `correctNotes` | `['C', 'D', 'F', 'G']` |
| `missedNotes` | `['E']` |
| `extraNotes` | `['E5']` |
| `accuracy` | `0.8` (`4/5`) |
| `noteCorrectness` | `0.6` (`max(0, (4−1)/5) = 3/5`) |

**Why this is structurally identical to Example 2's numbers, and deliberately so:** melody/scale matching (this document, `docs/MUSIC/SCALE_SYSTEM.md`) compares on the full `midiNote`, not the pitch class — `E5` (76) is simply not equal to the expected `E4` (64), full stop, regardless of the fact that both are "an E." The engine does **not** give octave-adjacent partial credit for melody/scale exercises. This is the direct, load-bearing contrast with `docs/MUSIC/CHORD_SYSTEM.md`'s chord detection, which is deliberately **octave-invariant by design** (`docs/MUSIC/CHORD_SYSTEM.md` §4.2, §6.1: a C major triad played in any octave is still "correct"). The two behaviors are not inconsistent with each other — they are different tools for different exercise types, exactly as `docs/PRODUCT_SPEC.md` §13 frames chord detection as octave-tolerant specifically, not as a general property of the engine. An implementer should not generalize octave-tolerance from the chord case to the melody/scale case, or vice versa.

## 5. Data structures

Consumes `NoteEvent[]`, `ExpectedNote[]` (via `docs/MUSIC/SCALE_SYSTEM.md`/`docs/MUSIC/CHORD_SYSTEM.md`'s matchers), and `ScoringWeights`; produces every numeric/bucket field of `ExerciseVerdict` and `Score` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5, §4.14) — reproduced here for direct reference, not redefined:

```typescript
interface NoteTimingResult { midiNote: number; deviationMs: number; bucket: 'onTime' | 'slightlyOff' | 'late' | 'early'; }
interface Score { value: number; breakdown: { correctness: number; timing: number; consistency: number; }; }
interface ScoringWeights { correctness: number; timing: number; consistency: number; }
```

## 6. Interfaces / contracts

**Proposed internal function signatures** (`packages/music-engine`'s `scoring/` module):

```typescript
function scoreCorrectness(correctNotes: string[], extraNotes: string[], expectedLength: number): number;
function bucketDeviation(deviationMs: number): 'onTime' | 'slightlyOff' | 'late' | 'early';
function scoreTiming(perNoteTiming: NoteTimingResult[]): number;
function scoreCompleteness(expectedSequence: ExpectedNote[], playedNotes: NoteEvent[]): number;
function scoreConsistencyWithinAttempt(perNoteTiming: NoteTimingResult[]): number;
function scoreConsistencyAcrossAttempts(recentAttemptDeviations: number[][]): number;  // see §4.4's open question
function computeCompositeScore(components: { correctness: number; timing: number; consistency: number }, weights: ScoringWeights): Score;
```

None of these individually are the engine's single external public contract — `evaluate(noteEvents, definition): ExerciseVerdict` (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §6) composes all of them (plus `scoreConsistencyAcrossAttempts`, called separately per §4.4) into the final verdict.

## 7. Dependencies

`docs/MUSIC/NOTE_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/CHORD_SYSTEM.md` for the correctness inputs this document scores; `packages/shared-types` for all referenced contracts. No network, no framework.

## 8. Error handling

- A `ScoringWeights` that fails the sum-to-`1.0` check should never reach the engine at runtime at all (caught at seed time, `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`) — but as defense in depth, the engine should not silently normalize a bad weights object; if one somehow reaches `computeCompositeScore`, it should log the anomaly rather than silently producing a skewed score (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 1's "both mechanisms must hold" principle, applied to content integrity rather than security).
- Division-by-zero guards: `noteCorrectness`/`accuracy`/`completeness` all divide by `expectedSequence.length` (or its unique-count variant) — a `Lesson` with a genuinely empty `expectedSequence` (orientation-type lessons, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2) must never reach these formulas at all; see `docs/CONTENT/LESSON_ARCHITECTURE.md` §Architecture for the explicit engine-level special case that routes empty-sequence lessons around this entire scoring path.

## 9. Testing requirements

Restated and detailed from `docs/TESTING/UNIT_TESTING.md` §6.4–§6.5:
- **All five of §4.9's worked examples, verbatim, each as its own fixture** — Example 0 (the mandatory canonical case) is non-negotiable per `docs/PRODUCT_SPEC.md` §29; Examples 1–4 (extra notes, substitution, out-of-order, octave mismatch) are the required minimum coverage for the correctness/`noteCorrectness` divergence this document depends on throughout. Example 2 vs. Example 4 specifically (identical numeric results, different root causes) is worth a paired assertion so a future refactor can't accidentally collapse the two into a single case.
- Each component (correctness, timing, completeness, consistency) tested in isolation against known inputs before being tested in composition.
- The correctness-vs-accuracy divergence example (§4.1) as an explicit fixture — a case with extra notes present, asserting `accuracy` is unaffected while `noteCorrectness`/`compositeScore` are lower than an otherwise-identical extras-free fixture.
- One fixture landing in each of the four timing buckets (§4.2's worked-example table), asserting `perNoteTiming` reports every note's individual bucket.
- The completeness-vs-accuracy divergence example (§4.3) as an explicit fixture.
- The composite formula tested generically against **injected** weights (not the `0.6/0.25/0.15` default hardcoded into the test), plus a note-ID-style weighting (`correctness: 1.0, timing: 0, consistency: 0`) confirming timing is fully ignorable when a lesson's metadata says so.
- A `ScoringWeights` fixture that fails the sum-to-1.0 check, asserting it is caught at validation, not silently accepted.
- `passed` boundary cases (`compositeScore.value / 100` exactly equal to `passThreshold`, one point above, one point below) using an explicit, clearly-labeled placeholder `passThreshold` (e.g. `0.8`, matching the illustrative value already used in `docs/TESTING/UNIT_TESTING.md` §6.5, kept consistent across this documentation set) — not a claimed product default.

## 10. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29:
- [ ] Unit test coverage on chord/scale/scoring logic is ≥90%.
- [ ] The canonical worked example's `accuracy: 0.8` is reproduced exactly (owned jointly with `docs/MUSIC/SCALE_SYSTEM.md`; §4.9 Example 0).
- [ ] §4.9's four additional worked examples (extra notes, substitution, out-of-order, octave mismatch) each reproduce their documented `ExerciseVerdict` fields exactly.
- [ ] A tutor response never states a note/chord/theory fact that contradicts the structured `ExerciseVerdict` data it was given (§4.8's boundary, tested at the AI layer — `docs/ARCHITECTURE/AI_ARCHITECTURE.md` §11).

## 11. Out of scope

- Note/chord/scale correctness matching itself → `docs/MUSIC/NOTE_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/CHORD_SYSTEM.md`
- How `ScoringWeights` are chosen per lesson type → `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`
- `ScoringWeights` seed-time validation mechanics → `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`
- Acoustic/microphone pitch detection → explicitly out of scope entirely, Phase 3 (§4.7)
- The Gemini prompt template that turns `ExerciseVerdict` into prose → `docs/AI/AI_PROMPTS.md`

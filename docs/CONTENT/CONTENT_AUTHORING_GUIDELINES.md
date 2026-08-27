# Content Authoring Guidelines

**Status:** Draft v1.0 — content documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §5, §9, §13, §24; `docs/PRODUCT/MVP_SCOPE.md`
**Companion documents:** `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` (the exact file format this document explains how to fill in), `docs/CONTENT/BEGINNER_CURRICULUM.md` (the 14 lessons already authored — the worked reference for everything below), `docs/CONTENT/LESSON_ARCHITECTURE.md` (the runtime behavior authored content drives), `docs/MUSIC/SCORING_SYSTEM.md` (the formulas `scoringWeights` parameterizes), `docs/AI/AI_PROMPTS.md` (the **separate**, Gemini-generated text this document is explicitly not about — see §4.6), `docs/UX/ACCESSIBILITY.md`

---

## 1. Purpose

Give whoever writes a new (or edits an existing) lesson content file — a human content author, or an agent doing the same job — concrete, checkable guidance: how to follow `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`'s shape correctly, what a submission must pass before it can be seeded, and how the *prose* fields (`instructions`, `hints`) should read. This document governs **static, human-authored content that ships identically to every learner** — it has nothing to do with, and must not be confused with, the dynamically Gemini-generated text specified in `docs/AI/AI_PROMPTS.md` (§4.6 draws this line explicitly, since it is the single most important distinction in this document).

## 2. Scope

In scope: the step-by-step workflow for adding a lesson, a pre-submission validation checklist, guidance for choosing `scoringWeights`, tone/voice/style guidance for `instructions` and `hints` text, the static-vs-AI-generated content boundary, curriculum scope guardrails, and accessibility considerations for authored text.

Out of scope: the JSON schema itself (`docs/CONTENT/LESSON_CONTENT_SCHEMA.md`), the actual MVP lesson list (`docs/CONTENT/BEGINNER_CURRICULUM.md`), runtime unlocking/scoring/hint-escalation mechanics (`docs/CONTENT/LESSON_ARCHITECTURE.md`, `docs/MUSIC/SCORING_SYSTEM.md`), Gemini prompt template text and tutor-response tone (`docs/AI/AI_PROMPTS.md`, `docs/AI/AI_PROMPT_ARCHITECTURE.md`).

## 3. Requirements

- **A wrong key press is a normal part of learning, never a failure state** (`docs/PRODUCT_SPEC.md` §5; PR-LESSON-04) — every piece of authored text this document governs must read consistently with that framing; no punitive language anywhere in `instructions` or `hints`.
- **Hints escalate, and never produce a hard stop** (`docs/PRODUCT_SPEC.md` §9; PR-LESSON-05) — every lesson's `hints` array must actually help a stuck learner progress, not just repeat the instructions in different words.
- **No content beyond `docs/PRODUCT/MVP_SCOPE.md`'s boundary** — this document is bound by the same MVP scope every other document in this batch is; an author (human or agent) does not get to add a 15th "bonus" lesson, a minor-chord lesson, or a Phase-2 scale key because it seems easy to author. `docs/PRODUCT/MVP_SCOPE.md` §1 is explicit: "If another document appears to disagree with this one, this document wins."
- **Static content is never phrased as if responding to a specific learner's specific attempt.** This is the structural difference from Gemini-generated feedback (§4.6) — authored `instructions`/`hints` text is written once and read by every learner who reaches that lesson, so it can never say anything like "you played a C# instead of a C" (a specific, attempt-dependent fact only the deterministic engine and Gemini's *Exercise Feedback* template, `docs/AI/AI_PROMPTS.md` §2, ever have access to).
- **Every submission passes `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`'s seed-time validation before merge** — this document's checklist (§4.2) is a pre-flight version of that same validation, meant to be run by the author before a PR, not a substitute for it.

## 4. Architecture

### 4.1 Workflow for adding or editing a lesson

1. Confirm the topic is actually in MVP scope — check `docs/PRODUCT/MVP_SCOPE.md` §2 and `docs/CONTENT/BEGINNER_CURRICULUM.md` §4.1's existing 14 lessons first. If it isn't there and isn't a small in-place edit to one of the 14, it almost certainly belongs in `docs/PRODUCT/FUTURE_ROADMAP.md` instead, not a new content file.
2. Pick the next `level` and `prerequisiteLessonId` consistent with the existing linear chain (`docs/CONTENT/LESSON_ARCHITECTURE.md` §4.1) — MVP has no branching, so this is always "insert after an existing lesson, renumbering `level` downstream if needed," never "attach to a new parent."
3. Write the file against `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`'s full field list (§4.1 of that document) — every field, including `hints` (§4.5 below).
4. Choose `scoringWeights` using §4.3 below, not by copying an arbitrary existing lesson's values without checking whether this lesson's own timing/correctness balance actually matches.
5. Run the pre-submission checklist (§4.2).
6. Run the actual seed-time validation locally (`scripts/seedLessons.ts` in dry-run/validate mode, `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` §Interfaces) before opening a PR — a content PR that fails CI validation on a rule already checkable locally is an avoidable review cycle.

### 4.2 Pre-submission checklist

A condensed, author-facing version of `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` §4.4's full validation table — every item here maps to a rule enforced mechanically at seed time, so this checklist catches nothing the seed script wouldn't also catch, but catches it faster, before a PR:

- [ ] `id` matches the filename, and isn't already used by another lesson.
- [ ] `level` is unique and fits the existing sequence with no gap.
- [ ] `prerequisiteLessonId` points at a real, already-existing lesson (or is `null`, for the one root lesson only).
- [ ] `expectedSequence` is non-empty unless `type: 'orientation'`.
- [ ] Either every note in `expectedSequence` has `expectedTimeMs`, or none do — never a mix.
- [ ] `tempoBpm` is set whenever any `expectedTimeMs` is present.
- [ ] `scoringWeights.correctness + timing + consistency` sums to `1.0` (§4.3 below has the actual guidance for what values to pick).
- [ ] `passThreshold` is set to `0.8` (`docs/CONTENT/BEGINNER_CURRICULUM.md` §5's recommendation) unless a project decision has formally ratified a different value.
- [ ] `hints` has at least one entry, ideally three (§4.5).
- [ ] `instructions` and every `hints` entry read consistently with §4.4's tone guidance — read them aloud once; if any sentence sounds like it's scolding the learner, rewrite it.

### 4.3 How to choose `ScoringWeights`

Start from `docs/PRODUCT_SPEC.md` §13's stated default, `{correctness: 0.6, timing: 0.25, consistency: 0.15}` — this is the right starting point for most content, not a fixed constant (`docs/MUSIC/SCORING_SYSTEM.md` §4.5 is explicit that weights are per-lesson metadata). Adjust from there based on what the lesson is actually testing:

| Situation | Guidance | Worked precedent |
|---|---|---|
| The lesson has **no timed notes at all** (every `ExpectedNote` omits `expectedTimeMs`) | `{correctness: 1, timing: 0, consistency: 0}` — **required**, not optional, per `docs/MUSIC/SCORING_SYSTEM.md` §4.2: a nonzero timing/consistency weight on an untimed lesson would multiply a meaningless default score into a visible one. | `docs/CONTENT/BEGINNER_CURRICULUM.md`'s L1–L8. |
| The lesson is specifically about rhythm/timing | Shift weight toward `timing` — `docs/PRODUCT_SPEC.md` §13's own example: "a rhythm exercise shifts weight toward timing." | `l09-basic-rhythm`: `{0.4, 0.5, 0.1}` (`docs/CONTENT/LESSON_CONTENT_SCHEMA.md` §4.3). |
| The lesson is a `'scale'` | The `docs/PRODUCT_SPEC.md` §13 default is this case's paradigm — use it as-is unless there's a specific reason not to. | `l10`–`l13`: `{0.6, 0.25, 0.15}`. |
| The lesson is `type: 'chord'` | `{correctness: 1, timing: 0, consistency: 0}` — chord onset simultaneity is judged by the clustering window (`docs/MUSIC/CHORD_SYSTEM.md` §4.1), not per-note `expectedTimeMs` deviation, so a chord lesson's notes should have no `expectedTimeMs` and the same untimed rule above applies. | `l14-intro-to-major-triads`. |
| The lesson is a pure note-recall/`'noteId'` drill | `{correctness: 1, timing: 0, consistency: 0}` — recall speed is the *Note Identification game's* concern (`Exercise`, not `Lesson` — a different, static-config-driven system, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.3), not a lesson's. | `l03`–`l05`. |

Whatever combination is chosen, it must sum to exactly `1.0` (§4.2's checklist) — if the three values you'd naturally pick don't sum cleanly, adjust `correctness` last, since it should always be the largest component for a beginner-facing lesson (`docs/PRODUCT_SPEC.md` §5's "guide, don't punish" framing prioritizes "did you get the notes right" over "were you perfectly on time," for every MVP lesson type without exception).

### 4.4 Tone and voice for `instructions`

**The governing principle, restated from `docs/PRODUCT_SPEC.md` §5 because every other rule in this section is downstream of it:** a wrong key press is a normal part of learning, not a failure state. Authored text is the learner's first and most frequent point of contact with the app's voice — it sets the tone before any AI Tutor conversation or Gemini-generated feedback ever does.

| Do | Don't |
|---|---|
| "This time, play them in steady time, one every half second." | "You need to play these in time or you won't pass." |
| "Notice the groups of two and three black keys — that's your landmark." | "Memorize the black key pattern." (correct information, cold delivery) |
| "Watch the demonstration first, then try to match it." | "If you get this wrong, review the demonstration again." (leads with the failure case) |
| Second person, present tense, short sentences. | Passive voice, jargon introduced without context (e.g. "root position" used before it's been explained once in plain language). |
| One clear instruction per lesson, stated once, plainly. | Multiple instructions bundled into one dense paragraph a beginner has to parse before they can start. |

**Concretely:** every `instructions` string should answer, in the first sentence, "what am I about to do" — not "here is background theory." `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` §4.3's worked example ("You already know these five notes — C, D, E, D, C. This time, play them in steady time...") is the reference to match: it opens by connecting to what the learner already knows, states the new thing plainly, and gives one concrete action.

### 4.5 Writing `hints`

Three tiers, per `docs/CONTENT/LESSON_ARCHITECTURE.md` §4.4's proposed escalation, ordered general → specific → near-explicit:

1. **Tier 1 (gentle, general):** point at a strategy or a place to look, not the answer — "Try tapping your foot to a steady beat before you start playing."
2. **Tier 2 (more specific):** narrow it down concretely, without fully giving the answer away — "Focus on hitting C, D, and E right on the beat rather than rushing ahead."
3. **Tier 3 (near-explicit):** state exactly what to do — "Play each note exactly on the half-second mark: C, then D, then E, then D, then C."

The array's last element repeats for every attempt beyond tier 3 (`docs/CONTENT/LESSON_ARCHITECTURE.md` §4.4) — so tier 3 should be written as something a learner can act on indefinitely, not a one-shot line that reads oddly on repetition ("try again!" repeated five times reads very differently from a concrete instruction repeated five times — write tier 3 as the latter).

Every hint must be genuinely actionable and specific to *this* lesson's actual content — a hint generic enough to paste unchanged into any lesson ("keep practicing!") provides no real escalation and fails the spirit of PR-LESSON-05 even if it technically satisfies "the array has three entries."

### 4.6 The boundary with AI-generated content — read this before writing anything

**This document governs static content only.** Every string this document is about (`instructions`, `hints`) is written once, by a human or content-authoring agent, committed to the repo, seeded into Firestore, and shown **identically to every learner** who reaches that lesson (`docs/PRODUCT_SPEC.md` §17: `lessons` is public-read content, not per-user data). It is never regenerated, never personalized, and never has access to what a specific learner just did.

**`docs/AI/AI_PROMPTS.md` governs a completely different kind of text**: Gemini-generated, per-attempt, per-learner responses — Exercise Feedback ("you missed F between E and G," referencing *this specific attempt's* `ExerciseVerdict`), Tutor Q&A, Theory Explanations, motivational nudges. That text is never committed to a file, is generated fresh (or served from a cache keyed on a specific `(concept, skillLevel)` pair, `docs/PRODUCT_SPEC.md` §15) at request time, and is explicitly forbidden from inventing a fact not present in the structured data it was given (`docs/AI/AI_PROMPTS.md` §1).

**The two must never be confused, in either direction:**
- Do not write a lesson's `instructions` or `hints` as if they were reacting to a specific learner's specific mistake — that is Gemini's job, downstream, and static content has no way to know what any given learner actually played.
- Do not write Gemini prompt templates that duplicate static `instructions` text verbatim as filler — `docs/AI/AI_PROMPTS.md` §1 already establishes that structured data is trimmed before rendering, not dumped wholesale; static lesson prose is a separate concern from what gets fed into a prompt.
- If a hint feels like it should reference what the learner specifically just played, that is a signal the feedback belongs in the Exercise Feedback flow (`docs/AI/AI_PROMPTS.md` §2), which already runs after every scored attempt — not a reason to try to parameterize a static `hints` string.

### 4.7 Curriculum scope guardrails

Restated because it is the easiest mistake for either a human author or an authoring agent to make under time pressure: **do not add a chord type, scale key, lesson topic, or exercise type beyond what `docs/PRODUCT/MVP_SCOPE.md` §2 and `docs/CONTENT/BEGINNER_CURRICULUM.md` §4.1 already establish**, even if it seems like a natural, small extension of an existing lesson. Concretely, for MVP: no minor/7th/diminished/augmented chords or inversions (even though the *engine* can detect minor triads — `docs/MUSIC/CHORD_SYSTEM.md` §3 — lesson *content* stays major-triad-only), no scale key beyond C/G/D/A major, no song content, no second/branching path. A genuinely good idea that falls outside this boundary belongs in `docs/PRODUCT/FUTURE_ROADMAP.md` as a Phase 2+ note, not in a merged content file.

### 4.8 Accessibility in authored text

`instructions` and score/hint text are announced via ARIA live regions to screen-reader users (`docs/PRODUCT_SPEC.md` §24; `docs/UX/SCREEN_SPECIFICATIONS.md` §8's Lesson Player accessibility notes) — write with that audience in mind, not only the sighted, piano-watching one:

- Avoid instructions that are only meaningful visually with no textual equivalent (e.g., "play the highlighted key" is fine only because the UI Layer also gives that key an accessible name, `docs/PRODUCT_SPEC.md` §24: "C4, white key" — never author text that depends on a color alone, like "play the green key," since correctness is never color-only anywhere in this product, PR-A11Y and `docs/PRODUCT_SPEC.md` §24).
- Prefer describing *what* to do over *where on screen* something is, since "where on screen" has no meaning read aloud.

## 5. Data structures

This document defines no new data shapes — every field it gives guidance for is already specified in `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` §4.1.

## 6. Interfaces / contracts

None of its own — this document is guidance for humans/agents authoring files against `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`'s schema, not a code-facing contract.

## 7. Dependencies

`docs/CONTENT/LESSON_CONTENT_SCHEMA.md` (the schema this guidance is written against), `docs/CONTENT/BEGINNER_CURRICULUM.md` (the worked reference content), `docs/MUSIC/SCORING_SYSTEM.md` (§4.3's `ScoringWeights` guidance), `docs/PRODUCT/MVP_SCOPE.md` (§4.7's scope boundary), `docs/AI/AI_PROMPTS.md` (§4.6's boundary).

## 8. Error handling

Not applicable in the runtime sense. The relevant failure mode is a human/agent process one: a content PR that violates §4.2's checklist should fail CI (`docs/CONTENT/LESSON_CONTENT_SCHEMA.md` §4.4's seed-time validation) before it can merge, and a PR whose tone violates §4.4/§4.6 is a review-time human judgment call, not a mechanically checkable rule — reviewers should read new `instructions`/`hints` text aloud as part of review, per §4.2's checklist item.

## 9. Testing requirements

- Every new/edited lesson file passes `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`'s full seed-time validation (mechanical, CI-enforced).
- No mechanical test can verify tone (§4.4) or the static/AI-generated boundary (§4.6) — these are reviewed by a human at PR time; `docs/CONTENT/BEGINNER_CURRICULUM.md`'s 14 lessons are the worked reference a reviewer checks new content against for consistency of voice.

## 10. Acceptance criteria

- [ ] Every lesson content file in `content/lessons/` passes `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`'s validation.
- [ ] No `instructions` or `hints` string in any MVP lesson uses punitive framing (PR-LESSON-04) — spot-checked against §4.4's Do/Don't table.
- [ ] No lesson content file introduces a topic outside `docs/PRODUCT/MVP_SCOPE.md` §2.
- [ ] No `instructions`/`hints` string references a specific learner's specific attempt (§4.6) — that content only ever comes from a Gemini-generated response, never from a static file.

## 11. Out of scope

- The JSON schema itself → `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`
- The actual 14 MVP lessons → `docs/CONTENT/BEGINNER_CURRICULUM.md`
- Runtime unlocking/hint-escalation/completion mechanics → `docs/CONTENT/LESSON_ARCHITECTURE.md`
- Gemini prompt template text, tutor tone, AI guardrails → `docs/AI/AI_PROMPTS.md`, `docs/AI/AI_GUARDRAILS.md`
- Scoring formulas themselves (only how to choose their per-lesson weights) → `docs/MUSIC/SCORING_SYSTEM.md`

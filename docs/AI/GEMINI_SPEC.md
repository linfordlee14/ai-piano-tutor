# Gemini Feature Specification

**Status:** Draft v1.0 — specification only, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §6, §7, §14, §15, §19, §31; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5, §4.8–4.10
**Companion documents:** `docs/ARCHITECTURE/AI_ARCHITECTURE.md` (system-level call pattern, entry point), `docs/ARCHITECTURE/DATA_FLOW.md` (field-by-field request traces for Flows 1 & 2), `docs/AI/AI_PROMPT_ARCHITECTURE.md` (the client→Function→Gemini call architecture in depth), `docs/AI/AI_PROMPTS.md` (the actual prompt template text), `docs/AI/AI_CONTEXT_MODEL.md` (AI memory), `docs/AI/AI_GUARDRAILS.md`, `docs/AI/AI_ERROR_HANDLING.md`, `docs/AI/AI_COST_CONTROL.md`

This is the master feature spec for every place Gemini touches AI Piano Tutor. It defines, for each of five features, exactly when Gemini is called, exactly when it is not, what goes in, what comes back, and what happens when it fails. The other `docs/AI/` documents each zoom into one cross-cutting concern (prompt text, memory, guardrails, errors, cost); this document is the one place all five features are described end to end, side by side.

---

## 1. Purpose

Enumerate every Gemini-backed feature in the product, and for each one, pin down the contract precisely enough that `functions/src/ai/` can be implemented without further product decisions. Five features:

1. Exercise Feedback
2. Tutor Q&A
3. Theory Explanation
4. Motivation / Nudge
5. Daily Practice Plan — **Phase 2 / Future, not MVP**

## 2. Scope

In scope: purpose, trigger, non-trigger, input schema, assembled context, output shape and cap, validation approach, fallback behavior, rate limits, and cost tier for each of the five features above.

Out of scope: the prompt template text itself (`AI_PROMPTS.md`), the mechanics of how a Cloud Function assembles and calls Gemini (`AI_PROMPT_ARCHITECTURE.md`), the internal shape and lifecycle of `AIMemorySummary` (`AI_CONTEXT_MODEL.md`), output validation/injection defense mechanics (`AI_GUARDRAILS.md`), failure-mode-to-fallback mapping (`AI_ERROR_HANDLING.md`), and the cost/caching/rate-limit mechanics behind the numbers used here (`AI_COST_CONTROL.md`).

## 3. Requirements — the non-negotiable boundary

Restated from `docs/PRODUCT_SPEC.md` §14 because every requirement below depends on it: **Gemini is never asked to determine whether a note, chord, or scale was played correctly, calculate accuracy, calculate timing, calculate a score, determine lesson completion, determine XP, determine streaks, or validate user-submitted performance data.** Those are the exclusive responsibility of `packages/music-engine`, invoked server-side. Gemini receives an already-computed `ExerciseVerdict` (or an aggregate derived from a sequence of them, `AIMemorySummary`) as read-only input and explains it in natural language. It does not recompute, sanity-check, second-guess, or contradict any field it was given.

This is not a style preference — it's the reason the AI layer is trustworthy enough to ship on something presenting itself as a piano *teacher*. Every one of the five feature specs below inherits this rule without exception, including the two (Motivation/Nudge, Theory Explanation) that never touch performance data directly — they inherit it because their *system instructions* still forbid inventing any note/chord/theory fact not present in their (much smaller) structured input. See `AI_GUARDRAILS.md` for this rule restated as an enforced guardrail, not just a design intention.

## 4. Architecture — call pattern in brief

Full depth: `AI_PROMPT_ARCHITECTURE.md`. In one line: **client → Cloud Function (callable) → Function assembles prompt from a template + server-loaded, server-trusted context → Function calls Gemini → Function validates the response → Function returns a typed result to the client.** The client never calls Gemini directly, never assembles a prompt, and never supplies its own copy of `recentExerciseContext` or `learnerMemorySummary` — those are read from Firestore by the Function, from documents the client cannot write (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9). There is exactly one arrow into the `Gemini` node in the system diagram (`docs/PRODUCT_SPEC.md` §33; `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` §4.3), and it originates from Cloud Functions.

**Callable-function-to-template mapping (MVP).** `docs/PRODUCT_SPEC.md` §20 and `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` §6 name exactly two MVP callables that touch Gemini — `getExerciseFeedback` and `getTutorResponse` — even though §15 defines five prompt templates. The mapping below is this document's resolution of that gap (also flagged in §14 below and in the final report):

| Callable function | Prompt template(s) it may invoke | Selection logic |
|---|---|---|
| `getExerciseFeedback` | Exercise Feedback (always); Motivation/Nudge (conditionally) | Exercise Feedback runs on every call, in the same invocation that scores the attempt. Motivation/Nudge runs only when a fresh milestone is detected server-side **and** it isn't covered by the pre-written pool (§7.4 below). |
| `getTutorResponse` | Tutor Q&A **or** Theory Explanation | Deterministic, server-side text match of `learnerQuestion` against a small curated concept-phrase table (`AI_COST_CONTROL.md`). A match routes to Theory Explanation (cache-checked first); no match routes to Tutor Q&A. The client-facing input shape is `{ learnerQuestion: string }` either way — quick-suggestion chips populate the same field with pre-written text designed to match the table, per `docs/ARCHITECTURE/DATA_FLOW.md` §5.1. |
| *(Phase 2)* `generateDailyPlan` | Daily Practice Plan | Not built for MVP. |

## 5. Feature 5.1 — Exercise Feedback

**Purpose:** turn an already-computed, already-persisted `ExerciseVerdict` into 2–3 sentences of natural-language teaching feedback — what was right, what was missed, one next step.

**Trigger — exact:** automatically, server-side, as the final step inside `getExerciseFeedback`, immediately after the deterministic verdict has been computed by `packages/music-engine` and written to Firestore (`PracticeSession`, `ProgressSummary`). One call per completed exercise/lesson-step attempt. This is the "end of a scored exercise" trigger named in `docs/PRODUCT_SPEC.md` §14.

**Explicitly NOT called:**
- Never per `NoteEvent` / per keystroke — only once, after the whole attempt is scored.
- Never against a client-computed, provisional verdict — only the server-authoritative one (`docs/ARCHITECTURE/DATA_FLOW.md` §3).
- Never re-run to "improve" or re-word an already-returned feedback string — one attempt, one feedback call, one result.
- Never called if the deterministic scoring step itself fails (a scoring error is a different, higher-severity failure than "no prose to explain it" — see `AI_ERROR_HANDLING.md`).

**Input schema:** `{exerciseType, expected, played, accuracyScore, missedNotes, attemptNumber, learnerLevel}` — the exact shape given in `docs/PRODUCT_SPEC.md` §15 and used identically in `docs/ARCHITECTURE/DATA_FLOW.md` §4.2 step 8. Assembled server-side, entirely from data the Function already has in hand at this point in the call:

| Field | Source |
|---|---|
| `exerciseType` | `Lesson.type` or `Exercise.kind` (the definition just used to score the attempt) |
| `expected` | Note names derived from the `Lesson.expectedSequence` / `Exercise` definition |
| `played` | Note names derived from the attempt's `noteEvents`, in the order played |
| `accuracyScore` | `ExerciseVerdict.accuracy` (canonical field name is `accuracy`; see §14 open-question note below) |
| `missedNotes` | `ExerciseVerdict.missedNotes` (matches canonical field name exactly) |
| `attemptNumber` | `ExerciseAttempt.attemptNumber` (informational only, not trust-critical for feedback tone) |
| `learnerLevel` | `AIMemorySummary.skillLevel` (canonical enum `'beginner' \| 'returning'`; see §14 open-question note on the field-name mismatch) |

**Context assembled:** nothing beyond the table above — deliberately minimal. Exercise Feedback does not read `AIMemorySummary.weakNotes/weakChords/strengths` or prior `PracticeSession` history; the verdict just computed *is* the context. This keeps the prompt short (cost) and keeps the feature's blast radius small (a bug in memory aggregation can't corrupt exercise feedback).

**Output shape and length cap:** plain text, 2–3 sentences, structured as (a) what was right, named specifically, (b) the specific miss, named by note/chord exactly as given, (c) one concrete next step. Returned to the client as `feedbackText: string`, alongside the `verdict: ExerciseVerdict` that was already going to be returned regardless of whether this call succeeds. Full template text: `AI_PROMPTS.md` §1.

**Validation approach:** non-empty; sentence/length heuristic against the 2–3-sentence cap; a coarse check that every note/chord name mentioned appears somewhere in the input JSON (best-effort — see `AI_GUARDRAILS.md`, this cannot be made airtight by a regex alone, so the human-reviewed eval set in `docs/PRODUCT_SPEC.md` §26 is the real backstop for tone/fact accuracy over time).

**Fallback if Gemini fails/times out/returns invalid output:** the learner still sees the full deterministic `ExerciseVerdict` — correct/missed notes, score, pass/fail — exactly as if the call had succeeded. Only `feedbackText` degrades, to a canned, deterministic line templated directly from the verdict fields (no Gemini involved at all), e.g. built from a small set of sentence fragments keyed on `missedNotes.length` and `accuracy`. The learner is never blocked and never sees a broken screen. Full mapping: `AI_ERROR_HANDLING.md`.

**Rate limits:** counts against its own, separate **Exercise Feedback safety-net cap** (`AI_COST_CONTROL.md` §5) — deliberately not the same counter as Tutor Q&A, and deliberately sized much higher, since volume here is already naturally bounded by how many real, server-validated exercise attempts a learner can complete, and exhausting it must never block scoring (`AI_ERROR_HANDLING.md` §5.1). It still exists, rather than being uncapped, so a scripted-attempt abuse pattern (fabricated `NoteEvent[]` sequences submitted in a tight loop) or a retry-storm bug is caught rather than running unbounded.

**Cost tier:** cheapest available tier (`docs/PRODUCT_SPEC.md` §14/§15/§31) — high volume, low creative demand, structured input.

## 6. Feature 5.2 — Tutor Q&A

**Purpose:** a grounded, conversational answer to a learner's free-text question, calibrated to their skill level and — when relevant — their actual recent performance.

**Trigger — exact:** learner-initiated only, inside the Tutor chat screen (`docs/PRODUCT_SPEC.md` §10): typing and submitting a free-text question, or tapping a quick-suggestion chip (which submits pre-written text through the identical path).

**Explicitly NOT called:**
- Never polled, never pre-fetched, never called to "keep the chat feeling alive."
- Never called while the learner is only viewing prior messages in the current session — chat history is session-local UI state, not re-fetched or re-summarized.
- Never called after the per-user rate-limit check fails — the check happens first and short-circuits to `{ ok: false, error }` before Gemini is invoked (`docs/ARCHITECTURE/DATA_FLOW.md` §5.1 step 5).
- Never called with a client-supplied `recentExerciseContext`, `learnerMemorySummary`, or `skillLevel` — even if a modified client sends them, the callable's signature accepts only `{ learnerQuestion: string }` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9).

**Input schema:** client supplies only `learnerQuestion: string`. The Function assembles the full canonical `TutorRequest` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9):

```typescript
interface TutorRequest {
  learnerQuestion: string;
  recentExerciseContext: ExerciseVerdict | null;
  learnerMemorySummary: AIMemorySummary;
  skillLevel: 'beginner' | 'returning';
}
```

**Context assembled:** `recentExerciseContext` — the most recent `PracticeSession.verdict`, or `null` if the learner has none yet; `learnerMemorySummary` — the current `AIMemorySummary` document; `skillLevel` — `learnerMemorySummary.skillLevel`, duplicated onto the request for prompt-template convenience. Full assembly trace: `docs/ARCHITECTURE/DATA_FLOW.md` §5.

**Output shape and length cap:** `TutorResponse` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.10): `text` ≤150 words unless the learner explicitly asks for more detail; optional `suggestedAction` deep-link hint; `tier: 'mid'`; `cached: false` (the Tutor Q&A branch is never cache-served — grounded answers are per-learner by definition; caching lives on the Theory Explanation branch, §7 below). The response must ground itself in `recentExerciseContext`/`learnerMemorySummary` **when the question is actually about the learner's own playing**, and must not force those fields into an answer to a generic theory question where they aren't relevant. Full template text: `AI_PROMPTS.md` §2.

**Validation approach:** non-empty; word-count cap enforced (truncation is not an acceptable fallback for a conversational answer — a truncated mid-sentence reply reads as broken, so a cap violation is treated as an invalid response and replaced wholesale, not trimmed); matches the plain-text contract (no stray JSON/markup leakage from a structured-output misconfiguration). Full checks: `AI_GUARDRAILS.md`.

**Fallback if Gemini fails/times out/returns invalid output:** the Tutor screen shows a deterministic "temporarily unavailable, here's what to practice instead" card, built from `ProgressSummary`/`AIMemorySummary` data already available client-side — never from Gemini. Full mapping: `AI_ERROR_HANDLING.md`.

**Rate limits:** the primary consumer of the per-user daily live-call cap (`AI_COST_CONTROL.md`) — this is the feature named directly in `docs/PRODUCT_SPEC.md` §19's "a daily cap on tutor calls for free-tier users" and §29's "Per-user rate limiting demonstrably blocks a scripted burst of tutor calls beyond the configured cap" acceptance criterion.

**Cost tier:** mid tier — the one feature in this spec where open-ended response quality is genuinely load-bearing (`docs/PRODUCT_SPEC.md` §14/§15).

## 7. Feature 5.3 — Theory Explanation

**Purpose:** a calibrated, fully general (non-personal) explanation of one music-theory concept, cacheable per `(concept, skillLevel)` pair so repeat questions across the whole user base cost nothing after the first.

**Trigger — exact:** the *same* client action as Tutor Q&A — a learner submits `learnerQuestion` (typed, or via a quick-suggestion chip) — but the Function routes it here instead of Tutor Q&A when the question matches a known concept in a small, curated phrase table maintained in `functions/src/ai/` alongside the prompt templates. This routing mechanism is this document's proposed resolution of a real gap: `docs/PRODUCT_SPEC.md` §15's cost note describes theory questions as cacheable but does not specify how a request is recognized as one at call time, and neither `docs/ARCHITECTURE/DATA_FLOW.md` nor `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` resolves it either (both explicitly defer prompt-level detail to `docs/AI/`). See §14 below.

⚠ **Design constraint driving this choice:** the routing check must not itself be an LLM call (that would defeat the cost purpose of caching in the first place) and must not require a second, structurally distinct client payload (the canonical `TutorRequest`/callable signature is `{ learnerQuestion: string }` only — `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9, confirmed by `docs/ARCHITECTURE/DATA_FLOW.md` §5.1). A cheap, deterministic phrase/keyword match against a small, hand-maintained list of known concepts is the only option consistent with both constraints.

**Explicitly NOT called (live):**
- On a cache hit for the same `(concept, skillLevel)` pair — served straight from cache, zero Gemini call, `cached: true`.
- On a question that doesn't match any known concept — falls through to Tutor Q&A instead (which is still capable of answering generic theory questions correctly from its own model knowledge; nothing is lost except the caching benefit for that specific unrecognized phrasing).
- Never includes any learner-specific data in its input or output — this is precisely what makes it safe to cache and reuse across every learner who asks the same thing at the same level.

**Input schema:** `{concept: string, skillLevel: 'beginner' | 'returning'}` — per `docs/PRODUCT_SPEC.md` §15's table exactly. `concept` is a value from the same curated table used for routing (a small closed vocabulary, e.g. `"black-keys"`, `"major-triad"`, `"time-signature"` — exact list owned by `functions/src/ai/`, not this document).

**Context assembled:** none beyond `concept` and `skillLevel` — no `recentExerciseContext`, no `learnerMemorySummary`. This is the point: the response must be fully general to be cacheable.

**Output shape and length cap:** plain text, 3–5 sentences (this document's proposed cap — `docs/PRODUCT_SPEC.md` §15 specifies "calibrated to level" but no exact sentence count; flagged as a filled gap, non-blocking), calibrated to `skillLevel`. Full template text: `AI_PROMPTS.md` §3.

**Validation approach:** same shape as Theory Explanation's sibling templates — non-empty, length cap, no learner-specific leakage (a coarse check that the response contains no first/second-person performance claims beyond generic "you" address is part of the guardrail, since a leaked personal reference would poison the cache for every future learner who hits it — see `AI_GUARDRAILS.md`).

**Fallback if Gemini fails/times out/returns invalid output:** falls back to the same deterministic canned-message pattern as Tutor Q&A (this branch shares `getTutorResponse`'s error handling, since it's a mode of the same callable) — never written *into* the cache, so a transient failure never permanently poisons a `(concept, skillLevel)` entry.

**Rate limits:** a cache hit does not count against the per-user daily cap (it costs the app nothing). A cache miss (first-ever ask for that pair, or a cache invalidation) does count, same as any other live call.

**Cost tier:** cheapest available tier, and cacheable — `docs/PRODUCT_SPEC.md` §15 calls this "likely the single highest-leverage cost optimization in the whole AI layer." Full caching mechanics: `AI_COST_CONTROL.md`.

## 8. Feature 5.4 — Motivation / Nudge

**Purpose:** one short, warm line of encouragement tied to the learner's streak or a milestone, shown on the dashboard.

**Trigger — exact:** evaluated at most once per calendar day per learner, at the first point after a fresh milestone is detected server-side (a streak-count change, or a lesson/level completion) that either the dashboard loads or an exercise completes — in practice, most naturally piggybacked onto `getExerciseFeedback`'s existing invocation (§4 table above), since exercise completion is already a server round trip and a milestone is most often *caused* by the exercise that just completed. **The large majority of cases never reach Gemini at all:** every common, enumerable streak/milestone value (round numbers — 3, 7, 14, 30, 60, 100-day streaks; standard milestones like a level completion) is served from a small pre-written pool, matched by simple rule against fields already synced to the client (`UserProfile.streak`, `UserProfile.xp`) or already known server-side. A live Gemini call is reserved for the rare case: a milestone that doesn't match anything in the pool.

**Explicitly NOT called:**
- Never on every dashboard load — evaluated once per fresh milestone, not polled.
- Never for a pool-covered case — see above; this is the default path for the large majority of nudges, not an edge case.
- Never twice for the same milestone (idempotent — a milestone produces at most one nudge, pooled or live).
- No dedicated callable function exists for this in MVP. This document proposes folding the rare live-call path into `getExerciseFeedback`'s response (an optional field) rather than adding a new callable purely to generate one sentence — flagged as a gap-filling proposal, see §14.

**Input schema:** `{streak: number, recentMilestone: string | null}` per `docs/PRODUCT_SPEC.md` §15.

**Context assembled:** none beyond the two input fields — deliberately the smallest of all five templates.

**Output shape and length cap:** exactly one short sentence, plain text. Full template text: `AI_PROMPTS.md` §5.

**Validation approach:** non-empty, single-sentence heuristic, no note/chord/theory content, no score/accuracy content (this is pure encouragement — a guardrail check specifically rejects a response that smuggles in a theory claim or a score reference, since neither belongs here).

**Fallback if Gemini fails/times out/returns invalid output:** silently fall back to the nearest pool entry rather than showing an error or omitting the nudge entirely — a missing "nice streak" line is low-stakes enough that the fallback should be invisible, not a degraded-state banner.

**Rate limits:** negligible by design — at most one live call per learner per day, only on the rare pool-miss path.

**Cost tier:** cheapest available tier — and, per `docs/PRODUCT_SPEC.md` §15, the top common cases should not reach a live call at all; the pre-written pool is the primary cost lever for this feature, not tiering.

## 9. Feature 5.5 — Daily Practice Plan (Phase 2 / Future — **not MVP**)

⚠ **Explicitly deferred.** `docs/PRODUCT_SPEC.md` §7's pushback is direct: personalization needs historical performance data to be *accurate*; a brand-new user has none. Calling Gemini to generate a practice plan from zero signal is pure cost with no accuracy benefit over a deterministic rule. MVP ships the rule-based generator only: **next incomplete lesson + review the learner's last 3 misses.** This section documents the Phase 2 shape now so that work doesn't start from zero later — it is not an instruction to build it for MVP.

**Purpose (Phase 2):** a personalized daily practice session — a small set of timed segments — generated from the learner's actual weak areas and recent trend, once there's enough usage data for personalization to outperform the deterministic rule.

**Trigger (Phase 2, proposed — not committed):** on explicit learner action (opening "today's practice" / requesting a refreshed plan) or once per day on first dashboard load.

**Explicitly NOT called — this is the case named directly in this document's brief:**
- **Never for a brand-new user with zero `PracticeSession` history.** The deterministic rule-based generator handles this case from day one, in MVP and beyond.
- ⚠ Proposed, not yet ratified: even after Phase 2 ships, a learner stays on the deterministic generator until they cross a minimum data threshold (e.g., **3 completed sessions**, PROPOSED) — personalization needs signal to beat the deterministic baseline, not merely be present. This threshold is not specified anywhere in `docs/PRODUCT_SPEC.md` and needs product sign-off before Phase 2 implementation.

**Input schema (per `docs/PRODUCT_SPEC.md` §15):** `{weakAreas, recentAccuracyTrend, availableMinutes, streak}`.

⚠ **Open note (non-blocking, Phase 2 scope):** `weakAreas` and `recentAccuracyTrend` are reasonably derivable from `AIMemorySummary.weakNotes`/`.weakChords` and recent `PracticeSession.verdict.compositeScore` values respectively, but neither is a canonical field with that exact name today. `availableMinutes` has no source at all in the current data model or any screen in `docs/PRODUCT_SPEC.md` §10 — no UI currently asks a learner how much time they have today. None of this blocks MVP (the feature isn't built), but it should be resolved before Phase 2 implementation starts, not discovered mid-build.

**Output shape:** structured JSON — segments plus minutes — that a deterministic scheduler renders, not free text the UI parses creatively. Uses Gemini's structured-output/JSON-schema mode. Proposed shape (not yet implemented, sketched so Phase 2 has a starting point):

```typescript
interface DailyPracticePlanResponse {
  segments: Array<{
    type: 'review' | 'newLesson' | 'drill';
    targetId: string;    // a lessonId or exerciseId
    minutes: number;     // integer > 0
    rationale: string;   // ≤1 sentence, shown as a small caption
  }>;
  totalMinutes: number;  // must equal the sum of segments[].minutes
}
```

Full JSON-schema definition and validation rules: `AI_GUARDRAILS.md` §5.

**Cost tier:** cheapest available tier (`docs/PRODUCT_SPEC.md` §15).

**Not implemented for MVP** — full prompt-template authoring is intentionally deferred rather than pre-written now, consistent with §7's "needs real usage data first" reasoning: writing a production system instruction today would mean guessing at how `weakAreas`/`recentAccuracyTrend` should actually be weighted, before any real data exists to inform that.

## 10. Rate limits — cross-cutting summary

Full mechanics and the (currently missing) exact numbers: `AI_COST_CONTROL.md`. Summary of the shape — **two separate per-user daily counters, not one**, because their consequences on failure are too different to share a budget:

- **Tutor cap** — covers every live call `getTutorResponse` makes (both its Tutor Q&A and Theory-Explanation-cache-miss branches), checked and atomically incremented *before* Gemini is called — a request over the cap short-circuits to `{ ok: false, error: { code: 'RATE_LIMITED' } }` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.16 / `docs/ENGINEERING/ERROR_HANDLING.md` §4.3) without ever reaching Gemini. This is the cap `docs/PRODUCT_SPEC.md` §19/§29 name directly as "a daily cap on tutor calls."
- **Exercise Feedback safety-net cap** — a separate, deliberately more generous counter for `getExerciseFeedback`'s Gemini sub-call, sized so no plausible legitimate practice volume is ever throttled; it exists to catch a scripted-attempt or retry-storm bug, not to bound normal use. Exhausting it never blocks scoring — only `feedbackText` degrades to the canned line, exactly as on any other Gemini-side failure (`AI_ERROR_HANDLING.md` §5.1).
- Cache hits (Theory Explanation) and pool hits (Motivation/Nudge) do **not** count against either cap — they cost the app nothing.
- ⚠ **BLOCKING:** `docs/PRODUCT_SPEC.md` never states either cap's actual number. See `AI_COST_CONTROL.md` §5 for the proposed starting values and the full flag.

## 11. Cost tier — cross-cutting summary

| Feature | Tier | Cacheable? |
|---|---|---|
| Exercise Feedback | Cheapest | No |
| Tutor Q&A | Mid | No |
| Theory Explanation | Cheapest | **Yes** — per `(concept, skillLevel)` |
| Motivation / Nudge | Cheapest (rare live-call path only) | Pool-served by default, not a cache in the technical sense |
| Daily Practice Plan *(Phase 2)* | Cheapest | No |

Per `docs/PRODUCT_SPEC.md` §14: do not hardcode a specific Gemini model version anywhere in implementation. "Cheapest" and "mid" are relative tier choices to be checked against [ai.google.dev/gemini-api/docs/pricing](https://ai.google.dev/gemini-api/docs/pricing) at implementation time — the model lineup moves fast enough that a name written into this spec would be stale before the feature ships.

## 12. Data structures

This document does not own type definitions. Canonical shapes: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — `ExerciseVerdict` (§4.5), `AIMemorySummary` (§4.8), `TutorRequest` (§4.9), `TutorResponse` (§4.10). The Exercise Feedback and Motivation/Nudge prompt-input shapes used in §5 and §8 above are **not** currently defined in `TYPES_AND_CONTRACTS.md` — they exist today only as a convention repeated identically across `docs/PRODUCT_SPEC.md` §15 and `docs/ARCHITECTURE/DATA_FLOW.md` §4.2. Recommend they be formally added there (e.g. as `ExerciseFeedbackPromptInput`, `MotivationNudgePromptInput`) rather than continuing to exist only as copied JSON shapes across documents.

## 13. Dependencies

- Gemini API — server-side only, called from `functions/src/ai/` (`docs/PRODUCT_SPEC.md` §16, §19).
- `packages/music-engine` — the exclusive producer of every fact Gemini is given about correctness (§3 above).
- Firestore — source of `AIMemorySummary`, most-recent `PracticeSession`, and (proposed) the Theory Explanation cache (`AI_COST_CONTROL.md`).
- App Check — gates every Gemini-calling Function (`docs/PRODUCT_SPEC.md` §16, §19).
- `docs/AI/AI_PROMPTS.md` — the literal template text each feature above renders against.

## 14. Open questions raised while writing this document

Full categorized list, with the missing-rate-limit-number point, is in the final task report. Summarized here for anyone reading this file directly:

- ⚠ **BLOCKING** — no exact per-user daily rate-limit number is specified anywhere in `docs/PRODUCT_SPEC.md` (§10 above; full flag in `AI_COST_CONTROL.md` §5).
- ⚠ **IMPORTANT** — the Exercise Feedback input shape (`{exerciseType, expected, played, accuracyScore, missedNotes, attemptNumber, learnerLevel}`, used identically in `docs/PRODUCT_SPEC.md` §15 and `docs/ARCHITECTURE/DATA_FLOW.md`) is not defined in the canonical `TYPES_AND_CONTRACTS.md` and doesn't field-match `ExerciseVerdict`. Used as-is here for cross-document consistency; flagged for formal reconciliation (§12 above).
- ⚠ **IMPORTANT** — `learnerLevel` (Exercise Feedback's field name) and `skillLevel` (`AIMemorySummary`/`TutorRequest`'s canonical field name) appear to name the same beginner/returning concept but are spelled differently across the source documents. Both are populated from the same canonical `AIMemorySummary.skillLevel` value at assembly time; the naming should be unified in a future pass.
- ⚠ **IMPORTANT** — no callable Cloud Function is named anywhere in `docs/PRODUCT_SPEC.md` §20 for Theory Explanation or Motivation/Nudge, despite §15 defining five templates. §4's mapping table above is this document's proposed resolution (Theory Explanation as a routed mode of `getTutorResponse`; Motivation/Nudge as primarily client-side/pool-served with a rare fallback folded into `getExerciseFeedback`) — needs engineering sign-off before `functions/src/ai/` is structured around it.
- ⚠ **IMPORTANT** — no exact UI trigger is specified for Theory Explanation or Motivation/Nudge anywhere in `docs/PRODUCT_SPEC.md` §10. §7 and §8 above propose concrete trigger mechanisms; needs product confirmation.
- ⚠ **IMPORTANT** — anonymous-account-cycling as an unaddressed rate-limit-bypass vector (zero-friction anonymous sign-in, §18, means a scripted loop of new anonymous sessions each gets a fresh per-user quota). Not addressed by anything named in `docs/PRODUCT_SPEC.md` §19. Proposed mitigation in `AI_COST_CONTROL.md` §7.
- ⚠ **NON-BLOCKING** — `docs/PRODUCT_SPEC.md` §15's own table names the Theory Explanation tier "Cheap" where every other row and the canonical `TutorResponse.tier` enum (`'cheapest' | 'mid'`, exactly two values) says "Cheapest." Treated as synonymous throughout this document set.
- ⚠ **NON-BLOCKING** — `docs/PRODUCT_SPEC.md` §5 tags "one daily practice session generated for me" as unqualified `[MVP]`, while §6/§7 clarify only the rule-based version is MVP. Resolved per this document's explicit brief (Daily Practice Plan = Phase 2 throughout); noting the source document's own internal inconsistency for completeness.

## 15. Out of scope

- Prompt template text → `AI_PROMPTS.md`.
- Call-architecture mechanics, where template text physically lives → `AI_PROMPT_ARCHITECTURE.md`.
- `AIMemorySummary` computation/recomputation detail → `AI_CONTEXT_MODEL.md`.
- Output validation mechanics, injection defenses, structured-output schemas → `AI_GUARDRAILS.md`.
- Failure-mode-to-fallback mapping, error codes → `AI_ERROR_HANDLING.md`.
- Caching mechanics, rate-limit implementation, cost instrumentation and math → `AI_COST_CONTROL.md`.
- System-level component diagram and network boundary crossings → `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md`, `docs/ARCHITECTURE/DATA_FLOW.md`.

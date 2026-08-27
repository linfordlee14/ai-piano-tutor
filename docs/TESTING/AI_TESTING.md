# AI / Prompt Contract Testing

## 1. Purpose

Defines testing for the Gemini-calling Cloud Functions — deliberately **contract** tests, not quality tests. Per `docs/PRODUCT_SPEC.md` §26: *"not 'is Gemini's answer good' (expensive and unfalsifiable to assert on automatically) but contract tests — does the response fit the schema, respect the length cap, and does the fallback trigger correctly on an error or malformed response."* This distinction is the whole point of this document and should govern every test added to this suite going forward: if a proposed test's pass/fail depends on subjective judgment of Gemini's phrasing, it belongs in the human eval set (§7), not the automated suite.

**Source of truth:** `docs/PRODUCT_SPEC.md` §14 (the deterministic/AI boundary — non-negotiable), §15 (AI Prompt Architecture — per-template input/output contracts and guardrails), §19 (AI request protection), §26, §29. Validation rules: `docs/AI/AI_GUARDRAILS.md` (parallel batch — this document tests what that spec defines; treat it as authoritative on the exact validation logic once it exists). Contracts: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9–4.10 (`TutorRequest`, `TutorResponse`).

## 2. Scope

Covers: schema/shape validation of Gemini responses per template, length-cap enforcement, the fallback path when Gemini errors or returns malformed output, rate-limiting behavior, and the small human-reviewed eval set for the Exercise Feedback template. Does not cover: whether a given response is *pedagogically good* as an automated gate (explicitly rejected as unfalsifiable, §1), the Firestore plumbing around these Functions (`INTEGRATION_TESTING.md`), or full browser journeys through the Tutor screen (`E2E_TESTING.md` — including its Gemini-unavailable scenario, which tests the same fallback principle from the rendered-UI side rather than the Function-contract side).

## 3. The Boundary This Suite Exists to Protect

`docs/PRODUCT_SPEC.md` §14 states the rule this entire testing category serves: **Gemini is never asked to determine whether a note, chord, or scale was played correctly.** It receives an `ExerciseVerdict` the engine already computed (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5) and explains it in prose — it does not participate in producing it. This suite's job is to prove the *plumbing* around that boundary holds: that a template's input contract never lets raw `NoteEvent[]` data reach the prompt (§5 rule 3 in `TYPES_AND_CONTRACTS.md` — Gemini-facing types never contain raw note events, only already-summarized verdicts/memory), and that the guardrails preventing Gemini from inventing theory facts are actually enforced, not just described in a system prompt and hoped for.

## 4. Environment

- Every test in this suite mocks the Gemini client at the network/SDK boundary. **No test calls the real Gemini API** — that would be slow, non-deterministic, cost money on every CI run, and (per §1) is testing the wrong thing anyway.
- Mocks return controlled fixtures: well-formed responses matching each template's output contract, and deliberately malformed ones (empty string, truncated JSON, a response exceeding the length cap, a response containing an unrequested field) to drive the fallback-path tests.
- Rate-limiting tests run against the Firebase Emulator Suite (a per-user counter needs real, atomic Firestore state to test meaningfully) — this is the one sub-area of this suite that overlaps with `INTEGRATION_TESTING.md`'s environment; the overlap is intentional, since rate-limiting is simultaneously an AI-cost-control contract (this document) and a Function/Firestore integration concern.

## 5. Per-Template Contract Tests

For each template in `docs/PRODUCT_SPEC.md` §15, assert against its specific input/output contract:

### 5.1 Exercise Feedback (Cheapest tier)
- **Input contract:** the Function assembles `{exerciseType, expected, played, accuracyScore, missedNotes, attemptNumber, learnerLevel}` from server-held state (the just-computed `ExerciseVerdict` and lesson/learner context) — assert the prompt sent to the (mocked) Gemini client never includes a raw `NoteEvent[]`, only the already-summarized fields.
- **Output contract:** 2–3 sentences; assert a response is rejected/falls back if it's far outside that length band (see §6's length-cap test), and that the response text does not get parsed for any structured field this template doesn't define (it's prose, not the JSON-schema-mode template).
- **Never-invents-facts guardrail:** construct a mocked response that *does* state a theory fact not present in the input (e.g., names a chord quality the input `ExerciseVerdict` never mentioned) — this is the one guardrail that's hardest to test automatically as a *generation-time* prevention (that's a prompt-engineering/system-instruction concern owned by `docs/AI/AI_GUARDRAILS.md`), but the contract test suite can and should assert that IF the deployed validation logic includes any post-hoc fact-checking step (e.g., checking the response only references note/chord names present in the input `ExerciseVerdict`), that check actually catches the fixture. If no such post-hoc check exists, flag that as a gap against `docs/AI/AI_GUARDRAILS.md` rather than silently assuming the system instruction alone is sufficient — a system instruction is a strong hint to the model, not an enforced contract.

### 5.2 Tutor Q&A (Mid tier)
- **Input contract:** assert the Function-assembled `TutorRequest` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9) is built server-side from Firestore-held `AIMemorySummary` and the most recent `PracticeSession`, and that the client-supplied portion is *only* `learnerQuestion` — a test that attempts to pass extra client-supplied context fields (e.g. a fabricated `recentExerciseContext`) through the callable's public signature should find that signature rejects or ignores them entirely, never accepts a client-asserted performance history.
- **Length cap:** ≤150 words unless the learner explicitly asks for more — assert both the default cap and the "asked for more" exception path if the Function implements one.
- **`learnerQuestion` input validation:** length-capped before it's dropped into a prompt (§19) — assert an over-length or empty question is rejected with the typed error shape (`docs/PRODUCT_SPEC.md` §20) before any Gemini call is even attempted (saving cost on input that was never going to be valid).

### 5.3 Motivation / Nudge (Cheapest tier, mostly pre-written)
- Assert the common cases (e.g. a 7-day streak milestone) resolve from the pre-written pool without a live Gemini call at all (§15) — a test asserting the mock Gemini client's call count is `0` for these specific inputs is the correct way to verify this, not just checking the output text looks right.
- For inputs that do require a live call, assert the "one short line" output contract.

### 5.4 Theory Explanation (Cheap, cacheable)
- **Cache-hit path:** call twice with the same `(concept, skillLevel)` pair; assert the second call does not invoke the mocked Gemini client at all and returns `TutorResponse.cached: true` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.10).
- **Cache-miss path:** a new `(concept, skillLevel)` pair invokes Gemini and returns `cached: false`.
- This is explicitly called out in `docs/PRODUCT_SPEC.md` §15 as "likely the single highest-leverage cost optimization in the whole AI layer" — treat a cache-hit test that silently still calls the live/mocked client as a serious regression, not a minor miss.

### 5.5 Daily Practice Plan (Phase 2 — do not build this test yet)
- Deferred per `docs/PRODUCT_SPEC.md` §7's explicit pushback and §15's `(Phase 2)` tag. **No test for this template exists in any MVP milestone** — if a coding agent adds Gemini-personalized daily-practice-plan logic before Phase 2, that is itself the bug (`docs/ROADMAP/M4_GEMINI_TUTOR.md` §8 names this explicitly as out-of-scope), and this suite should stay empty here as a deliberate absence, not an oversight to "complete."

## 6. Fallback-Trigger Tests (Required for Every Template)

Per `docs/PRODUCT_SPEC.md` §15: *"a validated fallback — if a response fails validation (empty, absurdly long, malformed JSON for the structured templates), the UI shows a deterministic canned message instead of a broken screen."* For every template above, test at minimum:

```
- Mocked Gemini call throws / times out           → fallback triggers, typed error/fallback response returned
- Mocked Gemini returns an empty string            → fallback triggers
- Mocked Gemini returns a response far exceeding
  the template's length cap                        → fallback triggers (not silently truncated and shown as-is,
                                                       unless truncation is the documented behavior — confirm
                                                       against docs/AI/AI_GUARDRAILS.md once written)
- Mocked Gemini returns malformed JSON for a
  structured-output template                       → fallback triggers, not a client-side parse crash
- Mocked Gemini returns well-formed output          → fallback does NOT trigger (the negative case matters too —
                                                       a suite that always falls back would pass every test above
                                                       vacuously)
```

The last line matters: include at least one well-formed-response test per template specifically to prove the fallback logic is *conditional*, not a blanket override that happens to make every failure-mode test pass for the wrong reason.

This is the Function-contract half of the principle `E2E_TESTING.md` §5.3 verifies from the rendered-UI side (disabling Gemini entirely leaves the app functional) and is itself a named §29 acceptance line: *"Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly, not assumed."*

## 7. Rate Limiting

Per `docs/PRODUCT_SPEC.md` §19 and the §29 acceptance line *"Per-user rate limiting demonstrably blocks a scripted burst of tutor calls beyond the configured cap"*:

```
GIVEN an authenticated test user and the configured daily cap
WHEN the test issues (cap + 1) rapid tutor-call requests in a scripted burst
THEN the first `cap` calls succeed (or at least reach the Gemini-call step)
AND the (cap + 1)th call is rejected with a typed rate-limit error, BEFORE any
    Gemini call is attempted for that rejected request
AND the rejection is atomic — a burst of concurrent requests near the cap boundary
    does not allow more than `cap` successes through a race condition
```

**Gap to flag, not silently resolve:** neither `docs/PRODUCT_SPEC.md` nor `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` specifies where the rate-limit counter is persisted (no field for it exists on `UserProfile` in the canonical contracts) or what the exact daily cap value is. Write this test against whatever mechanism `docs/AI/AI_GUARDRAILS.md`/the eventual Function implementation defines, but do not invent a specific Firestore field path in this document — see this batch's final report for the flagged gap.

## 8. Human-in-the-Loop Eval Set (Exercise Feedback Only)

Per `docs/PRODUCT_SPEC.md` §26 and §15's guardrail note: a small, manually-reviewed eval set is *"a reasonable manual complement — not something to over-automate at this scale."* This is deliberately outside CI:

- A fixed set of representative `ExerciseVerdict` inputs (a range of accuracy scores, missed-note patterns, attempt numbers) run through the real Exercise Feedback template against the real Gemini API, off the CI critical path (a scheduled/manual job, not a PR gate).
- A human reviewer periodically checks tone (encouraging, not discouraging, per `docs/PRODUCT_SPEC.md` §5's "wrong key press treated as a normal part of learning" story) and factual grounding (does the explanation only reference what the input actually said) against the same set.
- This complements, but never replaces, §5.1's automated never-invents-facts contract check — the eval set catches subtle tone/quality drift the schema-level check can't see; the automated check catches hard schema violations the eval set's small sample size might miss on any given review cycle.
- **Not CI-blocking.** A bad review finding is a follow-up task (adjust the prompt template or its system instruction in `docs/AI/AI_PROMPTS.md`), not a merge-blocking failure — treating subjective eval results as a hard gate would violate this document's own §1 principle.

## 9. Error Handling

- Every Function under test returns the typed `{ok, data|error}` shape (`docs/PRODUCT_SPEC.md` §20) on every failure path exercised in §6 and §7 — a raw exception or an untyped error reaching the client is itself a test failure independent of whether the *right* fallback content eventually rendered.
- Prompt-injection hygiene (§19: a learner typing "ignore your instructions and…" should not extract the system prompt or trigger anything beyond plain text output): include at least one test with an adversarial `learnerQuestion` input and assert the Function's Gemini call remains scoped to plain-text output with no tool/function-calling surface exposed — this is a contract-shape assertion (no unexpected fields, no elevated capability exercised), not a claim that this suite can exhaustively prove prompt-injection resistance.

## 10. Acceptance Criteria

- [ ] Every template's output is validated against its schema/length-cap contract before reaching the client.
- [ ] Fallback triggers correctly on error, empty, malformed, and over-length mocked responses, for every template (§6).
- [ ] Fallback does NOT trigger on well-formed responses (the negative case, §6).
- [ ] A tutor response never states a note/chord/theory fact that contradicts the structured data it was given — enforced to whatever extent `docs/AI/AI_GUARDRAILS.md`'s validation logic supports, with any gap flagged rather than assumed covered (§5.1).
- [ ] Per-user rate limiting demonstrably blocks a scripted burst beyond the configured cap, atomically (§7).
- [ ] Theory Explanation caching is verified by call-count assertion, not output inspection alone (§5.4).
- [ ] The Exercise Feedback human eval set exists and runs on a periodic, non-CI-blocking cadence (§8).

## 11. Out of Scope

- Real Gemini API calls in CI — never (§4).
- Automated tone/quality scoring — explicitly human-reviewed only (§8).
- Daily Practice Plan template testing — Phase 2, not built yet (§5.5).
- Testing Gemini's underlying model behavior/accuracy in general (a Google-owned concern) — this suite tests *this application's* contract around that model, not the model itself.

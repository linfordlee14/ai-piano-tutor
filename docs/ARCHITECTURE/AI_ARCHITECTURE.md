# AI Architecture

**Status:** Draft v1.0 — architecture documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §14 (primary), also §2, §15, §16, §19, §20, §22
**Companion documents:** this document is the **architecture-level** companion to the more detailed `docs/AI/` directory (written in parallel — `docs/AI/GEMINI_SPEC.md`, `docs/AI/AI_PROMPTS.md`, `docs/AI/AI_GUARDRAILS.md`, `docs/AI/AI_COST_CONTROL.md`, `docs/AI/AI_CONTEXT_MODEL.md`). This document owns the boundary and the call pattern; it does not own prompt-template text or field-by-field guardrail implementation. Also see `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md`, `docs/ARCHITECTURE/DATA_FLOW.md`, `docs/ARCHITECTURE/SECURITY_ARCHITECTURE.md`, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`.

## 1. Purpose

State, once and unambiguously, what Gemini is and is not allowed to do in this system, how it is called, when it is called, and why the call pattern is shaped the way it is. This is the document `SYSTEM_ARCHITECTURE.md`, `APPLICATION_ARCHITECTURE.md`, and `DATA_FLOW.md` point to for the full version of the boundary they each restate in summary.

## 2. Scope

In scope: the deterministic/Gemini boundary, the call architecture (client → Cloud Function → Gemini), when Gemini is and is not invoked, and model tiering at the architecture level.

Out of scope: the exact prompt template text and input/output contracts for each of the five templates (→ `docs/AI/AI_PROMPTS.md`), the theory-explanation cache mechanism and cost-control levers in depth (→ `docs/AI/AI_COST_CONTROL.md`), the `AIMemorySummary` aggregation logic (→ `docs/AI/AI_CONTEXT_MODEL.md`), guardrail implementation specifics like exact length caps and validation regexes (→ `docs/AI/AI_GUARDRAILS.md`), and the security mechanisms that protect the AI call path — App Check, rate limiting, credential storage (→ `docs/ARCHITECTURE/SECURITY_ARCHITECTURE.md`, though this document restates the *reasons* those mechanisms exist where relevant to the call pattern).

## 3. Requirements — the non-negotiable rule

**Gemini must never determine whether a note or chord is correct, calculate accuracy, calculate timing, calculate scores, determine lesson completion, determine XP, determine streaks, or validate user-submitted performance data.** Every one of those is deterministic application logic, computed by `packages/music-engine` and invoked server-side inside Cloud Functions (`MUSIC_ENGINE_ARCHITECTURE.md`, `DATA_FLOW.md`). Gemini is the AI teaching/personalization layer only: it receives an already-computed `ExerciseVerdict` and explains it in prose.

`docs/PRODUCT_SPEC.md` §14 states the reasoning directly, and it's worth keeping intact rather than paraphrasing: *"An LLM asked 'was C-E-G played, is that a C major chord' will usually get it right — but 'occasionally wrong about music theory' is disqualifying for something presenting itself as a piano teacher. This isn't a style choice; it's the thing that makes the AI layer trustworthy enough to ship."*

This requirement is restated explicitly in `SYSTEM_ARCHITECTURE.md`, `APPLICATION_ARCHITECTURE.md`, `DATA_FLOW.md`, and here — by design, so that no reader of any one architecture document can miss it.

## 4. Architecture

### 4.1 The boundary, in full

Reproduced from `docs/PRODUCT_SPEC.md` §14:

| Deterministic engine handles | Gemini handles |
|---|---|
| Note/chord/scale recognition & validation | Natural-language explanation of a diagnosis the engine already produced |
| Keyboard/MIDI input mapping | Conversational Q&A ("why are there black keys") |
| Timing & accuracy scoring | Turning a "weak areas" list into a plain-language suggestion |
| Progress/streak/XP calculation | Motivational framing and encouragement copy |
| Lesson sequencing & unlock logic | Lesson-content authoring assistance (offline, for the content team — not a runtime call) |
| "What exactly did the learner get wrong" | — |

Every row in the left column is owned by `packages/music-engine` and the server-side logic around it — see `MUSIC_ENGINE_ARCHITECTURE.md` and `DATA_FLOW.md` for exactly where each computation happens. Nothing in the right column ever writes to `PracticeSession`, `ProgressSummary`, or any XP/streak field; Gemini's output only ever becomes `TutorResponse.text` or the Exercise Feedback prose returned alongside (never instead of) a deterministic `ExerciseVerdict`.

### 4.2 `ExerciseVerdict` is read-only input to Gemini

Restated from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5, because it's the sharpest, most concrete expression of §3's rule: **"Gemini never produces, modifies, or is asked to sanity-check any field on this object... Gemini receives a completed `ExerciseVerdict` as read-only input and explains it in prose; it does not participate in producing it."** Architecturally, this means the data flow into a Gemini prompt is strictly one-directional and happens strictly *after* `packages/music-engine` has already finished computing the verdict (`DATA_FLOW.md` §4, step 8) — there is no code path where a Gemini response feeds back into the verdict object, and no code path where Gemini is asked to double-check the engine's math.

### 4.3 Call architecture

```
client → Cloud Function (callable) → assembles prompt from template + structured learner context → calls Gemini server-side → returns a structured, validated response → client renders
```

**There is no arrow from client to Gemini, anywhere, ever** — restated from `SYSTEM_ARCHITECTURE.md` §4.3's Property 2. This is enforced two ways, not one:
1. **Credential-level:** the client never holds a Gemini API key or any credential capable of calling Gemini (`SECURITY_ARCHITECTURE.md` §Gemini credential protection) — there is nothing for a client-side call to authenticate with even if the code existed.
2. **Context-level, and arguably more important:** the tutor's context (`recentExerciseContext`, `learnerMemorySummary`) has to be assembled from data the client cannot write (`PracticeSession`, `AIMemorySummary` — `SECURITY_ARCHITECTURE.md`, `DATA_FLOW.md` §5). Even a Firebase AI Logic-style client-SDK pattern (a lighter-weight alternative Google offers for simple chat, noted as a fallback option in `docs/PRODUCT_SPEC.md` §1) is deliberately **not** used as the primary pattern here, because it would let the client construct its own request context — and the client cannot be trusted to do that honestly (`docs/PRODUCT_SPEC.md` §1). The thin Cloud Functions layer stays primary specifically because *this* app's tutor needs server-assembled learner context injected into every prompt.

Full request/response trace for both Gemini-calling flows (`getExerciseFeedback`'s Exercise Feedback call, `getTutorResponse`'s Tutor Q&A call): `DATA_FLOW.md` §4–§5.

### 4.4 When Gemini is called — and when it isn't

Only on:
- **Explicit learner action** — opening the tutor chat, tapping "ask why," requesting today's plan (Phase 2).
- **End of a scored exercise** — the Exercise Feedback template call, bundled into `getExerciseFeedback`'s response alongside the deterministic verdict (`DATA_FLOW.md` §4.3).

**Never:**
- Per keystroke.
- Polled.
- "Just in case."

`docs/PRODUCT_SPEC.md` §14 calls this "the biggest lever on both cost... and perceived speed" — a piano app that phones home on every key press would feel laggy and get expensive fast, for no user-facing benefit. This rule is what makes `SYSTEM_ARCHITECTURE.md` §4.3 Property 1 (the closed local loop) compatible with an AI layer existing at all: the AI layer sits entirely outside the loop's critical path by construction, triggered only at loop *boundaries* (exercise end) or by an out-of-loop user action (opening the tutor).

### 4.5 Model tiering

- **Cheapest/fastest available tier** — structured feedback and motivational copy: high volume, low creative demand (Exercise Feedback, Motivation/Nudge, Daily Practice Plan when it ships in Phase 2).
- **Mid tier** — open-ended tutor conversation (Tutor Q&A), where response quality is actually load-bearing.

**Do not hardcode a specific Gemini model version in this document or in the implementation brief.** Per `docs/PRODUCT_SPEC.md` §14: "Gemini's model lineup has moved fast... whatever's cheapest-and-good-enough should be re-checked against [ai.google.dev's current pricing page](https://ai.google.dev/gemini-api/docs/pricing) at implementation time rather than locked into this spec." This document follows the same instruction — model selection is a tier (cheapest / mid), not a pinned version string, and `TutorResponse.tier: 'cheapest' | 'mid'` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.10) is recorded per response specifically so tier choices can be audited and adjusted over time without a contract change.

## 5. Data structures

- `TutorRequest` — server-assembled only; the client-facing function signature accepts only `{ learnerQuestion: string }` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9). Full assembly trace: `DATA_FLOW.md` §5.
- `TutorResponse` — `{ text, suggestedAction?, tier, cached }` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.10). Never persisted as a growing log; the only durable trace of AI interaction is the `AIMemorySummary` aggregate, which is derived from `PracticeSession` data, not from tutor conversation content (`docs/PRODUCT_SPEC.md` §17, §25).
- `AIMemorySummary` — a single, rewritten-in-place Firestore document (`users/{uid}/aiMemory/summary`), never a growing transcript (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8). This is the entire "memory" mechanism — deliberately not a raw conversation log, both for cost (`docs/PRODUCT_SPEC.md` §31) and privacy (`docs/PRODUCT_SPEC.md` §25) reasons.
- Per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5, rule 3: **Gemini-facing types never contain a raw `NoteEvent[]`** — only already-summarized `ExerciseVerdict`/`AIMemorySummary` data. Gemini is never in a position to "recompute" anything from raw note data, which would be an indirect way of violating §3's rule.

## 6. Interfaces / contracts

| Function | Gemini template(s) it calls | Notes |
|---|---|---|
| `getExerciseFeedback` | Exercise Feedback | Also performs the deterministic re-scoring (`DATA_FLOW.md` §4) — this function's Gemini call is a same-round-trip addition to scoring, not a separate request |
| `getTutorResponse` | Tutor Q&A (and, per its caching behavior, effectively Theory Explanation when the question matches a cacheable `(concept, skillLevel)` pattern) | The only function whose primary job is a Gemini call; full trace: `DATA_FLOW.md` §5 |
| *(Phase 2, not built)* `generateDailyPlan` | Daily Practice Plan | Deferred until there's real usage data to personalize against (`docs/PRODUCT_SPEC.md` §7 pushback) |

**Output contract, every template:** a hard output-length cap, a system instruction forbidding invented note/chord/theory facts (the prompt always defers to the structured data it was given — restating §3's rule at the prompt level, not just the architectural one), and a validated fallback. The Daily Practice Plan template (Phase 2) uses Gemini's structured-output/JSON-schema mode so the client parses reliably instead of regex-scraping prose. Full per-template contracts: `docs/AI/AI_PROMPTS.md`.

**A note on the five templates vs. four MVP callables:** `docs/PRODUCT_SPEC.md` §15 defines five prompt templates (Exercise Feedback, Tutor Q&A, Daily Practice Plan, Motivation/Nudge, Theory Explanation), but §20 names only two Gemini-calling callables for MVP (`getExerciseFeedback`, `getTutorResponse`). This document's working assumption, used consistently across the architecture set: Theory Explanation is a cache-checked code path *within* `getTutorResponse` (triggered when a question matches a recognized `(concept, skillLevel)` pair, explaining why `TutorResponse.cached` exists as a field), and Motivation/Nudge is, per §15's own text ("the top few common cases... don't need a live call at all; a small pre-written pool is faster and free"), predominantly a client-or-Function-side deterministic copy selection with Gemini as a rare fallback — not a dedicated callable. This is flagged as an open question in the parent report; it does not weaken the boundary in §3 either way it's ultimately implemented, but it does affect `docs/AI/AI_PROMPTS.md` and `functions/src/ai/`'s file layout.

## 7. Dependencies

- **Gemini API** — via the official server-side SDK, called only from within Cloud Functions.
- **Secret Manager** — holds the Gemini credential; see `SECURITY_ARCHITECTURE.md` §Gemini credential protection.
- **Cloud Functions (2nd gen)** — the exclusive runtime environment for any Gemini call.
- **App Check** — gates the Gemini-calling Functions specifically (`docs/PRODUCT_SPEC.md` §16); see `SECURITY_ARCHITECTURE.md`.

## 8. Security considerations

Full treatment: `SECURITY_ARCHITECTURE.md`. Specifically relevant to the call *pattern* described here (rather than to Firestore rules or credential storage mechanics): the server-assembly requirement in §4.3 is itself a security property, not just an engineering convenience — it's what stops a learner from fabricating their own "recent performance" to manipulate the tutor into, say, claiming a skill level or streak they don't have. Prompt-injection hygiene (a learner typing "ignore your instructions and…" into the tutor chat) is bounded by the fact that the Gemini call is scoped to plain text generation only, with no tool- or function-calling access to anything sensitive — so even a successful injection can produce at most a strange text response, never an action.

## 9. Performance considerations

Since calls are already gated to explicit user action or exercise-end (§4.4), 1–3 seconds of Gemini latency is acceptable as long as the UI shows a clear "thinking" state (`docs/PRODUCT_SPEC.md` §22) — this is explicitly *not* a place to over-invest engineering effort chasing speed. This is a direct consequence of the call-frequency guarantee in §4.4: because Gemini is never on the critical path of anything time-sensitive (piano latency lives entirely in `AUDIO_ARCHITECTURE.md`'s ~50ms budget, unrelated to this document), a few seconds of AI latency has no compounding effect elsewhere in the system.

## 10. Error handling

- **Validated fallback, every call:** if a Gemini response fails validation (empty, absurdly long, malformed JSON for structured templates), the UI shows a deterministic canned message instead of a broken screen (`docs/PRODUCT_SPEC.md` §15).
- **Gemini entirely down:** every other screen must remain fully functional (§3's requirement extends to availability, not just correctness) — the AI Tutor screen specifically shows a "temporarily unavailable, here's what to practice instead" fallback card built from deterministic data (`docs/PRODUCT_SPEC.md` §10), never a dead end. This is an explicit acceptance criterion (§11 below), tested by disabling the Function entirely, not assumed.
- **Rate limit exceeded:** short-circuits to the `{ ok: false, error }` shape (`SYSTEM_ARCHITECTURE.md` §6) before Gemini is ever called — see `SECURITY_ARCHITECTURE.md` and `DATA_FLOW.md` §5, step 5.

## 11. Testing requirements

Per `docs/PRODUCT_SPEC.md` §26, AI/prompt testing is explicitly **not** "is Gemini's answer good" (expensive and unfalsifiable to assert on automatically). Instead:
- **Contract tests:** does the response fit the schema, respect the length cap, and does the fallback trigger correctly on an error or malformed response.
- **A small human-reviewed eval set** for the Exercise Feedback template specifically, reviewed periodically for tone — a reasonable manual complement, not something to over-automate at this scale.
- **The disabled-Gemini test:** disabling the Gemini Function entirely and confirming every other screen still works — this is as much a test of §3's boundary as it is a test of error handling (§10).

## 12. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29:
- [ ] A tutor response never states a note/chord/theory fact that contradicts the structured data it was given.
- [ ] Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly, not assumed.
- [ ] Per-user rate limiting demonstrably blocks a scripted burst of tutor calls beyond the configured cap.

## 13. Out of scope

- Exact prompt template text, input/output field contracts for all five templates → `docs/AI/AI_PROMPTS.md`
- Theory-explanation caching mechanism, per-call cost instrumentation → `docs/AI/AI_COST_CONTROL.md`
- `AIMemorySummary` aggregation logic (how weak notes/chords are derived from recent `PracticeSession`s) → `docs/AI/AI_CONTEXT_MODEL.md`
- Guardrail implementation specifics (exact length caps, input sanitization rules) → `docs/AI/AI_GUARDRAILS.md`
- App Check configuration, rate-limit counter implementation, Secret Manager setup → `docs/ARCHITECTURE/SECURITY_ARCHITECTURE.md`, `docs/SECURITY/`
- Full request/response sequencing → `docs/ARCHITECTURE/DATA_FLOW.md`

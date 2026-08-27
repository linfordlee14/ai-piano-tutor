# AI Cost Control

**Status:** Draft v1.0 — specification only, no implementation yet; contains one BLOCKING open question
**Source of truth:** `docs/PRODUCT_SPEC.md` §14, §15, §19, §25, §31; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.10
**Companion documents:** `docs/AI/GEMINI_SPEC.md` §10–11, `docs/AI/AI_PROMPT_ARCHITECTURE.md`, `docs/AI/AI_ERROR_HANDLING.md` (the `RATE_LIMITED` fallback UX this document's cap triggers), `docs/ARCHITECTURE/DATA_FLOW.md` (explicitly defers the theory-explanation cache's key/storage mechanism to this document — see §4 below)

## 1. Purpose

Turn `docs/PRODUCT_SPEC.md` §31's cost levers into implementable mechanics: the exact caching design, the exact (proposed) rate-limit numbers, the per-call instrumentation schema, and the cost math those numbers are meant to bound — with the one number the source spec never provides flagged clearly rather than silently invented.

## 2. Scope

In scope: every cost lever named in `docs/PRODUCT_SPEC.md` §14/§15/§31, made concrete — call-gating, tiering, caching mechanics, rate-limit numbers and their enforcement, an unaddressed abuse vector this document surfaces, spend instrumentation, and illustrative cost math.

Out of scope: prompt template text (`AI_PROMPTS.md`), output validation (`AI_GUARDRAILS.md`), the client-facing UX of a rate-limited or degraded call (`AI_ERROR_HANDLING.md`).

## 3. The levers, restated as a checklist

`docs/PRODUCT_SPEC.md` §31, each lever mapped to where it's actually implemented in this document set:

| Lever | Mechanism | Owned in |
|---|---|---|
| Called only on explicit user action, never per keystroke | Structural — no polling code path exists for any of the five templates | `GEMINI_SPEC.md` §5–9 (trigger/non-trigger per feature) |
| Cheapest tier for high-volume templates; mid tier only for open-ended tutor chat | Per-template tier assignment | `GEMINI_SPEC.md` §11 |
| Cache Theory Explanation per `(concept, skillLevel)` | Cache key structure and storage | §4, this document |
| Hard per-user daily rate caps | Two separate counters, atomic check-and-increment | §5, this document |
| Short, structured-context prompts, not raw transcripts | No conversation history retained; prompts built from small structured summaries | `AI_CONTEXT_MODEL.md` |
| Instrument spend from day one | Per-call log schema | §6, this document |

## 4. Theory Explanation caching — key and storage

`docs/PRODUCT_SPEC.md` §15 calls this "likely the single highest-leverage cost optimization in the whole AI layer." `docs/ARCHITECTURE/DATA_FLOW.md` §7 explicitly defers "the theory-explanation cache's key/storage mechanism" to this document, so it's specified concretely here, not left as a restatement of the principle.

**Cache key:** the exact pair `(concept, skillLevel)` — e.g. `"why-are-there-black-keys"` × `"beginner"`. Both values are drawn from the same small, curated, closed vocabularies already used for routing (`GEMINI_SPEC.md` §7) and skill-level classification (`AIMemorySummary.skillLevel`), so the key space is small and fully enumerable — there is no risk of unbounded cache growth from this template, unlike a cache keyed on arbitrary free text would be.

**Storage:** a Firestore collection, `theoryExplanationCache/{concept}__{skillLevel}` (double-underscore separator; concept ids are kebab-case by convention, so this can't collide). This reuses the datastore already in use rather than introducing a new caching service — no new infrastructure for MVP scale, consistent with `docs/PRODUCT_SPEC.md` §16/§32's general "don't add a dependency without a reason" posture. Document shape:

```typescript
interface TheoryExplanationCacheEntry {
  concept: string;
  skillLevel: 'beginner' | 'returning';
  text: string;        // the cached response text, ready to return as TutorResponse.text
  createdAt: FirebaseFirestore.Timestamp;
  hitCount: number;     // incremented on every cache read — cheap usage signal, no separate analytics query needed
}
```

**Security rule:** public-read, write only via the Admin SDK — the exact same pattern already established for `lessons` (`docs/PRODUCT_SPEC.md` §19), for the same underlying reason: this is non-personal, shared content, safe to read broadly and safe to trust only when Cloud Functions wrote it. `allow read: if true; allow write: if false;`

**Read path (inside `getTutorResponse`, on a concept match):** read `theoryExplanationCache/{concept}__{skillLevel}`; if present, increment `hitCount` and return `{ text, tier: 'cheapest', cached: true }` — zero Gemini call, zero cost beyond one Firestore read. If absent, call Gemini with the Theory Explanation template, and on a **valid** response (`AI_GUARDRAILS.md` §3), write the new cache entry before returning `{ text, tier: 'cheapest', cached: false }`. **Nothing is ever written to the cache on a failed or invalid response** (`AI_ERROR_HANDLING.md` §5.2) — a transient Gemini hiccup must never permanently poison an entry every future learner would then receive.

**Invalidation:** since the content is standard music theory that doesn't change, no TTL or scheduled expiry is needed for MVP — a cache entry is correct indefinitely by default. The one case that needs an escape hatch is a human eval pass (`docs/PRODUCT_SPEC.md` §26) catching a cached response that's subtly wrong or off-tone after the fact: the fix is a manual delete of that one document (a one-line admin script, not a feature to build), which causes the next matching request to regenerate and re-cache it. Not worth automating for MVP scale (a handful of concepts × 2 skill levels).

## 5. Hard per-user daily rate caps

`docs/PRODUCT_SPEC.md` §19 requires "a per-user rate-limit counter checked and atomically incremented inside the Function (e.g., a daily cap on tutor calls for free-tier users)" and §29 requires this be demonstrably enforced. **Two separate counters**, not one, because a cap exhausted on one has a completely different consequence than a cap exhausted on the other (`GEMINI_SPEC.md` §10, `AI_ERROR_HANDLING.md` §5):

### 5.1 The numbers — ⚠ BLOCKING, none of these are ratified

`docs/PRODUCT_SPEC.md` never states an exact cap anywhere — §19 says "a daily cap," §29 says "the configured cap," and §31 discusses cost in aggregate, illustrative terms only. **A Cloud Function cannot enforce "a cap" without an actual integer.** This document proposes starting values so implementation isn't blocked, explicitly marked as needing product sign-off before `functions/src/ai/` ships:

| Counter | Authenticated user | Anonymous user | Rationale |
|---|---|---|---|
| **Tutor cap** (every live `getTutorResponse` call — Tutor Q&A and Theory-Explanation cache-misses combined) | **20 / day** *(PROPOSED)* | **5 / day** *(PROPOSED)* | The number named as a starting anchor in this document's own brief. Generous enough that a genuinely curious learner never notices it in normal use — 20 questions in one day is already a lot of chat for a practice app — while bounding worst-case cost per account to a small, known number. Anonymous gets a lower allowance because minting a fresh anonymous account is free and instant (§7 below), so the per-account cap alone is a weaker defense there. |
| **Exercise Feedback safety-net cap** (`getExerciseFeedback`'s Gemini sub-call) | **100 / day** *(PROPOSED)* | **25 / day** *(PROPOSED)* | Sized so no plausible legitimate practice volume is ever throttled — even an unusually dedicated multi-hour practice day is very unlikely to complete 100 separate scored exercise attempts. Exists to catch a scripted/fabricated-attempt loop or a retry-storm bug (`docs/PRODUCT_SPEC.md` §31's explicit worry: "a bug that quietly removes one of the guardrails"), not to bound normal use — exhausting it never blocks scoring, only degrades `feedbackText` to the canned fallback (`AI_ERROR_HANDLING.md` §5.1). |

Motivation/Nudge's rare live-call path is self-limiting to at most one call per learner per day by its own trigger design (`GEMINI_SPEC.md` §8) and does not need a separate counter — implementers may fold it into the Exercise Feedback safety-net counter if maintaining one fewer counter is preferred.

### 5.2 Enforcement mechanics

**Storage:** `users/{uid}/aiUsage/{YYYY-MM-DD}` — one document per user per calendar day (UTC), holding both counters:

```typescript
interface DailyAIUsage {
  tutorCallCount: number;
  exerciseFeedbackCallCount: number;
}
```

Keying by date means the cap resets automatically at UTC midnight with no scheduled reset job — a new day is simply a new, absent document, read as zero. Old daily-usage documents accumulate; a Firestore TTL policy (a native feature, not custom code) can be set to auto-delete documents older than, say, 30 days, since nothing needs them past that window.

**Check-and-increment, atomically, before the call:** performed inside a Firestore transaction so two concurrent requests from the same user can't both read "19, under the cap of 20" and both proceed, landing at 21. The transaction reads the current count, and if it's already at or over the cap, aborts and returns `RATE_LIMITED` **without incrementing and without calling Gemini** (`docs/ARCHITECTURE/DATA_FLOW.md` §5.2 step 5's "short-circuits... before Gemini is ever called," applied precisely). If under the cap, the transaction increments the relevant counter and commits — this is the point at which a slot is considered "spent," even if the subsequent Gemini call then fails for an unrelated reason (network, timeout). This is a deliberate simplicity choice: refunding a slot on a downstream technical failure would need a second write and a reason to trust the failure was genuinely transient rather than attacker-induced; not refunding is simpler, and the cap is generous enough (§5.1) that an occasional lost slot from a genuine transient failure is not a real user-facing problem.

## 6. Instrumenting spend

`docs/PRODUCT_SPEC.md` §31: "a simple per-call log with token counts, not just a monthly bill review... catching a cost regression in week one is cheap; catching it after a usage spike is not." One structured log entry per Gemini call, every call, from day one:

```typescript
interface AICallLogEntry {
  timestamp: number;
  userId: string;
  template: 'exerciseFeedback' | 'tutorQA' | 'theoryExplanation' | 'motivationNudge';
  tier: 'cheapest' | 'mid';
  cacheHit: boolean;              // true only for Theory Explanation cache hits
  inputTokens: number;
  outputTokens: number;
  latencyMs: number;
  outcome: 'success' | 'validationFailed' | 'geminiUnavailable' | 'rateLimited';
}
```

**Where it lives:** structured Cloud Logging entries emitted from inside each Gemini-calling Function — no new logging infrastructure. This reuses exactly the mechanism `docs/PRODUCT_SPEC.md` §25 already endorses for product analytics ("BigQuery export... is the natural upgrade path if deeper querying is ever needed") applied to spend specifically: structured logs are exportable to BigQuery for aggregation (calls per day, spend by template, cache-hit rate) without building a bespoke pipeline. For MVP, a periodic query against exported logs is sufficient — a full real-time cost dashboard is reasonable to build later, but "simple per-call log" (the spec's own words) does not require one on day one.

**What this catches that a monthly bill review can't:** a cache that silently stops hitting (visible immediately as `cacheHit` trending toward `false` for a concept that used to hit) or a retry loop firing Gemini calls in a tight cycle (visible as a spike in `template`/`userId` call volume, hours before it would show up as a noticeable line item on a bill) — precisely the two failure patterns `docs/PRODUCT_SPEC.md` §31 names as "the thing to actually watch for."

**Explicitly not logged here:** `learnerQuestion` text or `feedbackText`/`TutorResponse.text` content — this is a spend/volume log, not a content log, consistent with `docs/PRODUCT_SPEC.md` §25's privacy boundary (product-usage and volume metrics only, never conversation content) and `AI_CONTEXT_MODEL.md` §8's "no raw conversation history retained" rule applied to logging infrastructure specifically, not just to the AI memory store.

## 7. An unaddressed abuse vector: anonymous-account-cycling

⚠ **IMPORTANT, flagged rather than silently resolved.** `docs/PRODUCT_SPEC.md` §2/§18 make anonymous sign-in explicitly zero-friction and instant — a deliberate, correct product decision (the north-star metric depends on it). But a per-`uid` rate cap, however well-chosen (§5.1), is only as strong as `uid`s are expensive to obtain. A script calling `signInAnonymously()` in a loop mints a fresh Firebase uid — and therefore a fresh daily cap — on every iteration, for free, indefinitely. Nothing named in `docs/PRODUCT_SPEC.md` §19's abuse-prevention list addresses this specifically: the session-write-throttling rule there ("rejecting a new `practiceSessions` write if the learner's previous session completed less than a few seconds ago") stops XP-farming, not cap-cycling, and App Check attests that traffic comes from the real app's code — it does not, and is not designed to, prove that a human is driving each call rather than a script automating the real app's own client code in a real browser.

**Proposed, non-blocking mitigation for MVP:** the lower anonymous cap already proposed in §5.1 (5/day tutor, 25/day exercise-feedback) bounds the damage *per account*, and combined with App Check (blocking anything that isn't the real client) raises the bar from "trivial" to "requires actually scripting a browser," which is a meaningfully higher-effort attack than a bare API loop. This is judged adequate for MVP launch scale, not a complete solution — flagged honestly as a residual risk rather than claimed as solved.

**Proposed hardening, if telemetry ever shows this happening (Phase 2+, not built speculatively):** a secondary, coarser-grained counter not keyed by `uid` alone — most simply, an IP-based counter alongside the per-user one. This has known weaknesses of its own (shared IPs behind NAT/school/corporate networks would share a budget unfairly), which is exactly why it isn't proposed as an MVP requirement — it's a lever to pull only if real abuse data justifies its UX cost, matching the spec's repeated "don't build a mitigation speculatively, build it when telemetry shows it's needed" posture elsewhere (`docs/PRODUCT_SPEC.md` §12's SoundFont fallback, §23's PWA scope).

## 8. Illustrative cost math — explicitly not a locked-in number

`docs/PRODUCT_SPEC.md` §31 already states this caveat and this document inherits it rather than re-deriving a new figure that would itself go stale: "pricing moves fast enough that a hardcoded number here would be stale within months... the cheapest Gemini tiers were priced at well under $1 per million input tokens [as of an August 2026 check]... even a few thousand short tutor calls a day... lands in a low-hundreds-of-dollars-a-month range." Check current pricing at [ai.google.dev/gemini-api/docs/pricing](https://ai.google.dev/gemini-api/docs/pricing) before relying on any number, including this one, at implementation time.

What §5's proposed caps add to that math, without asserting a new dollar figure of their own: **the caps put a hard ceiling on any single account's worst-case daily cost**, independent of what the per-token price happens to be at any given moment. A user maxing out both caps every single day (20 tutor calls + 100 exercise-feedback calls) is a bounded, known, small quantity of tokens — the caps convert "what if usage spikes" from an open-ended risk into an arithmetic one (`max calls/day × typical tokens/call × current price/token × user count`), which is the actual point of having a cap at all, independent of exactly where per-token pricing sits this quarter.

## 9. Data structures

```typescript
// Firestore: users/{uid}/aiUsage/{YYYY-MM-DD}
interface DailyAIUsage {
  tutorCallCount: number;
  exerciseFeedbackCallCount: number;
}

// Firestore: theoryExplanationCache/{concept}__{skillLevel}
interface TheoryExplanationCacheEntry {
  concept: string;
  skillLevel: 'beginner' | 'returning';
  text: string;
  createdAt: FirebaseFirestore.Timestamp;
  hitCount: number;
}

// Cloud Logging, one entry per Gemini call
interface AICallLogEntry {
  timestamp: number;
  userId: string;
  template: 'exerciseFeedback' | 'tutorQA' | 'theoryExplanation' | 'motivationNudge';
  tier: 'cheapest' | 'mid';
  cacheHit: boolean;
  inputTokens: number;
  outputTokens: number;
  latencyMs: number;
  outcome: 'success' | 'validationFailed' | 'geminiUnavailable' | 'rateLimited';
}
```

None of these are in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — all three are server-internal (never sent to or read by the client), which is why they're specified here rather than proposed as additions to the canonical shared-types file the way `GEMINI_SPEC.md` §12 proposes for the Exercise Feedback prompt-input shape.

## 10. Security considerations

- The rate-limit counter documents (`users/{uid}/aiUsage/**`) follow the same ownership pattern as every other scored/tracked field on the user: readable by the owning uid (so a client could optionally show "X tutor questions left today"), writable only by Cloud Functions via the Admin SDK — a modified client cannot reset or inflate its own quota.
- The theory-explanation cache is public-read by design (§4) — this is safe specifically because §4's write path guarantees only validated, fully general, non-personal content is ever written there; the guardrail in `AI_GUARDRAILS.md` §3 (rejecting any first/second-person performance claim in this template's output) is what makes public-read safe rather than a privacy leak.
- Spend logs (§6) never contain learner-authored or Gemini-generated content — only volume/cost metadata — so they carry no privacy classification beyond ordinary operational logs.

## 11. Dependencies

- Firestore — both counters and the cache live there; no new datastore.
- Firestore TTL policies — proposed for pruning old `aiUsage` daily documents (§5.2), a native feature, not custom cleanup code.
- Cloud Logging (+ optional BigQuery export) — spend instrumentation (§6), reusing the mechanism `docs/PRODUCT_SPEC.md` §25 already endorses for analytics.
- App Check — the first layer of the anonymous-cycling mitigation (§7).

## 12. Testing requirements

- A test asserting a Firestore transaction race (two concurrent calls near the cap boundary) never allows the counter to exceed the configured cap — the specific scenario §5.2's atomic-transaction design exists to prevent.
- `docs/PRODUCT_SPEC.md` §29's own acceptance criterion: a scripted burst of tutor calls beyond the configured cap is demonstrably blocked, and the (`n+1`)th call never reaches Gemini at all (assert on a call-count mock, not just on the returned error).
- A cache-hit test: two consecutive Theory Explanation requests for the same `(concept, skillLevel)` result in exactly one Gemini call and two returned responses, the second with `cached: true`.
- A cache-poisoning-prevention test: a mocked invalid/failed Gemini response for a cache-miss must not result in a cache document being written.
- A log-emission test: every one of the four call outcomes in `AICallLogEntry.outcome` is asserted to produce exactly one log entry, so a future refactor can't silently drop logging on one path.

## 13. Acceptance criteria

- [ ] Per-user rate limiting demonstrably blocks a scripted burst of tutor calls beyond the configured cap (`docs/PRODUCT_SPEC.md` §29, verbatim).
- [ ] Exhausting the Exercise Feedback safety-net cap never blocks or delays exercise scoring — only `feedbackText` degrades.
- [ ] A Theory Explanation cache hit produces zero Gemini API calls, verified by a call-count assertion, not just by response content.
- [ ] Every Gemini call, success or failure, produces exactly one `AICallLogEntry`.
- [ ] Both cap numbers in §5.1 are confirmed or revised by product before `functions/src/ai/` is implemented against them.

## 14. Open questions

- ⚠ **BLOCKING** — `docs/PRODUCT_SPEC.md` never specifies an exact per-user daily rate-limit number for tutor calls (§19, §29). §5.1 proposes starting values (20/day tutor, 100/day Exercise Feedback safety-net, both authenticated; 5/day and 25/day anonymous) — needs product sign-off.
- ⚠ **IMPORTANT** — anonymous-account-cycling as a rate-limit-bypass vector is not addressed by anything named in `docs/PRODUCT_SPEC.md` §19. §7 proposes a lower anonymous cap plus App Check as an MVP-adequate (not complete) mitigation, with IP-based secondary throttling flagged as a Phase 2 lever if telemetry ever shows it's needed.
- ⚠ **NON-BLOCKING** — whether the daily cap should reset at UTC midnight or the learner's local midnight is not specified anywhere; §5.2 proposes UTC for implementation simplicity, which is very unlikely to matter at this feature's stakes (a learner seeing their cap reset a few hours off their local midnight is not a meaningful UX problem).

## 15. Out of scope

- Prompt template text and where it lives → `AI_PROMPTS.md`, `AI_PROMPT_ARCHITECTURE.md`.
- Output validation that determines what's cacheable in the first place → `AI_GUARDRAILS.md`.
- The client-facing UX of a rate-limited or degraded call → `AI_ERROR_HANDLING.md`.
- `AIMemorySummary` computation (the mechanism that keeps prompts short in the first place) → `AI_CONTEXT_MODEL.md`.
- Firebase's general free-tier cost behavior (Auth/Firestore/Hosting/Functions, non-Gemini) → `docs/PRODUCT_SPEC.md` §31; not re-derived here since Gemini is "the real variable cost" per that section.

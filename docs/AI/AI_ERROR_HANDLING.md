# AI Error Handling

**Status:** Draft v1.0 — specification only, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §2, §10, §14, §20, §29; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9, §4.10
**Companion documents:** `docs/ENGINEERING/ERROR_HANDLING.md` (the project-wide error-code taxonomy; this document uses `GEMINI_UNAVAILABLE` and `RATE_LIMITED` by the names established there — reconciled during the documentation QC pass, see §6 below), `docs/AI/AI_GUARDRAILS.md` (defines when a response counts as "invalid" — the trigger this document maps to a fallback), `docs/AI/GEMINI_SPEC.md`, `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §9

## 1. Purpose

Map every way a Gemini call can fail to a specific, concrete, already-decided fallback — so "the learner is never blocked" (`docs/PRODUCT_SPEC.md` §2) is a checklist someone can verify against, not an aspiration re-derived per feature.

## 2. Scope

In scope: the failure-mode taxonomy for every Gemini-backed feature, the client-facing error codes, the per-feature fallback mapping, the error contract shape, and client-side handling guidance.

Out of scope: *how* a response is judged invalid (`AI_GUARDRAILS.md` §3–6 — this document treats "failed validation" as one input condition and starts from there), the project-wide error-code taxonomy beyond the AI-specific codes this document needs (`docs/ENGINEERING/ERROR_HANDLING.md`), rate-limit *numbers* and counter mechanics (`AI_COST_CONTROL.md` — this document uses the concept of a cap being exceeded, not its value).

## 3. The principle this document exists to make concrete

`docs/PRODUCT_SPEC.md` §2: **the learner is never blocked.** Stated for the AI layer specifically in §14: disabling the Gemini Function entirely must leave every other screen fully functional — this is an explicit MVP acceptance criterion (§29), not a nice-to-have. Every row in §5 below is a direct consequence of this one rule applied to one specific feature.

The sharpest version of the rule, worth stating once because it's easy to blur in implementation: **a Gemini failure must never be allowed to also become a scoring failure.** `getExerciseFeedback`'s deterministic re-scoring (`packages/music-engine`, run server-side) and its Gemini sub-call are two different operations with two different failure domains. A learner's exercise is scored, persisted, and reflected in their progress **regardless of whether the Gemini call inside the same Function invocation succeeds** — the two must never be allowed to fail together just because they happen to run in the same request.

## 4. Failure-mode taxonomy

| Failure mode | Where/how detected | Client-facing code |
|---|---|---|
| Gemini API unreachable (network error, 5xx, service outage) | `try`/`catch` around the API call | `GEMINI_UNAVAILABLE` |
| Gemini call exceeds its time budget (proposed: 8 seconds — no exact figure given in `docs/PRODUCT_SPEC.md`, chosen to sit comfortably above the "1–3 seconds acceptable" UX guidance in §22 while still failing fast rather than hanging) | A timeout wrapper around the call | `GEMINI_UNAVAILABLE` |
| Gemini rate-limits the app's own API key (Google-side quota, HTTP 429-equivalent) | Non-2xx-equivalent SDK response | `GEMINI_UNAVAILABLE` |
| The app's **own** per-user daily cap is already exhausted | A local counter check, performed *before* Gemini is ever called (`docs/ARCHITECTURE/DATA_FLOW.md` §5.2 step 5) | `RATE_LIMITED` (canonical name confirmed against `docs/ENGINEERING/ERROR_HANDLING.md` §4.3 — see §6) |
| Gemini responds, but the response fails validation — empty, over the length cap, malformed JSON for a structured template, or otherwise off-contract (`AI_GUARDRAILS.md` §3) | `validateOutput()` returns `null` | *(absorbed — see §5; surfaces as `GEMINI_UNAVAILABLE` only for Tutor Q&A/Theory Explanation)* |

`GEMINI_UNAVAILABLE` is deliberately one code covering four different underlying causes (down, slow, Google-rate-limited, or producing unusable output). The client does not need to distinguish these — in every case, the correct UI response is identical ("no usable AI response right now"). The Function's own logs *do* distinguish them (`AI_COST_CONTROL.md` §6's per-call log records the actual failure reason), because an operator investigating a spike in fallbacks needs to know whether Gemini itself is down versus the app's own validation logic just got stricter than it should have.

`RATE_LIMITED` is kept as a **separate** code, deliberately, even though it's also "no usable AI response right now" — because the correct thing to *tell the learner* is different. "The tutor is temporarily unavailable, try again shortly" is honest for `GEMINI_UNAVAILABLE`. It's dishonest for a learner who has simply asked more questions than today's cap allows — that learner should be told they've reached today's limit and it resets tomorrow, not encouraged to retry a call that will keep failing until midnight.

## 5. Per-feature fallback mapping

### 5.1 Exercise Feedback

**The learner still sees the full deterministic `ExerciseVerdict` — correct/missed notes, timing, composite score, pass/fail — exactly as if the Gemini call had succeeded, every single time, unconditionally.** Only the natural-language explanation is at risk, never the underlying result.

Every failure mode in §4 (including hitting the app's own daily cap) is absorbed **silently** inside `getExerciseFeedback`: `feedbackText` is replaced with the deterministic canned line from `AI_GUARDRAILS.md` §5, built purely from `verdict` fields, and the Function still returns `{ ok: true, data: { verdict, feedbackText } }`. The client never sees an error for this feature, never renders a "feedback unavailable" state, and never needs to. This is deliberate, not an oversight: Exercise Feedback's job (2–3 templated sentences about a specific attempt) is narrow enough that one well-written canned fallback is a genuinely adequate substitute for *any* attempt, so there is no reason to surface a distinguishable failure state to the client at all.

**Why the daily cap doesn't block this feature either:** unlike Tutor Q&A, an exhausted rate-limit counter here does not stop the exercise from being scored, saved, or reflected in progress — it only means this particular call skips straight to the canned `feedbackText`, exactly as if Gemini itself were down. Blocking exercise completion because a rate limiter denied the *explanatory prose* would violate §3's rule outright.

### 5.2 Tutor Q&A / Theory Explanation

Unlike Exercise Feedback, there is no single canned string that's an adequate stand-in for "the answer to whatever the learner just asked" — a generic line would read as a non-answer to a specific question. So this is the one feature where a failure is surfaced to the client explicitly, as `{ ok: false, error: { code, message } }`, and the client renders a **fundamentally different UI component**, not a chat bubble with degraded text.

**The fallback card, exact content:**

- **On `GEMINI_UNAVAILABLE`:** "The tutor's temporarily unavailable. Here's what to practice instead:" followed by a single deterministic suggestion — the learner's next incomplete lesson, read from `UserProfile.progressSummary` (client-readable, already subscribed to via the existing `onSnapshot` hook per `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §4.2 — no extra read required). This mirrors `docs/PRODUCT_SPEC.md` §10's exact language ("temporarily unavailable, here's what to practice instead") and §7's deterministic daily-plan rule (next incomplete lesson), reused here rather than inventing a third piece of fallback logic.
- **On `RATE_LIMITED`:** distinct copy — "You've used today's tutor questions. Come back tomorrow, or keep practicing in the meantime:" followed by the same next-lesson suggestion. Same underlying card component, different headline text, driven by which code came back.

**Resolved during the documentation QC pass:** `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8 previously stated `AIMemorySummary` was "never read... directly by the client," contradicting `docs/PRODUCT_SPEC.md` §19 and the shipped `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`, both of which grant owner-only client read on that path — `TYPES_AND_CONTRACTS.md` §4.8 has been corrected to match (`docs/DECISIONS.md` D-01). `AIMemorySummary` is now confirmed client-readable, owner-only; writes remain Cloud-Functions-only. This document's fallback card still uses only `ProgressSummary` for its MVP content (unambiguously client-readable either way, and already subscribed to), but a richer fallback ("review your recent misses on F and B", drawing on `AIMemorySummary.weakNotes`) is now a straightforward later enhancement rather than one blocked on an unresolved contradiction.

**Theory Explanation shares this exact fallback**, since it's a mode of the same `getTutorResponse` callable (`GEMINI_SPEC.md` §7) — a cache-miss that then fails to produce a valid live response returns the same `{ ok: false, error: { code: 'GEMINI_UNAVAILABLE' } }`, and nothing is ever written to the cache on a failure path (`AI_PROMPTS.md` §4.4), so a transient outage can never poison a cache entry.

### 5.3 Motivation / Nudge

Every failure mode in §4 falls back to the nearest pre-written pool entry (`AI_PROMPTS.md` §5.1), invisible to the learner as a "failure" — the pool is already the primary path for the large majority of cases, so a live-call failure just means one more case gets pool-served that would otherwise have gotten a slightly more tailored live line. Never surfaced as an error, never shown as a degraded state, never omitted.

### 5.4 Daily Practice Plan *(Phase 2 — not built for MVP)*

Not applicable to MVP. Once built: falls back to the same deterministic rule-based generator that ships in MVP as the *only* generator (next incomplete lesson + review last 3 misses, `docs/PRODUCT_SPEC.md` §7) — meaning this feature's fallback already has to exist and be solid before the Gemini-personalized version is even started, which is a useful ordering constraint for whoever picks up Phase 2, not just a fallback design note.

## 6. Error codes

| Code | Meaning | Source |
|---|---|---|
| `GEMINI_UNAVAILABLE` | No usable Gemini-backed response was produced, for any reason (down, timeout, Google-side rate limit, or failed validation on the Tutor Q&A/Theory Explanation path specifically) | Given directly in this document's brief; expected to be defined in `docs/ENGINEERING/ERROR_HANDLING.md`'s taxonomy |
| `RATE_LIMITED` | The app's own per-user daily cap was already exhausted before this call was attempted | Reconciled during the documentation QC pass against `docs/ENGINEERING/ERROR_HANDLING.md` §4.3, the project-wide taxonomy — that document is canonical, and this document's earlier proposed name (`RATE_LIMITED`) has been renamed to `RATE_LIMITED` throughout to match it exactly, closing the naming collision this row originally flagged as a risk |

Both codes travel inside the project-wide error contract (`docs/PRODUCT_SPEC.md` §20): `{ ok: false, error: { code: string, message: string } }`. `message` is a short, developer-facing string for logs/debugging — the client never renders `message` directly to the learner; it selects UI copy by switching on `code` (§5.2's two distinct card headlines are exactly this in practice).

## 7. Client handling guidance

- **Switch on `code`, never on `message` text.** `message` can change (log-message wording isn't a stable contract); `code` is the stable value the client's UI logic depends on.
- **A network-level failure to reach the callable Function at all** (no response, not even a `{ ok: false }`) is handled identically to receiving `GEMINI_UNAVAILABLE` — from the client's perspective these are the same situation ("no usable response came back"), so the Tutor screen's fallback-card logic should trigger on *either* condition, not only on an explicit error payload. This is what makes the fallback robust even if the Function itself is unreachable, not just the Gemini call inside it.
- **Never render a raw exception, stack trace, or the literal `error.message` string to the learner** — every failure path in §5 has specific, pre-written, warm copy; there is no code path where "show the technical error" is the right answer for this product (`docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §9: "a canned deterministic message, never a raw error dump or a blank screen").
- **Exercise Feedback needs no client-side error branch at all** (§5.1) — this is worth stating explicitly so a future implementer doesn't add unnecessary error-state UI for a feature that structurally never surfaces one.

## 8. Data structures

```typescript
// Project-wide contract, docs/PRODUCT_SPEC.md §20 — reproduced here, not owned here
type FunctionResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: string; message: string } };

// This document's AI-specific codes, used as the `code` value above
type AIErrorCode = 'GEMINI_UNAVAILABLE' | 'RATE_LIMITED';
```

## 9. Security considerations

- `message` must never leak implementation detail useful to an attacker (a stack trace, an internal Firestore path, a Gemini API error body verbatim) — it's a short, generic, developer-facing string; full detail belongs in server-side logs only (`AI_COST_CONTROL.md` §6), not in the response the client receives.
- A `RATE_LIMITED` response must be indistinguishable in timing/shape from a `GEMINI_UNAVAILABLE` one beyond the `code` field itself — no response-time side channel should let a script easily distinguish "you're rate-limited" from "the service is generally down" in a way that would help it time retries to evade the cap.

## 10. Testing requirements

- A forced-failure test per row in §4 (mocked network error, mocked timeout, mocked 429, mocked exhausted counter, mocked invalid response) for each of `getExerciseFeedback` and `getTutorResponse`, asserting the exact fallback behavior in §5 — including the specific assertion that `getExerciseFeedback` **always** returns `{ ok: true }` regardless of which Gemini-side failure is injected.
- `docs/PRODUCT_SPEC.md` §29's own acceptance criterion, made concrete: an integration test that disables the Gemini-calling path entirely (e.g., points the Function at an unreachable endpoint) and walks through the full MVP lesson sequence plus a tutor question, asserting every other screen — piano, lessons, dashboard, progress — is completely unaffected.
- A test asserting no user-facing string anywhere in the Tutor screen's fallback path contains a raw `error.message`, stack trace fragment, or Firestore path.

## 11. Acceptance criteria

Directly from `docs/PRODUCT_SPEC.md` §29, restated as this document's own checklist:

- [ ] Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly.
- [ ] A tutor response never states a note/chord/theory fact that contradicts the structured data it was given (enforced upstream by `AI_GUARDRAILS.md`; this document's job is ensuring a *failed* call never reaches the learner as if it had succeeded).
- [ ] Per-user rate limiting demonstrably blocks a scripted burst of tutor calls beyond the configured cap, and the learner sees the `RATE_LIMITED` card, not a generic error or a silent hang.
- [ ] Exercise Feedback failure never blocks, delays, or alters the `ExerciseVerdict` the learner sees.

## 12. Out of scope

- What makes a response "invalid" in the first place → `AI_GUARDRAILS.md`.
- The project-wide error-code taxonomy beyond the two AI-specific codes here → `docs/ENGINEERING/ERROR_HANDLING.md`.
- Rate-limit counter mechanics and the actual cap number → `AI_COST_CONTROL.md`.
- Non-AI degraded states (Firestore offline banner, etc.) → `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` §10, `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §9.

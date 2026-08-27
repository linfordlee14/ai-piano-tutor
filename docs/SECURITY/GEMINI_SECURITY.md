# Gemini Security

**Status:** Draft v1.0 — security documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §14, §15, §16, §19, §29; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9, §4.10, §5
**Companion documents:** `docs/SECURITY/SECURITY_REQUIREMENTS.md` (SEC-16, SEC-17, SEC-19, SEC-20, SEC-26), `docs/SECURITY/APP_CHECK.md`, `docs/SECURITY/SECRETS_MANAGEMENT.md`, `docs/SECURITY/AUTHENTICATION.md`, `docs/ARCHITECTURE/DATA_FLOW.md` §5; forward references to `docs/AI/AI_COST_CONTROL.md`, `docs/AI/AI_GUARDRAILS.md`, `docs/AI/AI_PROMPTS.md` (parallel batch, not yet written — own prompt/content-level concerns)

## 1. Purpose

Own the **infrastructure security** of every Cloud Function that calls Gemini — App Check enforcement, atomic rate limiting, the client-facing request-shape boundary, and prompt-injection containment at the infrastructure level. This is deliberately distinct from `docs/AI/` (not yet written), which owns prompt *content*: template wording, tone guardrails, the fact-invention prohibition, output-schema validation, and theory-answer caching. The dividing line: **if it's about what the Gemini call can reach or how often it can be made, it's this document; if it's about what the prompt says or what a good answer looks like, it's `docs/AI/`.**

## 2. Scope

In scope: `getTutorResponse` and `getExerciseFeedback` — the only two Cloud Functions that call Gemini (`docs/PRODUCT_SPEC.md` §20) — specifically their request/response boundary, their App Check and rate-limit gating, and the infrastructure-level prompt-injection containment (no tool/function-calling access).

Out of scope: prompt template wording (`docs/AI/AI_PROMPTS.md`), the exact rate-limit cap number and theory-answer cache mechanism (`docs/AI/AI_COST_CONTROL.md`), output-length caps and fact-invention guardrail wording (`docs/AI/AI_GUARDRAILS.md`), model tier selection rationale beyond its cost implication (`docs/ARCHITECTURE/AI_ARCHITECTURE.md`), retained conversation data (`docs/AI/AI_CONTEXT_MODEL.md` — cross-referenced from `docs/SECURITY/PRIVACY.md` instead, since that's a privacy question, not an infrastructure one).

## 3. Requirements

- App Check enforced on both Gemini-calling Functions, blocking non-app traffic before it can spend Gemini quota.
- A per-user rate-limit counter, checked and incremented **atomically**, inside the Function, before the Gemini call — enforcing a daily cap on tutor calls.
- The client-facing `getTutorResponse` signature accepts **only** `{ learnerQuestion: string }` — never a full `TutorRequest` object — so a learner cannot forge their own `AIMemorySummary`/`recentExerciseContext` to manipulate the tutor's answer.
- The client-facing `getExerciseFeedback` signature accepts only an `ExerciseAttempt` (raw `NoteEvent[]` + informational metadata) — never a pre-computed `ExerciseVerdict` — the same pattern applied to score integrity instead of tutor context.
- Every Gemini call is scoped to plain-text output only — no tool-calling or function-calling capability, ever.
- `learnerQuestion` is length-capped and validated server-side before it reaches prompt assembly.

Full requirement statements with IDs: `docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-16, SEC-17, SEC-19, SEC-20, SEC-26.

## 4. Architecture

### 4.1 Request pipeline for a Gemini-calling Function

Two layers, in strict order — a platform layer that runs before any handler code, and a handler layer that runs inside it:

**Platform layer** (Callable Functions framework, before handler code executes):
1. **App Check token validated.** A request with a missing or invalid App Check token never reaches handler logic at all — this is configured via `enforceAppCheck: true` on the function definition, not a manual check inside the handler (`docs/SECURITY/APP_CHECK.md`).
2. **Firebase Auth ID token verified**, `context.auth` populated (`docs/SECURITY/AUTHENTICATION.md` §4.5).

**Handler layer** (runs only if both platform checks pass), for `getTutorResponse` specifically — reproduced from `docs/ARCHITECTURE/DATA_FLOW.md` §5.1, restated here with the security rationale for each step:

3. Guard: reject if `context.auth` is null (defensive restatement of the platform check — see `docs/SECURITY/AUTHENTICATION.md` §4.5 for why this line still belongs in handler code even though the platform already enforces it).
4. Read `users/{uid}/aiMemory/summary` → `AIMemorySummary`, server-side, via the Admin SDK.
5. Read the most recent `users/{uid}/practiceSessions/{id}` (ordered by `completedAt desc`, using the composite index `docs/PRODUCT_SPEC.md` §17 already specifies) → `PracticeSession.verdict` or `null` if none exists yet.
6. Assemble the full `TutorRequest` server-side from steps 3–5 plus the client-supplied `learnerQuestion` — **this is the step that makes context forgery impossible**, detailed in §6.
7. Length-cap and validate `learnerQuestion`; check and atomically increment the per-user rate-limit counter — **before** step 8, not after (§4.3).
8. Render the Tutor Q&A prompt template (`docs/AI/AI_PROMPTS.md`) from the assembled `TutorRequest` and call Gemini, using the Secret-Manager-sourced credential (`docs/SECURITY/SECRETS_MANAGEMENT.md`), with no tools/functions registered (§4.4).
9. Validate the raw response against the `TutorResponse` output contract (`docs/AI/AI_GUARDRAILS.md`) — length cap, well-formed; fall back to a deterministic canned message on failure.
10. Return the typed `{ ok: true, data: TutorResponse }` / `{ ok: false, error }` shape (`docs/PRODUCT_SPEC.md` §20).

`getExerciseFeedback` follows the identical platform-layer gating (steps 1–2); its handler-layer specifics (re-scoring via `packages/music-engine`, writing `PracticeSession`/`ProgressSummary`, then its own Gemini call for the Exercise Feedback template) are traced in full in `docs/ARCHITECTURE/DATA_FLOW.md` §4 — this document's contribution to that flow is exclusively the App Check/rate-limit/injection-containment layer, identical in kind to what's detailed here for the tutor path.

### 4.2 Why App Check matters specifically here, distinct from Firestore

`docs/SECURITY/APP_CHECK.md` covers the general mechanism. Stated specifically for this document: App Check on the Gemini-calling Functions is as much a **cost control** as a security control. Every request that reaches Gemini costs money (`docs/PRODUCT_SPEC.md` §31); App Check rejects non-app traffic (a bot, a scraped endpoint, a script replaying a captured request) before that traffic can trigger a single token of Gemini spend, not after. This is why `docs/PRODUCT_SPEC.md` §16 singles out "the Gemini-calling Functions" alongside Firestore rather than treating App Check as a single undifferentiated abuse-prevention line item.

### 4.3 Why the rate-limit check must be atomic, and must run before the Gemini call

"Checked and incremented atomically" (`docs/PRODUCT_SPEC.md` §19) means a Firestore **transaction** — read the current count, compare it to the cap, and write the incremented value, all as one operation the Firestore SDK guarantees is applied consistently even under concurrent requests. A bare `FieldValue.increment(1)` is atomic at the single-field level but does not, by itself, reject a request that pushes the count over the cap — it would happily increment past any limit. And a naive read-then-write (read count, check in application code, write count+1 as a separate operation) is racy: two concurrent requests can both read count=9 under a cap of 10, both decide "under cap," and both proceed, breaching the cap by one. A transaction closes exactly this race.

Equally important: this check must complete, and pass, **before** the Gemini call is made — not logged/decremented after the fact. `docs/ARCHITECTURE/DATA_FLOW.md` §5.2 states this explicitly ("Gemini is never called before the rate-limit check in step 5 passes — cost and abuse protection happens strictly before the expensive call, not after"). A burst of concurrent requests arriving faster than the transaction can serialize them will have every request past the first N rejected *before* touching Gemini, which is the entire cost-control point of the requirement — checking after the call would let the burst spend the quota, then merely record that it shouldn't have.

**Proposed counter shape** (not specified by `docs/PRODUCT_SPEC.md`/`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — **PROPOSED**, would need to be added to both if adopted):

| Field | Type | Notes |
|---|---|---|
| `date` | `string` (ISO date, `YYYY-MM-DD`) | Calendar-day reset, matching the existing convention `UserProfile.streak.lastPracticeDate` already uses (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15) — reset-on-read: if the stored `date` isn't today, treat the count as 0 and overwrite `date` as part of the same transaction, rather than running a separate scheduled reset job. |
| `count` | `number`, integer ≥0 | Calls made so far today. |

Proposed location: a Function-written-only document, e.g. `users/{uid}/rateLimits/tutorCalls` (or a field on the same document — either is compatible with the transaction pattern above; the exact placement is an implementation choice for `functions/src/ai/`, not a security-relevant decision). Whichever location is chosen, it inherits `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`'s default-deny posture automatically via the catch-all rule unless a new explicit block is added — and no explicit client-read/write block should be added for it, since there is no legitimate client reason to read or write this counter directly.

The exact **cap number** is out of scope for this document — owned by `docs/AI/AI_COST_CONTROL.md` (itself an open question there, per the parallel batch producing it). This document specifies the mechanism the number plugs into, not the number.

### 4.4 Prompt injection containment — the infrastructure half

`docs/PRODUCT_SPEC.md` §19: *"a learner typing 'ignore your instructions and…' into the tutor chat shouldn't be able to extract the system prompt or trigger anything beyond text generation — the Function's Gemini call is scoped to plain text output only, with no tool- or function-calling access to anything sensitive."*

This document owns the **infrastructure** half of that sentence: the Gemini client configuration used by both `getTutorResponse` and `getExerciseFeedback` registers **zero** tools/functions. Even in the hypothetical worst case where a prompt-injection attempt fully succeeds at making the model *want* to do something beyond generate text — call an external system, read arbitrary data, take an action — there is no tool surface available for it to call. This is a containment property of the calling infrastructure, independent of and in addition to any content-level defense (system-instruction wording, input sanitization heuristics) that `docs/AI/AI_GUARDRAILS.md` owns. The two layers are complementary, not redundant: content-level defenses reduce how often an injection attempt "succeeds" at derailing the model's output; infrastructure containment bounds the *damage* even when one does.

## 5. Data structures

No new persisted types beyond the proposed rate-limit counter in §4.3 (PROPOSED, not yet part of `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`). References `TutorRequest`/`TutorResponse` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9/§4.10) and `ExerciseAttempt`/`ExerciseVerdict` (§4.4/§4.5) without redefining them.

## 6. Interfaces/contracts

### 6.1 `getTutorResponse` — client can supply almost none of `TutorRequest`

| `TutorRequest` field | Client can supply it directly? | Actual source |
|---|---|---|
| `learnerQuestion` | **Yes** — the only client-supplied field | Request payload; length-capped and validated server-side (§4.1 step 7) before use |
| `recentExerciseContext` | No | Server reads the caller's own most recent `PracticeSession` from Firestore, via the Admin SDK, using `context.auth.uid` |
| `learnerMemorySummary` | No | Server reads `users/{uid}/aiMemory/summary`, via the Admin SDK |
| `skillLevel` | No | Server reads it off the `learnerMemorySummary` just loaded |

`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9, verbatim: *"the client-facing Cloud Function signature accepts only `{ learnerQuestion: string }`. `TutorRequest` as a whole is constructed inside the Function, never accepted as a full object from the client."* **This closes a specific, concrete attack:** even a fully client-controlled request body — a modified client sending arbitrary extra JSON fields alongside `learnerQuestion`, hoping the Function naively merges the payload into `TutorRequest` — cannot succeed, because the Function's construction of `TutorRequest` never reads those fields from the request at all; `recentExerciseContext` and `learnerMemorySummary` are sourced exclusively from Firestore documents the client cannot write (`docs/SECURITY/FIRESTORE_SECURITY_RULES.md` §5.4). A learner cannot claim to be more advanced than they are, cannot inject a fabricated "recent mistake" to steer the tutor's framing, and cannot smuggle instructions into what the tutor believes about their own performance — because none of that ever comes from them.

### 6.2 `getExerciseFeedback` — the same pattern, applied to score integrity

The structurally identical boundary, applied to the other Gemini-calling Function: it accepts an `ExerciseAttempt` (raw `NoteEvent[]` plus informational-only `lessonId`/`exerciseId`/`startedAt`/`submittedAt`/`deviceType`/`attemptNumber`) and **never** accepts a pre-computed `ExerciseVerdict`, `accuracy`, or `compositeScore` field from the client (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4, §4.5). This is the same underlying pattern as §6.1 — never trust a client-supplied *derived/summarized* value, only raw input, always recompute server-side — applied to two different Functions for two different reasons (context-fabrication resistance for the tutor; score-tampering resistance for exercise feedback). Full detail on this specific boundary: `docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-10/SEC-11, `docs/ARCHITECTURE/DATA_FLOW.md` §4.

## 7. Dependencies

- `docs/SECURITY/APP_CHECK.md` — the mechanism behind §4.1 step 1.
- `docs/SECURITY/AUTHENTICATION.md` — the mechanism behind §4.1 step 2/3.
- `docs/SECURITY/SECRETS_MANAGEMENT.md` — the credential used in §4.1 step 8.
- `docs/AI/AI_COST_CONTROL.md` (parallel batch) — the rate-limit cap number.
- `docs/AI/AI_GUARDRAILS.md` (parallel batch) — content-level prompt-injection defenses and output validation specifics.
- `docs/ARCHITECTURE/DATA_FLOW.md` §5 — the full end-to-end trace this document's security layer sits inside.

## 8. Security considerations

Restated as a single table for scannability — every control, why it exists, and what it specifically prevents:

| Control | Prevents |
|---|---|
| App Check (§4.2) | Non-app traffic (bots, scrapers, replayed requests) from spending Gemini quota at all |
| Auth token verification (§4.1 step 2/3) | An unauthenticated or spoofed-identity caller from reaching a per-user resource |
| Atomic rate limiting (§4.3) | A single compromised or scripted account from generating unbounded Gemini spend |
| `{ learnerQuestion }`-only signature (§6.1) | A learner fabricating their own performance context to manipulate the tutor's framing or answer |
| Raw-`ExerciseAttempt`-only signature (§6.2) | A learner submitting a forged score directly |
| No tool/function-calling access (§4.4) | A successful prompt injection from reaching anything beyond text generation |
| Server-side length cap on `learnerQuestion` (§4.1 step 7) | An oversized or malformed input bypassing a client-side-only cap |

## 9. Performance considerations

App Check and auth verification happen in the platform layer before any Gemini spend, so a rejected request costs effectively zero Gemini tokens. The rate-limit transaction (§4.3) adds a small amount of latency — a Firestore transaction round trip — before the Gemini call proper begins; this is an acceptable cost given `docs/PRODUCT_SPEC.md` §22 already treats 1–3 seconds of total Gemini-call latency as fine for these explicitly user-initiated calls (never per-keystroke, never polled — `docs/PRODUCT_SPEC.md` §14), so this is not a place to trade away correctness (atomicity) for a marginal latency win.

## 10. Error handling

Every rejection reason maps to a distinct typed error code, feeding the `{ ok: false, error: { code, message } }` contract (`docs/PRODUCT_SPEC.md` §20) and, on the client, a specific fallback — never a raw/opaque failure:

| Rejection reason | Where caught | Client-facing fallback |
|---|---|---|
| App Check token missing/invalid | Platform layer, before handler runs | Request never reaches the tutor UI's normal flow; treated the same as "Gemini unavailable" (`docs/PRODUCT_SPEC.md` §10's fallback card) |
| Auth token missing/invalid | Platform layer | Same as above — an unauthenticated caller shouldn't be reaching this Function's UI entry point in the first place |
| Rate limit exceeded | Handler, step 7, before Gemini call | A specific "you've reached today's limit" message — distinct from a generic outage message, since the cause is different and the learner can act on it (come back tomorrow) differently than they'd act on an outage |
| `learnerQuestion` invalid/oversized | Handler, step 7 | Client-side validation should already prevent this in the normal UI; server-side rejection here is a defensive backstop, surfaced as a generic input-error message |
| Gemini call fails or returns an invalid response | Handler, step 9 | Deterministic canned fallback message (`docs/PRODUCT_SPEC.md` §15) — never a broken screen |

## 11. Testing requirements

- Emulator test: a request to either Gemini-calling Function without a valid App Check token is rejected before any handler logic (and, critically, before any Gemini call is attempted) — assert on call count to a mocked Gemini client, not just on the HTTP-level rejection.
- Scripted-burst test: a rapid sequence of `getTutorResponse` calls from the same test identity, exceeding the configured cap, results in the excess calls being rejected with the rate-limit error code — and, again, assert the mocked Gemini client was never invoked for the rejected calls, not just that an error was returned.
- Payload-injection test: a `getTutorResponse` call whose request body includes extra fields shaped like `recentExerciseContext`/`learnerMemorySummary`/`skillLevel` is confirmed to have those fields **ignored** — the `TutorRequest` actually assembled and sent to the prompt template must be asserted to match Firestore state, not the forged payload.
- Contract test (per `docs/PRODUCT_SPEC.md` §26): the Gemini client configuration used in tests has no tools/functions registered — a structural assertion on the client setup, not a behavioral one, since a "does the model ever call a tool" test would be non-deterministic and expensive to rely on.
- Disabling test: with the Gemini-calling Functions entirely disabled/stubbed to always fail, every other screen in the app remains fully functional (this is primarily a `docs/ARCHITECTURE/` / whole-app test, restated here because it's also this document's own acceptance criterion below).

## 12. Acceptance criteria

Verbatim, `docs/PRODUCT_SPEC.md` §29:
- "Per-user rate limiting demonstrably blocks a scripted burst of tutor calls beyond the configured cap"
- "Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly, not assumed"
- "A tutor response never states a note/chord/theory fact that contradicts the structured data it was given" (this document's contribution: the structured data itself — `recentExerciseContext`, `learnerMemorySummary` — is guaranteed server-authoritative per §6.1, which is a precondition for this criterion to even be meaningful; the prose-level fidelity check itself is `docs/AI/AI_GUARDRAILS.md`'s to satisfy)

## 13. Out of scope

- Prompt template wording and tone — `docs/AI/AI_PROMPTS.md`.
- The exact rate-limit cap number, and whether it differs by any future tier — `docs/AI/AI_COST_CONTROL.md`. **Minor flag:** `docs/PRODUCT_SPEC.md` §19's exact phrase is "a daily cap on tutor calls for free-tier users" — the only place in the entire spec that implies a tiered/paid model might exist; no pricing, billing, or paid-tier concept is defined anywhere else in `docs/PRODUCT_SPEC.md`. This document treats the rate limit as applying uniformly to all users (anonymous and authenticated alike), since MVP has exactly one tier — "free" is simply the only tier that currently exists, not a hint that free vs. paid logic should be built now. Classification: NON-BLOCKING — resolves cleanly under "the only tier that exists," worth a note in case `docs/AI/AI_COST_CONTROL.md` reads the same phrase and wonders whether to design tier-differentiated logic prematurely.
- Model tier selection (`cheapest` vs. `mid`) — a cost/quality trade-off, `docs/ARCHITECTURE/AI_ARCHITECTURE.md`, not a security boundary.
- Retained tutor-conversation data and its privacy treatment — `docs/SECURITY/PRIVACY.md`, cross-referencing `docs/AI/AI_CONTEXT_MODEL.md`.

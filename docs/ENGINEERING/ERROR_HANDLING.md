# Error Handling

## 1. Purpose

The canonical error contract for AI Piano Tutor, and the concrete answer to "what does the learner actually see" for every failure mode PRODUCT_SPEC.md names. Two things anchor this document:

1. **The error shape** — `docs/PRODUCT_SPEC.md` §20: every Cloud Function returns `{ ok: true, data }` or `{ ok: false, error: { code, message } }`, never a raw exception the client has to interpret.
2. **The "never a dead end" principle** — stated as the app's design principle in `docs/PRODUCT_SPEC.md` §2 ("the learner is never blocked... If Gemini is down, the piano still works. If Firestore is down, the piano still works. If MIDI isn't available, the piano still works") and restated as concrete UI behavior in §10 (the degraded-service banner, the Tutor screen's fallback card) and §23 (offline strategy). This document is what turns that principle into a specific, testable mapping from *failure* to *UI state*, so no failure mode is ever handled by "throw an error and let the screen go blank."

⚠ A note on section numbering, so this document's citations don't send a reader looking for something that isn't there: this task's own brief referenced a "section 42/43-style 'never a dead end' principle." `docs/PRODUCT_SPEC.md` only has 35 sections — it references "the brief's section 43" and "section 37" itself, several times, but those citations point at the *original 44-section brief* PRODUCT_SPEC.md is a response to, which isn't part of this repository. Every citation in this document uses PRODUCT_SPEC.md's own section numbers, where the principle is fully restated inline (§2, §10, §20, §23) — not the original brief's numbering.

## 2. Scope

Covers: the Cloud Functions error-return contract and code taxonomy; the client-side pattern for consuming it without every call site re-implementing error handling; and the three specific degraded-mode scenarios PRODUCT_SPEC.md names by name — Gemini unavailable, Firestore unreachable, MIDI unavailable.

Out of scope: Firestore Security Rules denial behavior (`docs/SECURITY/FIRESTORE_SECURITY_RULES.md`), Gemini prompt-level guardrails and response validation specifics (`docs/AI/AI_GUARDRAILS.md`), and retry/backoff tuning for performance reasons rather than error-semantics reasons (`docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md`, `docs/TESTING/PERFORMANCE_TESTING.md`).

## 3. Requirements

**R1.** Every one of the three named degraded-mode scenarios (§5) must leave the piano and the currently-loaded lesson fully playable. This is not a per-scenario judgment call — it's the direct, non-negotiable consequence of `docs/PRODUCT_SPEC.md` §2's design principle and is already a stated acceptance criterion (§29: "Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly, not assumed").

**R2.** No error state is ever a bare error toast, a raw exception message, or a blank screen with no next action. Every entry in the taxonomy (§4) maps to a specific, named UI state (§4's table) — if a new error code is ever added that doesn't have an obvious UI mapping, that's a sign the code shouldn't exist yet, not a reason to ship it without one.

**R3.** No automatic, silent retry of a Gemini call. `docs/PRODUCT_SPEC.md` §31 names "a retry loop that fires Gemini calls in a tight loop" directly as a cost-risk to watch for, and §14's call-architecture rule is that Gemini is called "only on explicit learner action... never polled, never called per keystroke, never called 'just in case.'" A retry after `GEMINI_UNAVAILABLE` or `RATE_LIMITED` requires a new, explicit learner action (tapping "try again," asking a new question) — never an automatic background retry.

## 4. The Cloud Functions Error Contract

### 4.1 Shape

```typescript
// Proposed shape — see the open question below on why this isn't cited as
// an existing packages/shared-types export yet.
type FunctionResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: FunctionErrorCode; message: string } };
```

**Resolved during the documentation QC pass:** this wrapper type is now formally defined at `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.16, exported from `packages/shared-types`, using this document's `FunctionResult<T>` / `FunctionErrorCode` naming verbatim as the canonical source (see that section's note and `docs/DECISIONS.md` D-04). `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` §6 and `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §6/§9 each still restate the shape inline for readability in context — that's fine, since all three now point at the one shared definition here rather than independently inventing it.

### 4.2 Throw vs. return — how this is actually implementable with Firebase callable Functions

Firebase callable functions have two distinct ways to signal failure to the client SDK, and getting this distinction right is what makes §20's "every function returns a typed shape" buildable rather than just a diagram-level promise:

- **Application-level (designed, expected path):** the Function's own code catches every anticipated failure — auth check fails, input fails validation, Gemini is unreachable, the rate limit is hit — and *returns* (does not throw) `{ ok: false, error: { code, message } }` as a normal, successfully-resolved response. `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` §10 states this exact pattern for the auth-check case: "a failed check returns the `{ ok: false, error }` shape... not a raw 401 the client has to interpret." This is the path every named error code in §4.3 below is designed to travel.
- **Platform-level (residual, unavoidable):** a genuinely unrecoverable failure — the request never reaches the Function (network failure), the Function times out, or code crashes before any handler logic runs — still produces a rejected promise on the client, no matter how disciplined the Function's own code is. This is a physical reality of any RPC-style call over a network, not a design choice, and it cannot be fully eliminated by writing better Function code.

**The rule this produces:** a Cloud Function handler should catch as much as it possibly can and convert it into the application-level `{ ok: false, error }` return — that's the entire point of the taxonomy in §4.3. But the client-side calling code (§4.4) must still defend against the residual platform-level rejection, because no amount of care inside the Function eliminates it. The two are not parallel "layers" a developer chooses between; the application-level path is the designed, complete-coverage path, and the platform-level path is the unavoidable escape hatch underneath it that gets handled once, in one place, rather than defensively at every call site.

### 4.3 Error code taxonomy

| Code | Meaning | When it's returned | Source |
|---|---|---|---|
| `UNAUTHENTICATED` | No valid Firebase ID token on the request, or the token failed the Admin SDK's verification | Every callable Function checks `context.auth` before doing anything user-specific (§19); this is what a failed check returns | §19 |
| `VALIDATION_FAILED` | The request payload doesn't match its expected shape, or fails a length/content check | Malformed `ExerciseAttempt` (e.g. out-of-range `midiNote`, per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1's validation rule), or a `learnerQuestion` that fails the length cap (§19: "input validation/length caps on the learner's free-text question before it's dropped into a prompt") | §19, TYPES_AND_CONTRACTS §4.1/§4.4 |
| `RATE_LIMITED` | The caller has exceeded their per-user daily cap on tutor calls | The atomically-incremented rate-limit counter inside the Function (§19) rejects the request | §19, §29 |
| `GEMINI_UNAVAILABLE` | The Gemini API call itself failed, timed out, or returned an unusable response | Any Gemini-calling Function (`getTutorResponse`, and the feedback-prose portion of `getExerciseFeedback`) when the upstream call doesn't succeed | §10, §14 |
| `NOT_FOUND` | A referenced `lessonId`/`exerciseId` in a submitted `ExerciseAttempt` doesn't match any known `Lesson`/`Exercise` definition | The server-side re-validation step (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4) independently loads the definition it's scoring against — if that load fails, this is the code | TYPES_AND_CONTRACTS §4.4 |
| `INTERNAL` | Any other unanticipated server-side failure the handler's own try/catch still managed to convert into a response | Catch-all — should be rare, and every occurrence is worth investigating, not just displaying | §20 |

This set deliberately stays close to the five examples this task named (`UNAUTHENTICATED`, `RATE_LIMITED`, `VALIDATION_FAILED`, `GEMINI_UNAVAILABLE`, `INTERNAL`), plus `NOT_FOUND`, which is directly grounded in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4's server-side re-validation requirement. It does not include codes with no grounding in the spec (e.g. `CONFLICT`, `ALREADY_EXISTS`) — add one only when a real scenario needs it, per `docs/ENGINEERING/DEPENDENCY_POLICY.md`'s "every addition needs a stated reason" principle applied to a taxonomy instead of a package list.

### 4.4 Client-side consumption pattern

Per `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §6, every callable Function has exactly one thin client-side wrapper (`apps/web/lib/functions/`, `docs/ENGINEERING/PROJECT_STRUCTURE.md` §5), which "return[s] the `{ ok, data | error } `shape untouched so every call site handles errors the same way." Making that true for *every* call, including a rejected one, is this wrapper's job — not something each call site re-implements:

```typescript
// apps/web/lib/functions/getExerciseFeedback.ts — illustrative, not final implementation
async function getExerciseFeedback(attempt: ExerciseAttempt): Promise<FunctionResult<ExerciseVerdict>> {
  try {
    const result = await httpsCallable(functions, 'getExerciseFeedback')(attempt);
    return result.data as FunctionResult<ExerciseVerdict>; // the Function's own {ok,...} response
  } catch (err) {
    // Platform-level rejection (§4.2) — never surfaced to the caller as a throw.
    return { ok: false, error: { code: 'INTERNAL', message: 'Could not reach the server.' } };
  }
}
```

Every call site above this wrapper receives a `FunctionResult<T>` and only ever branches on `result.ok` — never a `try/catch` around the call itself, and never a raw exception reaching UI code. This is what makes R2 (§3) achievable in practice rather than a rule each screen has to remember independently.

## 5. The Three Named "Never a Dead End" Scenarios

Two of these three are **not** Cloud Function errors at all — they're a different mechanism, and conflating them with the `FunctionErrorCode` taxonomy above would be a mistake worth heading off explicitly.

### 5.1 Gemini unavailable → deterministic fallback

Routed through the Cloud Function taxonomy: `GEMINI_UNAVAILABLE` (or `RATE_LIMITED` — see the UI-mapping table in §6, where these two intentionally share a UI state). The learner never needs to know or care *why* the tutor isn't answering right now; the AI Tutor screen shows the exact fallback `docs/PRODUCT_SPEC.md` §10 specifies: "a clear 'temporarily unavailable, here's what to practice instead' fallback card built from deterministic data — never a dead end." This is directly the behavior `docs/PRODUCT_SPEC.md` §29's acceptance criterion tests: "Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly, not assumed."

### 5.2 Firestore unreachable → offline persistence + degraded banner

**Not a `FunctionErrorCode` at all.** Direct client reads/writes to Firestore (the dashboard's `onSnapshot` subscriptions, per `docs/PRODUCT_SPEC.md` §20: "the Next.js client talks to Firestore directly... for reads") never go through a Cloud Function, so they can't return the `{ ok, error }` shape — connectivity state here is detected via the Firestore SDK's own mechanisms (its offline persistence layer, `onSnapshot`'s `metadata.fromCache`/connection state), not returned as an error code.

Mechanism, per `docs/PRODUCT_SPEC.md` §23: enable Firestore's built-in offline persistence (`enableIndexedDbPersistence`). Reads serve from the local cache; writes queue automatically and flush on reconnect — §23 is explicit that this is "closer to 'flip a configuration flag' than 'build a sync engine.'" The client surfaces this as the persistent, non-alarming banner §10 specifies verbatim: **"Playing offline — your progress will sync later."** The banner is a shell-level concern (rendered once in the root layout, per `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §9), not duplicated per screen. The piano and the currently-loaded lesson keep working locally throughout, because the Engine Core has zero network dependency by design (§11) — this scenario never touches R1 (§3) at all, by construction.

### 5.3 MIDI unavailable → clear message, not silent failure

⚠ **Scope note:** MIDI input itself is explicitly **Phase 2**, not MVP (`docs/PRODUCT_SPEC.md` §6, §7, §8 all defer it). It's documented here — tagged Phase 2 throughout — because `docs/PRODUCT_SPEC.md` §11 already specifies the feature-detection/fallback pattern as forward-looking architecture, and because a consistent error-handling reference should cover it alongside the other two "never a dead end" examples rather than have it live nowhere until Phase 2 begins. Nothing in this subsection implies MIDI work belongs in the MVP milestone set (`docs/PRODUCT_SPEC.md` §28).

**Not a `FunctionErrorCode` either** — a pure client-side browser-capability check. Per §11: feature-detect with `'requestMIDIAccess' in navigator` before ever exposing MIDI-related UI. §11's verified browser gap is permanent, not transient: "Safari on both macOS and iOS does not support the Web MIDI API and Apple has not committed to adding it" — so on a large share of this app's mobile-first user base, this isn't an occasional failure to handle gracefully, it's the expected steady state. If a learner reaches a MIDI-connect affordance on an unsupported browser, the required behavior is a clear "MIDI isn't available in this browser" message — never silently doing nothing when tapped.

## 6. Error Code → UI State Mapping

| Code / scenario | UI state | Learner-facing message pattern | Notes |
|---|---|---|---|
| `UNAUTHENTICATED` | Silent re-auth attempt (anonymous session refresh) first; if that fails, route to Sign In (§10) | No message if silently recovered; otherwise the normal Sign In screen, not an error page | Anonymous users carry a real Firebase uid (§19) — a token issue is usually transient |
| `VALIDATION_FAILED` | Generic retry affordance | "Something went wrong — try again." | Should not happen from a well-behaved client; this is not a teaching moment, so no detailed message. Log the code + timestamp for engineering, never the learner's raw input (§25 privacy boundary) |
| `RATE_LIMITED` | Same deterministic fallback card as `GEMINI_UNAVAILABLE` | "You've used today's tutor questions — here's what to practice instead," built from `AIMemorySummary` | Non-punitive framing — this is a cost guardrail, not something the learner did wrong |
| `GEMINI_UNAVAILABLE` | Deterministic fallback card (§5.1) | "Tutor's temporarily unavailable — here's what to practice instead" (§10's exact language) | Same card as `RATE_LIMITED` on purpose — the learner doesn't need to distinguish the cause |
| `NOT_FOUND` | Generic retry / refresh prompt | "Something went wrong — try again." | Indicates a stale client reference (e.g. a removed lesson) or a bug; log for engineering |
| `INTERNAL` | Generic retry affordance, escalating to the degraded-banner pattern if repeated | "Something went wrong — try again." | If this repeats across multiple actions, treat it like the offline banner (§5.2) rather than repeating a per-action retry prompt indefinitely |
| Firestore unreachable (§5.2, not a Function code) | Persistent, non-alarming shell banner | "Playing offline — your progress will sync later." (§10's exact language) | Piano and current lesson unaffected; writes queue via `enableIndexedDbPersistence` |
| MIDI unavailable (§5.3, not a Function code, **Phase 2**) | Clear inline message, MIDI UI hidden or disabled | "MIDI isn't available in this browser." | Feature-detected before any MIDI UI is shown at all — this message is the fallback for the rare case a learner still finds their way to a MIDI-related affordance |

## 7. Error Handling

The general rule underlying every row above: **an error code without an obvious row in this table is not ready to ship** (R2, §3). Adding a new `FunctionErrorCode` and a new UI state are the same PR, never sequential.

Retry behavior (R3, §3): `RATE_LIMITED` and `GEMINI_UNAVAILABLE` never trigger an automatic retry. The fallback card is a complete, satisfying end state on its own — not a spinner waiting to try again — because the retry has to come from a new explicit learner action, per `docs/PRODUCT_SPEC.md` §14's call-architecture rule.

Logging: `GEMINI_UNAVAILABLE` and `RATE_LIMITED` occurrences are worth logging with a token/call count specifically because `docs/PRODUCT_SPEC.md` §31 recommends instrumenting Gemini spend "from day one... a simple per-call log with token counts, not just a monthly bill review," and a spike in either code is the earliest signal that one of the cost guardrails (§19, §31) has regressed. Full cost-monitoring ownership: `docs/AI/AI_COST_CONTROL.md`; this document only notes that these two codes are the trigger worth watching.

## 8. Testing Requirements

- A test that mocks/disables the Gemini-calling Function entirely and asserts every other screen remains fully functional — this is `docs/PRODUCT_SPEC.md` §29's acceptance criterion, and it's this document's mapping table (§6) that the test is actually verifying.
- A test that a scripted burst of tutor calls beyond the configured per-user cap returns `RATE_LIMITED` and that the client renders the fallback card, not a generic error (§29).
- Contract tests per `docs/PRODUCT_SPEC.md` §26: does a Gemini response fit its schema, respect its length cap, and — specifically relevant here — "does the fallback trigger correctly on an error or malformed response." This is exactly this document's `GEMINI_UNAVAILABLE` path, exercised with a deliberately malformed upstream response rather than a hard failure.
- A test asserting the client-side function wrapper (§4.4) converts a rejected `httpsCallable` promise into a resolved `{ ok: false, error: { code: 'INTERNAL', ... } }` value — i.e., that no call site above the wrapper can ever receive a thrown exception.
- A Firestore-offline test: simulate a disconnected client, assert the degraded banner appears and the current lesson remains fully playable, then simulate reconnect and assert queued writes flush (§23).

## 9. Acceptance Criteria

- [ ] Disabling the Gemini Function entirely leaves every other screen fully functional (`docs/PRODUCT_SPEC.md` §29, verified by test, not assumed).
- [ ] Per-user rate limiting demonstrably blocks a scripted burst of tutor calls beyond the configured cap, and the client shows the fallback card rather than a raw error (`docs/PRODUCT_SPEC.md` §29).
- [ ] Every `FunctionErrorCode` in §4.3 has a corresponding row in the §6 mapping table — no code ships without a defined UI state.
- [ ] No screen in the app can reach a state with no visible next action (a retry, a fallback, a navigation) after any of the failure modes in §5 or §6.
- [ ] A client-side rejected `httpsCallable` call never propagates as an uncaught exception past the function wrapper layer (§4.4).

## 10. Out of Scope

- Firestore Security Rules denial syntax and behavior — `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.
- Gemini prompt-level guardrails (length caps, injection hygiene, structured-output validation detail) — `docs/AI/AI_GUARDRAILS.md`.
- Gemini spend monitoring and alerting mechanics — `docs/AI/AI_COST_CONTROL.md`.
- Performance-driven retry/backoff tuning (as distinct from the error-semantics "never auto-retry Gemini" rule in §7) — `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md`.

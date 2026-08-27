# Integration Testing — Cloud Functions & Firebase Emulator Suite

## 1. Purpose

Defines integration testing for everything that only becomes meaningful once Cloud Functions, Firestore, and Auth run together as a system: the write-only-via-Function pattern, server-side re-validation of learner attempts, and full-subtree cascading operations (account deletion, data export). These tests run against the **Firebase Local Emulator Suite**, never production data.

**Source of truth:** `docs/PRODUCT_SPEC.md` §16 (Firebase Architecture), §18 (Authentication — account deletion/export), §19 (Security Architecture), §20 (API Architecture — callable functions and error contract), §26 (testing strategy), §29 (acceptance criteria). Contracts: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4–4.8, §4.15, §5 (cross-cutting rule 1: server-written-only fields are enforced *redundantly* by both rules and Function logic — this document tests both halves of that redundancy, not just one).

## 2. Scope

Covers Cloud Functions tested end-to-end against the emulator suite: `getExerciseFeedback`, `getTutorResponse` (contract-shape only here — response *content* correctness is `AI_TESTING.md`'s job), `deleteAccount`, `exportUserData`, and the Firestore Security Rules those functions and their surrounding client code interact with. Does not cover: pure `packages/music-engine` logic in isolation (`UNIT_TESTING.md`), Gemini prompt/response contract correctness (`AI_TESTING.md`), or full browser-driven user journeys (`E2E_TESTING.md`) — though the reference E2E scenario (`docs/TESTING/TESTING_STRATEGY.md` §4) and this suite's `getExerciseFeedback` tests cover overlapping ground from different layers, and that overlap is intentional defense-in-depth, not redundant waste.

## 3. Environment

- **Firebase Local Emulator Suite:** Firestore, Functions, and Auth emulators running together, started via the project's `firebase emulators:start` (or CI equivalent), configured in `firebase.json` (`docs/PRODUCT_SPEC.md` §27).
- **No production data, ever.** Every test seeds its own emulator state from fixtures at the start of the test (or suite) and tears it down after. A test that reads or depends on data left over from a previous run is a bug in the test, not a valid pattern.
- **Gemini calls are stubbed/mocked** in every integration test that exercises `getTutorResponse` or `getExerciseFeedback`'s AI-explanation path — this suite tests the *plumbing* (auth check → scoring → Firestore write → response shape), not Gemini's actual output, which is `AI_TESTING.md`'s job. Mocking the Gemini client at the boundary keeps these tests fast and deterministic.
- **Auth emulator:** used to create real (emulated) anonymous and email/password test users with real Firebase ID tokens, so tests exercise the actual `context.auth` verification path (`docs/PRODUCT_SPEC.md` §19) rather than a stubbed auth object.

## 4. The Write-Only-via-Function Pattern — the Core Test

This is the single most important integration test in the suite, because it's the mechanism that closes the "client self-reports a perfect score" cheating vector (`docs/PRODUCT_SPEC.md` §17, §19). It has two halves, and **both must be tested together in the same suite** to prove the pattern actually holds — testing only the positive half proves the Function works; testing only the negative half proves the rules exist; neither alone proves a client *cannot get the same result some other way*.

**4.1 Positive half — the Function can write:**

```
GIVEN an authenticated (emulated) test user with a valid ID token
AND a seeded Lesson definition matching a known-passing NoteEvent[] fixture
WHEN the client calls getExerciseFeedback with an ExerciseAttempt containing that NoteEvent[]
THEN the call succeeds (ok: true)
AND a new document exists at users/{uid}/practiceSessions/{sessionId} matching the expected ExerciseVerdict
AND users/{uid}.progressSummary[lessonId] is updated with the new bestScore/attempts/status
```

**4.2 Negative half — the client cannot write the same paths directly:**

```
GIVEN the same authenticated test user
WHEN that user's client SDK (not the Admin SDK, not a Function) attempts to directly
     write a document to users/{uid}/practiceSessions/{anySessionId}
     — including a "perfect score" payload crafted to look exactly like what
     getExerciseFeedback would have written
THEN the write is rejected by the emulated Firestore Security Rules (PERMISSION_DENIED)
```

Repeat 4.2 for every server-written-only path named in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`: `users/{uid}.progressSummary` (see §6 below — this one has a subtlety), `users/{uid}/aiMemory/summary`, and every field on `users/{uid}` other than `settings.*` (`currentLevel`, `currentLessonId`, `streak`, `xp` — per §4.15's field-level ownership table).

**4.3 Also test:** attempting the write as a *different* authenticated user (user B trying to write into user A's subtree) is rejected, and attempting it while unauthenticated is rejected — both are implied by, but distinct from, the same-user negative case in 4.2, and should be their own assertions.

## 5. `getExerciseFeedback` — Server-Side Re-Validation

Per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4, the Function treats every field of the incoming `ExerciseAttempt` as attacker-controlled — it does not trust `attemptNumber`, does not trust that `noteEvents` timestamps are honest, and recomputes the verdict itself by loading the `Lesson`/`Exercise` definition independently and re-running `packages/music-engine` server-side.

- **Tampered verdict rejected:** submit a `NoteEvent[]` that does *not* match the lesson's `expectedSequence` (e.g., missing every note) alongside a client-supplied hint suggesting a perfect score (if the callable's request shape allows any client-supplied scoring field at all — if it doesn't, per §20's callable design, assert instead that the Function's *response* verdict is computed solely from `noteEvents` and ignores any other client-supplied field). The written `PracticeSession.verdict` must reflect the actual `NoteEvent[]` submitted, never a client-suggested one.
- **Fabricated `attemptNumber` ignored:** submit `attemptNumber: 1` on a user who already has several prior `PracticeSession` records for that lesson; assert the Function computes the real attempt count server-side (from Firestore) rather than trusting the client's number, for anywhere `attemptNumber` affects behavior (e.g. consistency scoring across attempts, per `docs/PRODUCT_SPEC.md` §13).
- **Unauthenticated call rejected:** every callable Function checks `context.auth` before doing anything user-specific (`docs/PRODUCT_SPEC.md` §19) — assert a call with no auth context returns the typed error shape (`docs/PRODUCT_SPEC.md` §20: `{ ok: false, error: { code, message } }`), not a crash or a silent no-op.
- **Malformed `ExerciseAttempt` rejected cleanly:** missing required fields, an out-of-range `midiNote` in `noteEvents`, or neither/both of `lessonId`/`exerciseId` present (§4.4 requires exactly one) all return the typed error shape, not a 500 or an unhandled exception.
- **Abuse-prevention rule:** a new `practiceSessions` write is rejected if the learner's previous session completed less than the configured minimum interval ago (`docs/PRODUCT_SPEC.md` §19 — "blocking naive script-driven XP farming"); assert two rapid-fire calls from the same user produce one success and one rejection.
- **Error contract shape:** every response — success and failure — from every callable Function in this suite conforms to `{ ok: true, data }` or `{ ok: false, error: { code, message } }` (`docs/PRODUCT_SPEC.md` §20). Assert this shape generically as a shared test helper used across every Function test, not re-verified ad hoc per test.

## 6. `ProgressSummary` — a Field, Not a Subcollection (Rules-Testing Nuance)

`ProgressSummary` is embedded as a map field directly on the `users/{uid}` document (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7), on the *same document* as `settings` (client-writable) and `xp`/`streak`/`currentLevel` (Function-only). This means the negative-write test in §4.2 for this specific field cannot be "does the client have write access to some other path" — it must specifically prove that **a client cannot smuggle a `progressSummary` (or `xp`, `streak`, `currentLevel`, `currentLessonId`) change into a request that also legitimately touches `settings`.**

```
GIVEN an authenticated test user
WHEN the client SDK sends a single update() to users/{uid} that changes
     BOTH settings.volume (legitimately client-writable)
     AND progressSummary.<someLessonId>.bestScore (should be Function-only)
THEN the entire write is rejected
AND settings.volume is NOT silently applied as a partial success
```

This test exists because it is the failure mode a naive "allow write if `request.auth.uid == uid`" rule would miss entirely — see this document's cross-reference in `docs/ROADMAP/DEVELOPMENT_METHODOLOGY.md`'s open-questions handling and the flag raised in this batch's final report: `docs/PRODUCT_SPEC.md` §19 describes the settings rule using a `users/{uid}/settings` path notation that reads like a separate document, while `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` is explicit that `settings` is a nested field on the single `users/{uid}` document alongside Function-only fields. Whichever exact rule implementation `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` lands on (a field-level `diff().affectedKeys()` check is the likely correct shape), this test asserts the **outcome** that must hold regardless of implementation detail, which is the right level to test this at.

## 7. Account Deletion — Full Subtree Cascade

Per `docs/PRODUCT_SPEC.md` §18, this is non-negotiable day-one-scope for privacy compliance (POPIA/GDPR), not later polish.

```
GIVEN a test user with:
  - a users/{uid} profile document
  - multiple users/{uid}/practiceSessions/{sessionId} documents
  - a users/{uid}/aiMemory/summary document
WHEN deleteAccount is called by that authenticated user
THEN users/{uid} no longer exists
AND every document under users/{uid}/practiceSessions/** no longer exists
AND users/{uid}/aiMemory/summary no longer exists
AND the corresponding Firebase Auth user record no longer exists
AND a subsequent call to deleteAccount for the same (now-deleted) uid
    returns a clean typed error, not a crash
```

Also test: a user cannot call `deleteAccount` for a different `uid` (server trusts only `context.auth.uid`, never a client-supplied target uid, per §19's authentication-boundaries rule).

## 8. Data Export — Full Subtree Completeness

```
GIVEN the same fixture as §7 (profile + practiceSessions + aiMemory)
WHEN exportUserData is called by that authenticated user
THEN the returned export contains the users/{uid} profile fields
AND contains every practiceSessions document
AND contains the aiMemory/summary document
AND contains no other user's data
```

Assert completeness by comparing the exported record set's cardinality and key fields against what was seeded, not just that *some* export object came back non-empty — a silently-partial export (e.g., missing `practiceSessions` because a subcollection query was forgotten) is exactly the kind of bug this test exists to catch, since it's invisible in casual manual testing.

## 9. `lessons` Collection — Public-Read, Fully Write-Blocked

```
GIVEN the emulator seeded with at least one lesson document at lessons/{lessonId}
WHEN any client (authenticated or not) issues a read
THEN the read succeeds

WHEN any client (authenticated, including an admin-looking test user, or not)
     attempts any write to lessons/{lessonId} — create, update, or delete
THEN the write is rejected
```

Per `docs/PRODUCT_SPEC.md` §19, `lessons` is writable only via the Admin SDK (a seed script today, an admin Function in a later phase) — there is no authenticated-client code path that should ever succeed here, and this test suite should not special-case any uid as an exception without an explicit, documented admin mechanism that does not exist in MVP.

## 10. Error Handling

- Every Function under test returns the typed `{ok, data | error}` shape (§5 above) — never a raw thrown exception reaching the client, and never an HTTP-level 500 with no structured body for a *predictable* failure (bad input, auth failure, rate limit). An genuinely unexpected internal error may still surface as a generic `{ ok: false, error: { code: 'internal', message: '...' } }` rather than leaking a stack trace to the client.
- Firestore Security Rules denials surface to the client as `PERMISSION_DENIED` and the integration suite asserts on that specific error, not just "the write failed somehow."

## 11. Acceptance Criteria

- [ ] A test calling `getExerciseFeedback` asserts `practiceSessions`/`progressSummary` are written correctly (§4.1).
- [ ] A direct client SDK write attempt to those same paths is rejected by the emulated rules (§4.2), including the combined-write smuggling case (§6).
- [ ] Account deletion cascades through the full subtree — profile, `practiceSessions`, `aiMemory`, and the Auth record itself (§7).
- [ ] Data export returns the complete subtree with no silent omissions (§8).
- [ ] `lessons` is fully write-blocked for every client, authenticated or not (§9).
- [ ] Every tested Function returns the typed `{ok, data|error}` contract shape (`docs/PRODUCT_SPEC.md` §20).

## 12. Out of Scope

- Gemini response content/quality — mocked out entirely in this suite; see `AI_TESTING.md`.
- Load testing the emulator or Functions under concurrent traffic — not a pre-launch concern (`docs/TESTING/TESTING_STRATEGY.md` §9).
- Browser-driven UI assertions — this suite calls Functions and inspects Firestore state directly; full journeys through the actual rendered app are `E2E_TESTING.md`'s job, even where the underlying Function is the same one tested here.

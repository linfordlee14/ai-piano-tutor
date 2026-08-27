# Security Requirements

**Status:** Draft v1.0 — security documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §18 (Authentication), §19 (Security Architecture), §20 (API Architecture), §25 (Analytics), §29 (Acceptance Criteria — Security, AI tutor, and Lessons & Progress bullets); `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15 and §5
**Companion documents:** `docs/SECURITY/AUTHENTICATION.md`, `docs/SECURITY/AUTHORIZATION.md`, `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`, `docs/SECURITY/SECRETS_MANAGEMENT.md`, `docs/SECURITY/GEMINI_SECURITY.md`, `docs/SECURITY/APP_CHECK.md`, `docs/SECURITY/PRIVACY.md`; forward references to `docs/REQUIREMENTS_TRACEABILITY.md` and `docs/ARCHITECTURE/SECURITY_ARCHITECTURE.md` (both being written in parallel batches)

This document is the **master security requirement list** for AI Piano Tutor. Every requirement below has a stable ID (`SEC-01`, `SEC-02`, …) that must never be renumbered or reused — `docs/REQUIREMENTS_TRACEABILITY.md` references these IDs directly, and the companion documents in this folder implement them. If a requirement changes in nature enough that its ID no longer fits, retire the ID with a note rather than repurposing it.

---

## 1. Purpose

Enumerate every security requirement AI Piano Tutor must satisfy, derived from `docs/PRODUCT_SPEC.md` §18–21 and §29, each with a stable ID, a precise statement, its source, and testable acceptance criteria. This is the list every other document in `docs/SECURITY/` implements in detail — this document states *what* must be true; the companion documents state *how*.

## 2. Scope

In scope: every security requirement stated or directly implied by `docs/PRODUCT_SPEC.md` §18 (Authentication), §19 (Security Architecture), §20 (API error contract as it bears on security), §25 (Analytics privacy boundary), and the Security/AI-tutor/Lessons-&-Progress bullets of §29 (Acceptance Criteria).

Out of scope: implementation detail (rules syntax → `FIRESTORE_SECURITY_RULES.md`; secret storage mechanics → `SECRETS_MANAGEMENT.md`; Gemini-specific infrastructure → `GEMINI_SECURITY.md`; App Check mechanics → `APP_CHECK.md`; privacy/legal detail → `PRIVACY.md`); non-security requirements (functional, performance, accessibility — owned elsewhere in `docs/`); prompt wording and AI tone guardrails (`docs/AI/`) — this document covers only the *infrastructure* security boundary around AI calls, not their content quality.

## 3. Core philosophy — stated once, binding on every requirement below

**Default-deny.** Every Firestore Security Rule and every Cloud Function authorization check starts by denying everything, then explicitly allows only the specific, narrow thing that's needed. A rule or a Function handler is never written as "allow everything except X" — it is always "deny everything except Y." Where a requirement below doesn't spell this out explicitly, it's still the governing assumption.

**Gemini is never a source of truth.** Gemini is never asked to determine whether a note, chord, or scale was played correctly, and it is never trusted to validate, produce, or sanity-check a score or verdict (`docs/PRODUCT_SPEC.md` §14, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5). This is *why* `practiceSessions`, `progressSummary`, and `aiMemory` are Cloud-Function-written only (SEC-08, SEC-09, SEC-26 below) — that ownership rule is what closes the vector where a tampered client, or a manipulated/hallucinating AI response, could cause a fake result to become the persisted, trusted record.

## 4. Requirement index

| ID | Title | Category |
|---|---|---|
| SEC-01 | Multi-provider authentication | Authentication |
| SEC-02 | Anonymous session progress survives linking | Authentication |
| SEC-03 | Server-side identity verification on every Cloud Function | Authentication |
| SEC-04 | Uniform authorization — no unguarded unauthenticated code path | Authorization |
| SEC-05 | Firestore Security Rules are default-deny | Firestore rules |
| SEC-06 | `lessons/**` — public read, client write fully denied | Firestore rules |
| SEC-07 | `users/{uid}/settings` — owner-scoped read/write | Firestore rules |
| SEC-08 | `users/{uid}` Function-written fields are not client-writable | Firestore rules |
| SEC-09 | `practiceSessions/**` and `aiMemory/**` — owner read, client write fully denied | Firestore rules |
| SEC-10 | Server-side validation of all scored data | Score integrity |
| SEC-11 | No client-reported score is ever persisted verbatim | Score integrity |
| SEC-12 | Gemini credentials never in client code, public repo, or `NEXT_PUBLIC_*` | Gemini credential protection |
| SEC-13 | Gemini credentials stored only in Cloud Functions Secret Manager | Gemini credential protection |
| SEC-14 | No Gemini credential in the built client bundle (CI-enforced) | Gemini credential protection |
| SEC-15 | App Check enforced on Firestore | App Check |
| SEC-16 | App Check enforced on the Gemini-calling Cloud Functions | App Check |
| SEC-17 | Per-user Gemini rate limiting, atomic, server-side | Rate limiting |
| SEC-18 | Abuse prevention — `practiceSessions` write cooldown | Abuse prevention |
| SEC-19 | Prompt injection containment — text-only output, no tool access | Prompt injection protection |
| SEC-20 | Input validation and length caps on learner free text | Input validation |
| SEC-21 | Input validation on all structured client input | Input validation |
| SEC-22 | Secrets management baseline hygiene | Secrets management |
| SEC-23 | Account deletion | Account deletion |
| SEC-24 | Data export | Data export |
| SEC-25 | Analytics privacy boundary | Privacy |
| SEC-26 | AI output is never authoritative over scored/persisted data | Score integrity / Gemini trust boundary |

## 5. Requirements

### Authentication

**SEC-01 — Multi-provider authentication**
Requirement: AI Piano Tutor must support Anonymous (try-first), Google Sign-In, and Email/Password as Firebase Authentication providers at MVP launch, with the provider list additive by default so future providers (e.g. Apple Sign-In) can be added without a data-model change.
Source: `docs/PRODUCT_SPEC.md` §5, §6, §18.
Acceptance criteria:
- A new visitor can start using the piano via an anonymous session with zero signup friction.
- A returning learner can sign in with Google in one tap.
- A learner can sign up/sign in with email and password.
- Detail: `docs/SECURITY/AUTHENTICATION.md`.

**SEC-02 — Anonymous session progress survives linking**
Requirement: When a try-first anonymous visitor creates a real account (Google or email/password), their in-session progress must carry over to the new account via `linkWithCredential` rather than resetting to zero.
Source: `docs/PRODUCT_SPEC.md` §18, §6, §29.
Acceptance criteria (verbatim, `docs/PRODUCT_SPEC.md` §29):
- "An anonymous try-first session's progress survives linking to a real account"
- "Progress persists across sign-out/sign-in and across devices for the same account"
Note: the *conflict* sub-case — linking to a credential that already belongs to a separate existing account — has no ratified resolution policy. See `docs/SECURITY/AUTHENTICATION.md` §Security considerations, flagged as an **open question**, not silently resolved here.

**SEC-03 — Server-side identity verification on every Cloud Function**
Requirement: Every callable Cloud Function verifies the caller's Firebase ID token before doing anything user-specific. A client-supplied `uid` (in a request payload) is never trusted on its own — the only trusted identity source is the token-verified `context.auth.uid` populated by the Callable Functions platform itself.
Source: `docs/PRODUCT_SPEC.md` §18, §19.
Acceptance criteria:
- Every callable Function's handler rejects (does not proceed past its guard clause) when `context.auth` is null.
- A Function payload containing a `uid` field, if one is ever accidentally sent, is never read as the identity to act on.
- Anonymous users carry a real Firebase uid and are subject to the identical check — there is no separate "unauthenticated" code path (`docs/PRODUCT_SPEC.md` §19).
- Detail: `docs/SECURITY/AUTHENTICATION.md`.

### Authorization

**SEC-04 — Uniform authorization, no unguarded unauthenticated code path**
Requirement: Authorization rules apply identically regardless of whether the caller is an anonymous-uid user or a linked/authenticated-uid user. No code path exists that is reachable without *some* Firebase Auth identity (anonymous or otherwise) except the fully public `lessons/**` read.
Source: `docs/PRODUCT_SPEC.md` §19 ("Anonymous users still carry a real Firebase uid, so the same rules apply uniformly — there's no separate 'unauthenticated' code path to accidentally leave unguarded").
Acceptance criteria:
- A rules-unit-test suite exercises the same authorization assertions against both an anonymous-uid test identity and a linked-uid test identity, expecting identical outcomes for identical `uid` ownership.
- No admin/moderator role or bypass exists in MVP (`docs/PRODUCT_SPEC.md` §7/§34/§36 — Phase 3 admin CMS is out of scope).
- Detail: `docs/SECURITY/AUTHORIZATION.md`.

### Firestore rules

**SEC-05 — Firestore Security Rules are default-deny**
Requirement: `firestore.rules` starts by denying every read and write, then explicitly allows only the specific, narrow operations enumerated in SEC-06 through SEC-09. This must hold from the very first commit, not be retrofitted.
Source: `docs/PRODUCT_SPEC.md` §19 ("default deny, then explicit allow"), §28 (M0 milestone: "security rules default-deny from commit one").
Acceptance criteria:
- The ruleset ends in an unconditional catch-all `allow read, write: if false` for every path not explicitly matched above it.
- Detail: `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.

**SEC-06 — `lessons/**` — public read, client write fully denied**
Requirement: `lessons/{lessonId}` is readable by anyone (including a fully unauthenticated caller); it is writable only via the Admin SDK (a seed script today, an admin Function in Phase 3) — never from any client, authenticated or not.
Source: `docs/PRODUCT_SPEC.md` §17, §19.
Acceptance criteria (verbatim, `docs/PRODUCT_SPEC.md` §29):
- "A client cannot write to `lessons` at all"
- Detail and rules syntax: `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.

**SEC-07 — `users/{uid}/settings` — owner-scoped read/write**
Requirement: `settings` (`keyboardMapping`, `volume`, `sustainDefault`) is readable and writable only by the request whose `request.auth.uid` equals the document's `uid`. This is the one client-writable region of the `users/{uid}` document — a deliberate, narrow exception because it holds preferences, not scored facts (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.11, §4.15).
Source: `docs/PRODUCT_SPEC.md` §19.
Acceptance criteria:
- A rules test asserts a user can update their own `settings`.
- A rules test asserts a user cannot read or write another uid's `settings`.
- Detail and rules syntax: `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.

**SEC-08 — `users/{uid}` Function-written fields are not client-writable**
Requirement: every field on the `users/{uid}` document other than `settings` — `displayName`, `email`, `isAnonymous`, `createdAt`, `currentLevel`, `currentLessonId`, `streak`, `xp`, `progressSummary` — is readable by the owning uid but writable only by Cloud Functions via the Admin SDK. The rule denies any client write that touches these fields, whether alone or bundled alongside a legitimate `settings` update.
Source: `docs/PRODUCT_SPEC.md` §17, §19; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15's field-level ownership table ("everything else | No | Cloud Functions (Admin SDK) only").
Acceptance criteria (verbatim, `docs/PRODUCT_SPEC.md` §29):
- "A client cannot write its own `practiceSessions`/progress/AI-memory documents directly (verified by a rules test, not a manual check)" — the `progressSummary` clause of this bullet applies directly here, since `progressSummary` is embedded on this same document (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7), not a separate collection.
- A rules test asserts a client update that includes `xp`, `streak`, `progressSummary`, or `currentLevel` — even bundled with a valid `settings` change in the same write — is rejected in full.
- Detail and rules syntax: `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.

**SEC-09 — `practiceSessions/**` and `aiMemory/**` — owner read, client write fully denied**
Requirement: `users/{uid}/practiceSessions/{sessionId}` and `users/{uid}/aiMemory/summary` are readable by the owning uid and writable only by Cloud Functions via the Admin SDK — `allow write: if false` for every client request, with no partial-write exception (unlike `users/{uid}`, there is no client-writable field in either of these at all).
Source: `docs/PRODUCT_SPEC.md` §17, §19.
Acceptance criteria (verbatim, `docs/PRODUCT_SPEC.md` §29):
- "A client cannot write its own `practiceSessions`/progress/AI-memory documents directly (verified by a rules test, not a manual check)"
- Detail and rules syntax: `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.

### Score integrity

**SEC-10 — Server-side validation of all scored data**
Requirement: any data that determines correctness, accuracy, timing, completeness, consistency, or a composite score is computed server-side, inside a Cloud Function, by re-running `packages/music-engine` against the raw `NoteEvent[]` the client submitted and the `Lesson`/`Exercise` definition the server independently loads. The Function treats every field of the incoming `ExerciseAttempt` as attacker-controlled (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4).
Source: `docs/PRODUCT_SPEC.md` §13, §19; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4, §4.5.
Acceptance criteria:
- `getExerciseFeedback` never reads a `compositeScore`, `accuracy`, `missedNotes`, or any other verdict field from the client request — only `noteEvents`, and the informational-only `lessonId`/`exerciseId`/`deviceType`/`attemptNumber`/timestamps.
- A rules/emulator test submits a forged high-accuracy `ExerciseVerdict`-shaped payload and confirms the server ignores it and recomputes independently.

**SEC-11 — No client-reported score is ever persisted verbatim**
Requirement: no write path exists, anywhere in the system, by which a client-supplied score, accuracy value, or pass/fail verdict is written to Firestore without independent server-side recomputation first. This is the specific rule that closes the "modified client self-reports a perfect score" cheating vector.
Source: `docs/PRODUCT_SPEC.md` §19 ("a tampered client can't self-report a perfect score, because the client never writes scores at all").
Acceptance criteria:
- Combined with SEC-05/SEC-08/SEC-09 (rules deny the write path) *and* SEC-10 (the Function independently recomputes) — per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 1, both mechanisms must hold simultaneously; neither alone is sufficient.

### Gemini credential protection

**SEC-12 — Gemini credentials never in client code, public repo, or `NEXT_PUBLIC_*`**
Requirement: no Gemini API credential appears in `apps/web` source, in any file committed to the repository in plaintext, or in any environment variable prefixed `NEXT_PUBLIC_*` (which Next.js inlines into the client bundle at build time). This is **non-negotiable — no exceptions, including local testing.**
Source: `docs/PRODUCT_SPEC.md` §19.
Acceptance criteria: see SEC-14 (CI-enforced). Detail: `docs/SECURITY/SECRETS_MANAGEMENT.md`.

**SEC-13 — Gemini credentials stored only in Cloud Functions Secret Manager**
Requirement: the Gemini API credential is stored as a Cloud Functions 2nd-gen secret parameter (Secret Manager-backed), readable only inside the server-side execution context of the specific Function(s) it's bound to.
Source: `docs/PRODUCT_SPEC.md` §19 ("Stored as a Cloud Functions secret (Secret Manager), read only inside the Function's server-side execution").
Acceptance criteria: detail and the full secrets inventory: `docs/SECURITY/SECRETS_MANAGEMENT.md`.

**SEC-14 — No Gemini credential in the built client bundle (CI-enforced)**
Requirement: a CI step scans the built client bundle for the credential (or a recognizable fragment/pattern of it) and fails the build if found. This is a mechanical, automated check — not a code-review habit that can be skipped under deadline pressure.
Source: `docs/PRODUCT_SPEC.md` §29.
Acceptance criteria (verbatim, `docs/PRODUCT_SPEC.md` §29):
- "No Gemini credential appears anywhere in the built client bundle — grepped as a CI check, not a code-review habit"

### App Check

**SEC-15 — App Check enforced on Firestore**
Requirement: Firebase App Check is enabled, in enforced (not monitor-only) mode, on Firestore from day one — before M0/M1 ships, not retrofitted after abuse is observed.
Source: `docs/PRODUCT_SPEC.md` §16 ("Enabled from day one on Firestore *and* the Gemini-calling Functions — cheap to add early, painful to retrofit after abuse shows up").
Acceptance criteria: detail: `docs/SECURITY/APP_CHECK.md`.

**SEC-16 — App Check enforced on the Gemini-calling Cloud Functions**
Requirement: App Check is enabled, in enforced mode, on every Cloud Function that calls Gemini (`getTutorResponse`, `getExerciseFeedback`) from day one — this specifically blocks traffic that isn't the real app before it can spend Gemini quota, which is a cost-control property distinct from Firestore's abuse-prevention framing.
Source: `docs/PRODUCT_SPEC.md` §16, §19.
Acceptance criteria: detail: `docs/SECURITY/APP_CHECK.md`, `docs/SECURITY/GEMINI_SECURITY.md`.

### Rate limiting

**SEC-17 — Per-user Gemini rate limiting, atomic, server-side**
Requirement: a per-user rate-limit counter is checked and incremented atomically inside the Gemini-calling Function, *before* the Gemini call is made, enforcing a daily cap on tutor calls. "Atomic" means a Firestore transaction (read-check-write as one operation) — not a bare increment, which would not reject an over-cap request, and not a separate read-then-write, which is racy under concurrent requests.
Source: `docs/PRODUCT_SPEC.md` §19 ("a per-user rate-limit counter checked and atomically incremented inside the Function (e.g., a daily cap on tutor calls for free-tier users)").
Acceptance criteria (verbatim, `docs/PRODUCT_SPEC.md` §29):
- "Per-user rate limiting demonstrably blocks a scripted burst of tutor calls beyond the configured cap"
Note: the exact cap number is owned by `docs/AI/AI_COST_CONTROL.md` (itself an open question there, per the task defining this batch) — this requirement covers the *mechanism*, not the number. Detail: `docs/SECURITY/GEMINI_SECURITY.md`.

### Abuse prevention

**SEC-18 — Abuse prevention: `practiceSessions` write cooldown**
Requirement: a new `practiceSessions` write is rejected if the learner's previous session completed less than a short cooldown window ago, blocking naive script-driven XP farming. Because `practiceSessions` is already fully client-write-denied (SEC-09), the authoritative enforcement of this cooldown lives inside `getExerciseFeedback` itself (the only path by which this collection is ever written); the Firestore-rules-language version of this check is documented as an illustrative/defense-in-depth pattern, not the primary enforcement point.
Source: `docs/PRODUCT_SPEC.md` §19 ("a Firestore rule rejecting a new `practiceSessions` write if the learner's previous session completed less than a few seconds ago, blocking naive script-driven XP farming").
Acceptance criteria:
- A scripted burst of exercise-completion calls, submitted faster than the cooldown window, results in only the first being accepted; subsequent calls within the window are rejected with a typed error, not silently written.
- Exact cooldown window: **PROPOSED at 5 seconds** — `docs/PRODUCT_SPEC.md` §19 specifies only "less than a few seconds," no exact value is ratified. Treat 5s as a starting point to tune from real usage data, not a locked constant.
- Detail and rules-language sketch: `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.

### Prompt injection protection

**SEC-19 — Prompt injection containment: text-only output, no tool access**
Requirement: every Gemini call from every Cloud Function is scoped to plain-text generation only — no tool-calling, function-calling, or code-execution capability is ever granted to the model. A learner attempting a prompt-injection ("ignore your instructions and…") cannot extract the system prompt or trigger anything beyond text generation, because there is nothing beyond text generation for the model to reach.
Source: `docs/PRODUCT_SPEC.md` §19 ("the Function's Gemini call is scoped to plain text output only, with no tool- or function-calling access to anything sensitive").
Acceptance criteria:
- The Gemini client configuration used by every Cloud Function has no tools/functions registered.
- Detail: `docs/SECURITY/GEMINI_SECURITY.md`. (Content-level defenses — system-instruction wording, output-length caps, fact-invention guardrails — are owned by `docs/AI/AI_GUARDRAILS.md`; this requirement covers only the infrastructure containment.)

### Input validation

**SEC-20 — Input validation and length caps on learner free text**
Requirement: `learnerQuestion` (the one client-supplied field of a tutor request) is length-capped and validated server-side, before it is dropped into a prompt template — not merely capped client-side, which a modified client could bypass.
Source: `docs/PRODUCT_SPEC.md` §19 ("input validation/length caps on the learner's free-text question before it's dropped into a prompt").
Acceptance criteria: a request with an oversized/malformed `learnerQuestion` is rejected or truncated server-side regardless of what the client enforced. Exact cap value owned by `docs/AI/AI_GUARDRAILS.md`. Detail: `docs/SECURITY/GEMINI_SECURITY.md`.

**SEC-21 — Input validation on all structured client input**
Requirement: every field of every client-supplied payload is validated server-side against its documented shape and range before use — not just the AI-facing free text. This includes `NoteEvent.midiNote`/`velocity` range checks (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1: integers in `[0, 127]`, out-of-range values logged and rejected rather than crashing) and every field of `ExerciseAttempt`.
Source: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1, §4.4.
Acceptance criteria: an `ExerciseAttempt` containing an out-of-range `midiNote` is rejected (or the offending event is dropped and logged) rather than causing a crash or an undefined scoring result.

### Secrets management

**SEC-22 — Secrets management baseline hygiene**
Requirement: `.env.local` is gitignored and never committed; all sensitive environment variables are server-side only (never `NEXT_PUBLIC_*`); dependency scanning (`npm audit`/Dependabot) runs as a matter of course, not an occasional manual check.
Source: `docs/PRODUCT_SPEC.md` §19 ("secrets stay out of the repo (`.env.local` gitignored), all sensitive env vars are server-side only, and dependency scanning (`npm audit`/Dependabot) runs as a matter of course").
Acceptance criteria: `.gitignore` includes `.env.local` (and equivalents) from the first commit; Dependabot (or equivalent) is configured on the repository before M0 is called complete. Detail: `docs/SECURITY/SECRETS_MANAGEMENT.md`.

### Account deletion

**SEC-23 — Account deletion**
Requirement: a Cloud Function (`deleteAccount`) deletes a user's full data subtree — the `users/{uid}` document, its `practiceSessions` and `aiMemory` subcollections, and the underlying Firebase Auth record — on the owning user's own request. This is **non-negotiable day-one MVP scope**, not later polish.
Source: `docs/PRODUCT_SPEC.md` §6, §18, §20.
Acceptance criteria: ships no later than the M5 hardening milestone (`docs/PRODUCT_SPEC.md` §28); only the owning `context.auth.uid` can trigger deletion of their own data (no admin override exists in MVP — `docs/SECURITY/AUTHORIZATION.md`). Detail: `docs/SECURITY/PRIVACY.md`.

### Data export

**SEC-24 — Data export**
Requirement: a Cloud Function (`exportUserData`) exports a user's full data subtree, in the same scope as SEC-23, on the owning user's own request. Equally non-negotiable day-one MVP scope.
Source: `docs/PRODUCT_SPEC.md` §6, §18, §20.
Acceptance criteria: same ship timing and ownership constraints as SEC-23. Detail: `docs/SECURITY/PRIVACY.md`.

### Privacy

**SEC-25 — Analytics privacy boundary**
Requirement: the analytics stream (Firebase/Google Analytics) collects product-usage events only — screen views, lesson starts/completions, taps, AI tutor call *counts*. It never receives tutor conversation content or the specific notes a learner played.
Source: `docs/PRODUCT_SPEC.md` §25 ("collect product-usage events only... never send tutor conversation content or the specific notes a learner played into the general analytics stream. That data already lives correctly-scoped in Firestore under the user's own record; duplicating it into an analytics tool would only create a second, less-protected copy").
Acceptance criteria: an audit of every analytics event call site confirms no event payload contains a note name/MIDI number, a chord/scale identification, or any string sourced from tutor chat content. Detail: `docs/SECURITY/PRIVACY.md`.

### Score integrity / Gemini trust boundary

**SEC-26 — AI output is never authoritative over scored/persisted data**
Requirement: Gemini is never asked to determine whether a note, chord, or scale was played correctly, and no field of `ExerciseVerdict`, `PracticeSession`, `ProgressSummary`, or `AIMemorySummary` is ever produced, modified, or "sanity-checked" by a Gemini response. Gemini receives a verdict the deterministic engine already computed and explains it in prose — it never participates in producing it. This is the specific rule that closes the vector where a manipulated or hallucinating AI response could cause a fake result to become the persisted, trusted record.
Source: `docs/PRODUCT_SPEC.md` §14, §19; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5 ("Gemini never produces, modifies, or is asked to sanity-check any field on this object").
Acceptance criteria (verbatim, `docs/PRODUCT_SPEC.md` §29):
- "A tutor response never states a note/chord/theory fact that contradicts the structured data it was given"
- "Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly, not assumed"
- A code-level audit confirms no Cloud Function ever writes a Gemini response's content into `ExerciseVerdict`, `PracticeSession.verdict`, `ProgressSummary`, or any `xp`/`streak`/`currentLevel` field.

## 6. Dependencies

- `docs/PRODUCT_SPEC.md` §18–21, §29 — the source requirements this document restates with IDs.
- `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15, §5 — field-level ownership and the redundant-enforcement doctrine (rules + Function, both, always) that SEC-05–SEC-11 and SEC-26 depend on.
- `docs/ARCHITECTURE/DATA_FLOW.md`, `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` — confirm the request/response shapes these requirements govern (`ExerciseAttempt` in, `ExerciseVerdict` out; `{ learnerQuestion }` in, `TutorResponse` out).
- `docs/DATA/FIRESTORE_SCHEMA.md` (parallel batch, not yet written) — the full collection list this document's Firestore-rules requirements must stay in sync with as it lands.
- `docs/AI/AI_COST_CONTROL.md`, `docs/AI/AI_GUARDRAILS.md` (parallel batch, not yet written) — own the exact numeric caps SEC-17/SEC-20 reference by mechanism, not by number.

## 7. Out of scope

- Rules syntax, secret storage mechanics, App Check provider choice, and privacy/export implementation detail — each owned by its named companion document, not restated here.
- Prompt wording, tone, and theory-fact guardrail content — `docs/AI/`.
- Non-security requirements (functional/UX/performance/accessibility acceptance criteria) — owned by their respective spec sections and documents.
- A legal compliance review of POPIA/GDPR applicability — recommended, not performed, by this document set. See `docs/SECURITY/PRIVACY.md` §Security considerations, flagged explicitly as an open question.

# Authorization

**Status:** Draft v1.0 — security documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §7, §17, §18, §19, §34, §36; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15, §5
**Companion documents:** `docs/SECURITY/SECURITY_REQUIREMENTS.md` (SEC-04), `docs/SECURITY/AUTHENTICATION.md`, `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`

This is a **new document** — `docs/PRODUCT_SPEC.md` does not contain an authorization-specific section; its content is derived from the authorization statements scattered across §17–19 and consolidated here as the single reference for "who can do what."

## 1. Purpose

State, in one place, every actor class in AI Piano Tutor and exactly what each can read and write. `docs/SECURITY/AUTHENTICATION.md` establishes *who someone is*; this document establishes *what that identity is then allowed to do*. `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` is where that "what" becomes enforced rules syntax — this document is the policy table that rules file implements.

## 2. Scope

In scope: the complete set of actor classes (anonymous user, authenticated user, Cloud Functions/Admin SDK, unauthenticated caller, admin/moderator) and a read/write matrix for each against every data region named in `docs/PRODUCT_SPEC.md` §17.

Out of scope: how identity is established (`AUTHENTICATION.md`), the literal rules syntax that enforces this table (`FIRESTORE_SECURITY_RULES.md`), App Check (a request-provenance check that runs independently of and prior to authorization — `APP_CHECK.md`).

## 3. Requirements

- No actor may read or write another actor's `uid`-scoped data, ever — every rule in this document is scoped to `request.auth.uid == uid` in the resource path.
- Anonymous users are full actors on their own data, not a restricted tier — `docs/PRODUCT_SPEC.md` describes anonymous users as carrying "a real Firebase uid," with the same Firestore document shape and the same rule engine applied uniformly (§17, §19). The only asymmetry between anonymous and linked users is *recoverability*, not *authorization* — see §4.2.
- Cloud Functions acting via the Admin SDK are the trusted server-side compute layer, by design — this is not a loophole to be closed, it is the mechanism by which SEC-10/SEC-11 (server-side score recomputation) is possible at all.
- No admin/moderator role exists in MVP. Phase 3's admin CMS (`docs/PRODUCT_SPEC.md` §7, §34, §36) is explicitly out of scope — no role, custom claim, or privileged code path for it should be implemented, tested, or referenced anywhere in MVP code.

Full requirement statement with ID: `docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-04.

## 4. Architecture

### 4.1 Actor classes

| Actor | How identified | Exists in MVP? |
|---|---|---|
| Unauthenticated caller | No Firebase session at all | Yes — the pre-auth Landing screen state, before even an anonymous session starts |
| Anonymous user | Firebase Auth anonymous session, real uid, `isAnonymous: true` | Yes |
| Authenticated (linked) user | Firebase Auth session via Google or Email/Password, real uid, `isAnonymous: false` | Yes |
| Cloud Functions (Admin SDK) | No `request.auth` — a trusted server execution context, not a client request | Yes |
| Admin / moderator | A privileged role/claim | **No — does not exist in MVP** |

### 4.2 Why anonymous and authenticated users are treated identically

A naive reading of "try before you buy" might assume anonymous users get a *reduced* capability set until they sign up. That is not how this system is built. Per `docs/PRODUCT_SPEC.md` §17/§19, an anonymous sign-in creates a real Firebase uid and a real `users/{uid}` Firestore document, structurally identical to a linked user's. Every Firestore Security Rule in `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` is written in terms of `request.auth.uid == uid` — it has no branch on `request.auth.token.firebase.sign_in_provider`, so it cannot distinguish "anonymous" from "linked" even if it wanted to.

**The precise, correct statement:** anonymous users have **full read/write parity with authenticated users on their own uid's data.** They can play piano, attempt exercises, have those exercises server-scored and persisted, accumulate XP/streak/progress, and use the AI tutor — all of it, immediately, with no signup. The **only** difference between an anonymous user and a linked user is **recoverability**: if an anonymous learner loses their device or clears browser storage, there is no credential (no email, no password, no Google account) to sign back in with, and that uid's data — while never deleted — becomes practically unreachable to them. A linked user can sign in again from any device and resume. This is a recoverability property, not an authorization property, and it's the entire reason `docs/SECURITY/AUTHENTICATION.md` §4.3's linking flow exists — not to *unlock* capability, but to attach a *recovery path* to capability the learner already has.

### 4.3 The authorization matrix

| Actor | Read `lessons/**` | Read own `users/{uid}` (incl. `progressSummary`) | Write own `users/{uid}.settings` | Write own `users/{uid}` other fields (direct) | Read own `practiceSessions/**`, `aiMemory/**` | Write own `practiceSessions/**`, `aiMemory/**` (direct) | Read/write another uid's data | Write `lessons/**` |
|---|---|---|---|---|---|---|---|---|
| Unauthenticated caller | ✅ | ❌ (no uid to own) | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Anonymous user | ✅ | ✅ (own uid only) | ✅ (own uid only) | ❌ | ✅ (own uid only) | ❌ | ❌ | ❌ |
| Authenticated (linked) user | ✅ | ✅ (own uid only) | ✅ (own uid only) | ❌ | ✅ (own uid only) | ❌ | ❌ | ❌ |
| Cloud Functions (Admin SDK) | ✅ | ✅ (any uid) | ✅ (any uid) | ✅ (any uid — this is the *only* path) | ✅ (any uid) | ✅ (any uid — this is the *only* path) | ✅ (by design — trusted layer) | ✅ (seed script / admin Function only) |
| Admin/moderator | — | — | — | — | — | — | — | — |

Notes on the table:
- "Direct" writes mean via a client SDK, subject to Firestore Security Rules. Cloud Functions writes via the Admin SDK **bypass rules entirely** — the "❌" cells for anonymous/authenticated users on `users/{uid}` other-fields and on `practiceSessions`/`aiMemory` are exactly what those Function-only writes exist opposite of.
- No row includes any cross-uid capability for a client actor — there is no social/leaderboard feature in MVP (`docs/PRODUCT_SPEC.md` §6: "Leaderboards / multiplayer `[Future, low priority]`"), so there is no concept of one learner reading another learner's progress, session history, or AI memory. If a social feature is ever built, it needs its own explicit rule and its own row in this table — it does not fall out of anything here.
- The admin/moderator row is intentionally empty, not "TBD" — see §4.4.

### 4.4 Why Cloud Functions "bypassing" rules is not a loophole

Framing this precisely matters, because "the Admin SDK bypasses security rules" can sound like a gap if read out of context. It is not: Cloud Functions running via the Admin SDK **are** the trusted server-side compute layer this entire security model is built around (`docs/PRODUCT_SPEC.md` §16). Client trust ends at the Cloud Function boundary — that's the point at which every requirement in `docs/SECURITY/SECURITY_REQUIREMENTS.md` about server-side recomputation (SEC-10, SEC-11) and about Gemini never being authoritative (SEC-26) actually gets enforced. A rules-bypassing write is only safe *because* the Function that issues it independently re-validates/recomputes everything it writes, rather than trusting any client-supplied value (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5, rule 1: "Neither mechanism alone is sufficient... Both must hold"). The authorization boundary in this system is not "what can a request get past the rules file" — it's "what can a request get a Cloud Function to do," and every Cloud Function is written defensively against exactly that question.

### 4.5 No admin/moderator role in MVP

This is stated as a requirement, not an oversight: no custom claim, no `isAdmin` field, no privileged read/write path, and no admin-only Cloud Function should exist anywhere in MVP code. `docs/PRODUCT_SPEC.md` explicitly defers the admin CMS to Phase 3 (§7's deferred-items table, §34's final MVP definition, §36 referenced alongside them) — building any part of an admin authorization model now would be scope creep against the MVP cut the spec is explicit about (§7: "Everything not required to prove that statement is out of v1"). When Phase 3 admin tooling is scoped, it needs its own authorization design — a new actor row, new rules, and almost certainly Firebase custom claims — none of which this document anticipates or partially builds.

## 5. Interfaces/contracts

The authorization matrix in §4.3 is the contract. It is implemented two ways, redundantly, per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 1:
1. Firestore Security Rules (`docs/SECURITY/FIRESTORE_SECURITY_RULES.md`) — deny the client-side write/read paths the matrix marks ❌.
2. Cloud Function handler logic (`docs/SECURITY/AUTHENTICATION.md` §6) — every Function checks `context.auth` and scopes every Firestore operation it performs to `context.auth.uid`, independent of what the rules file would have allowed.

## 6. Dependencies

- `docs/SECURITY/AUTHENTICATION.md` — the identity this document's rows are keyed on.
- `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` — the enforcement of this matrix in rules syntax.
- `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15 — the field-level ownership table this matrix's `users/{uid}` columns restate at the actor level.

## 7. Security considerations

- The matrix's ❌ cells are exactly as important as its ✅ cells — a rules-unit-test suite should assert both directions explicitly (see §8), because a default-deny system that's only tested for what it *allows* can silently regress into allowing something it shouldn't.
- Anonymous-user parity (§4.2) is a deliberate design choice with a real trade-off: it means a device-loss event is unrecoverable data loss from the learner's perspective (though the data itself persists server-side, orphaned). This is a product/UX consideration the linking flow (`docs/SECURITY/AUTHENTICATION.md`) exists to mitigate by encouraging linking, not a flaw in the authorization model itself.

## 8. Testing requirements

Each row/column intersection in §4.3 maps to at least one rules-unit-test case (`@firebase/rules-unit-testing`, per `docs/PRODUCT_SPEC.md` §26):
- An anonymous-identity test principal can read/write its own `settings` and cannot write its own `progressSummary`/`xp`/etc.
- An anonymous-identity test principal cannot read or write a *different* uid's `users/{uid}` document, `practiceSessions`, or `aiMemory` — same assertions repeated for a linked-identity principal, expecting identical outcomes (this is what makes SEC-04's "uniform authorization" claim verified, not assumed).
- An unauthenticated (no-auth) test principal can read `lessons/**` and nothing else.
- No test principal — anonymous, linked, or unauthenticated — can write to `lessons/**`.
- No test asserts an "admin" capability, because none exists; if one is ever added in Phase 3, it arrives with its own new tests, not a relaxation of these.

## 9. Acceptance criteria

Verbatim, `docs/PRODUCT_SPEC.md` §29:
- "A client cannot write its own `practiceSessions`/progress/AI-memory documents directly (verified by a rules test, not a manual check)"
- "A client cannot write to `lessons` at all"

Derived:
- The full §4.3 matrix has a passing rules-unit-test for every ✅ and every ❌ cell that a client (as opposed to a Cloud Function) could conceivably attempt.

## 10. Out of scope

- Admin/moderator authorization design — Phase 3, not started, not partially built here (§4.5).
- Social/shared-data authorization (leaderboards, shared progress views) — no such feature exists in MVP (`docs/PRODUCT_SPEC.md` §6); would need its own row and rule if built later.
- Identity establishment itself — `docs/SECURITY/AUTHENTICATION.md`.
- Request-provenance verification (App Check) — a distinct, earlier check in the request pipeline, not an authorization decision — `docs/SECURITY/APP_CHECK.md`.

# Firestore Security Rules

**Status:** Draft v1.0 — security documentation with illustrative rules syntax, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §17, §19, §29; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7, §4.11, §4.15, §5
**Companion documents:** `docs/SECURITY/SECURITY_REQUIREMENTS.md` (SEC-05 through SEC-09, SEC-18), `docs/SECURITY/AUTHORIZATION.md` (the policy this file enforces), `docs/DATA/FIRESTORE_SCHEMA.md` (parallel batch, not yet written — canonical full collection/field list)

This is the one document in `docs/SECURITY/` where literal Firestore Rules-language syntax is expected and provided. Every other document in this folder describes policy in prose; this one is the policy compiled into the actual `firestore.rules` language.

## 1. Purpose

Specify the exact Firestore Security Rules logic for every collection named in `docs/PRODUCT_SPEC.md` §17/§19, as real rules-language syntax ready to paste into `firestore.rules` and adjust for the project's exact deployment. Every rule here implements one or more requirements from `docs/SECURITY/SECURITY_REQUIREMENTS.md`.

## 2. Scope

In scope: rules for `lessons/**`, `users/{uid}` (including the `settings` vs. everything-else split), `users/{uid}/practiceSessions/**`, `users/{uid}/aiMemory/**`, and the default-deny catch-all.

Out of scope: rules for collections not yet named in `docs/PRODUCT_SPEC.md` §17 (e.g. Phase 2's `songs`, `AIConversations` — explicitly deferred, §17) — add them here, following the same default-deny pattern, only when `docs/DATA/FIRESTORE_SCHEMA.md` ratifies their shape. Firebase Storage rules — not used in MVP (`docs/PRODUCT_SPEC.md` §16). App Check enforcement configuration — a separate console/CLI-level setting, not expressed in `firestore.rules` syntax (`docs/SECURITY/APP_CHECK.md`).

## 3. Requirements

Every rule below exists to satisfy one of `docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-05 (default-deny baseline), SEC-06 (`lessons`), SEC-07 (`settings`), SEC-08 (`users/{uid}` other fields), SEC-09 (`practiceSessions`/`aiMemory`), or SEC-18 (abuse-prevention cooldown, illustrative). Each rule below is annotated with which requirement it implements.

## 4. Architecture

### 4.1 Design decision: who creates `users/{uid}`

Before the ruleset, one implementation decision needs stating explicitly, because it shapes the `users/{uid}` rule in §5.3: **`docs/PRODUCT_SPEC.md` does not say who creates the initial `users/{uid}` document.** The only reading consistent with §17/§19's "everything except `settings` is Function-written only" is that document *creation* is Function-written too — otherwise there would be a special-cased client bootstrap path that the field-level ownership table doesn't account for, and a client "create" with attacker-chosen initial `xp`/`streak`/`currentLevel` values would be a real exploit if ever allowed. This document assumes an Auth `onCreate` trigger (Cloud Function, Admin SDK) provisions the document at account-creation time — see `docs/SECURITY/AUTHENTICATION.md` §4.2 step 3. Accordingly, `allow create: if false` for the client, full stop, matching every other non-`settings` write on this document.

### 4.2 Helper functions

Used throughout the ruleset below:

```
function isSignedIn() {
  return request.auth != null;
}

function isOwner(uid) {
  return isSignedIn() && request.auth.uid == uid;
}
```

`isOwner` is deliberately the *only* authorization primitive this ruleset uses for user-scoped data — there is no branch on `request.auth.token.firebase.sign_in_provider`, which is what makes anonymous and linked users identical under these rules (`docs/SECURITY/AUTHORIZATION.md` §4.2).

## 5. Interfaces/contracts — the ruleset

### 5.1 File skeleton

```
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {

    function isSignedIn() {
      return request.auth != null;
    }

    function isOwner(uid) {
      return isSignedIn() && request.auth.uid == uid;
    }

    // --- lessons/** ------------------------------------------------------
    match /lessons/{lessonId} {
      allow read: if true;
      allow write: if false;
    }

    // --- users/{uid} and its subcollections -------------------------------
    match /users/{uid} {
      allow read: if isOwner(uid);
      allow create: if false;
      allow update: if isOwner(uid)
        && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['settings']);
      allow delete: if false;

      match /practiceSessions/{sessionId} {
        allow read: if isOwner(uid);
        allow write: if false;
      }

      match /aiMemory/{docId} {
        allow read: if isOwner(uid);
        allow write: if false;
      }
    }

    // --- default deny -------------------------------------------------
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

This is the complete, default-deny ruleset for every collection named in `docs/PRODUCT_SPEC.md` §17 (SEC-05). Each piece is broken out and justified below.

### 5.2 `lessons/{lessonId}` — public read, client write fully denied (SEC-06)

```
match /lessons/{lessonId} {
  allow read: if true;
  allow write: if false;
}
```

Public read, unconditionally — even an unauthenticated caller can read lesson content, matching `docs/PRODUCT_SPEC.md` §19's exact rule (`allow read: if true; allow write: if false;`). Write is denied for **every** caller, including a signed-in owner of nothing in particular — there is no `uid` concept for content. The only way `lessons/**` is ever written is the Admin SDK, either the seed script that loads static JSON at deploy time today, or an admin Function in Phase 3 (`docs/PRODUCT_SPEC.md` §17) — neither goes through this rule at all, by definition of using the Admin SDK.

### 5.3 `users/{uid}` — the settings/everything-else split (SEC-07, SEC-08)

```
match /users/{uid} {
  allow read: if isOwner(uid);
  allow create: if false;
  allow update: if isOwner(uid)
    && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['settings']);
  allow delete: if false;
}
```

- **`allow read: if isOwner(uid)`** — the owning uid can read the whole document, including `progressSummary`, which is embedded on this same document rather than a separate collection (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7). This single rule is what makes `progressSummary` "readable by the owner" true without needing a separate rule block for it.
- **`allow create: if false`** — see §4.1. Document provisioning is Function-only.
- **`allow update: if isOwner(uid) && ...hasOnly(['settings'])`** — this is the field-level split the task set out to solve: `settings`, `progressSummary`, `xp`, `streak`, `currentLevel`, `currentLessonId`, `displayName`, `email`, `isAnonymous`, and `createdAt` are all fields on the *same* document (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15), so per-document rules can't distinguish them by path the way subcollection rules can. `diff().affectedKeys().hasOnly([...])` is the standard Firestore Rules pattern for exactly this: it computes which top-level field names changed between the existing document (`resource.data`) and the incoming write (`request.resource.data`), and rejects the write unless that changed-set is a subset of `['settings']`. A client update that touches `settings` alone succeeds; one that touches `progressSummary` alone fails; one that touches `settings` *and* `xp` in the same write **also fails in full** — Firestore rules evaluate the whole write as one unit, so there is no partial-success where the `settings` part is silently kept and the `xp` part is silently dropped. This directly satisfies the §29 acceptance bullet's `progressSummary` clause and `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15's field-level ownership table in one rule.
- **`allow delete: if false`** — account deletion is a Cloud Function operation (`deleteAccount`, `docs/SECURITY/PRIVACY.md`), not a direct document delete, because it must also delete the subcollections below and the Firebase Auth record — a bare Firestore `delete` on the root document would silently orphan `practiceSessions`/`aiMemory` (Firestore does not cascade-delete subcollections).

**Optional strengthening — validating the shape of `settings` itself.** The rule above permits *any* value for `settings` as long as no other top-level key changes. A stricter version also validates the settings shape against `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15:

```
function isValidSettingsUpdate() {
  let s = request.resource.data.settings;
  return s.volume is number && s.volume >= 0 && s.volume <= 1
    && s.sustainDefault is bool
    && s.keyboardMapping is map;
}

match /users/{uid} {
  allow update: if isOwner(uid)
    && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['settings'])
    && isValidSettingsUpdate();
  // ... read/create/delete as above
}
```

This is a defense-in-depth addition, not a requirement `docs/PRODUCT_SPEC.md` states explicitly — recommended because it costs nothing and closes an otherwise-open door (a client could currently write `settings.volume: "loud"` or omit `sustainDefault` entirely and the base rule wouldn't object).

### 5.4 `users/{uid}/practiceSessions/{sessionId}` and `users/{uid}/aiMemory/{docId}` — owner read, client write fully denied (SEC-09)

```
match /users/{uid}/practiceSessions/{sessionId} {
  allow read: if isOwner(uid);
  allow write: if false;
}

match /users/{uid}/aiMemory/{docId} {
  allow read: if isOwner(uid);
  allow write: if false;
}
```

Unlike `users/{uid}` itself, neither of these has *any* client-writable field — there is no `settings`-equivalent carve-out, so `allow write: if false` is unconditional. `aiMemory` uses `{docId}` rather than hardcoding `summary` so the rule still matches if the single-document convention (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8) is ever revisited; it does not weaken the rule, since write is denied regardless of `docId`.

### 5.5 Abuse prevention — the `practiceSessions` cooldown (SEC-18)

`docs/PRODUCT_SPEC.md` §19 states: *"a Firestore rule rejecting a new `practiceSessions` write if the learner's previous session completed less than a few seconds ago."* Read literally next to §5.4 above, this appears to be in tension with itself — `practiceSessions` is already `allow write: if false` for every client request, cooldown or not, so a rules-language cooldown *condition* has nothing to attach to under the active rule.

**Resolution, stated explicitly rather than left ambiguous:** the two statements are consistent once read through `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 1's redundant-enforcement doctrine. `practiceSessions` writes only ever happen one way — inside `getExerciseFeedback`, via the Admin SDK, which bypasses this file's rules entirely. That means:

- The **authoritative** enforcement of the cooldown must live inside `getExerciseFeedback` itself: before writing a new `PracticeSession`, the Function queries the learner's most recent prior session's `completedAt` and rejects (returns `{ ok: false, error }`, writes nothing) if the cooldown hasn't elapsed. This is application logic, not rules syntax, and belongs in `functions/src/scoring/`, not in this document (this document is rules-only, per the task that produced it).
- The **rules-language sketch below is illustrative** — it documents the pattern for the record and serves as a defense-in-depth backstop should a future regression ever introduce a client-authenticated write path to this collection (at which point it would no longer be inert). It is **not reachable today**, since `allow write: if false` already blocks every client write unconditionally, cooldown or not.

A rules-only cooldown check has one real constraint worth documenting: Firestore Rules cannot run a "find the most recent document in this subcollection" query — `get()`/`exists()` only read a document at a **known, exact path**. The workable rules-native pattern is to mirror the last-completion timestamp onto a well-known single field on the parent `users/{uid}` document (itself Function-written, alongside `progressSummary`), so a rule can `get()` it directly:

```
// ILLUSTRATIVE — not reachable under the active rule in §5.4, which is
// `allow write: if false` unconditionally. Shown here to document the
// pattern and as a defense-in-depth backstop only.
//
// Requires a Function-written `lastSessionCompletedAt` timestamp field on
// the parent users/{uid} document (would need to be added to
// docs/ENGINEERING/TYPES_AND_CONTRACTS.md §4.15 and docs/DATA/FIRESTORE_SCHEMA.md
// if this pattern is ever activated — not currently part of UserProfile).
//
// Cooldown window: PROPOSED at 5 seconds. docs/PRODUCT_SPEC.md §19 specifies
// only "less than a few seconds" — no exact value is ratified. Tune from
// real usage data once it exists; this is a starting point, not a constant
// to build other logic on top of.
function priorSessionTooRecent(uid) {
  let lastCompleted = get(/databases/$(database)/documents/users/$(uid)).data.lastSessionCompletedAt;
  return lastCompleted != null && request.time < lastCompleted + duration.value(5, 's');
}

match /users/{uid}/practiceSessions/{sessionId} {
  allow read: if isOwner(uid);
  allow create: if isOwner(uid) && !priorSessionTooRecent(uid); // illustrative only — see note above
  allow update, delete: if false;
}
```

Do not deploy this variant without also removing the unconditional `allow write: if false` it would otherwise contradict, and without first deciding, deliberately, that client writes to this collection should ever be permitted at all — which `docs/PRODUCT_SPEC.md` §19 does not call for. As written in §5.4, this is documentation of a pattern, not a recommendation to loosen the active rule.

## 6. Data structures

The collections this file governs, restated from `docs/PRODUCT_SPEC.md` §17 (full canonical field-level shapes: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`; full collection list including any added post-MVP: `docs/DATA/FIRESTORE_SCHEMA.md`, parallel batch, not yet written — this table must be kept in sync with that document as it lands):

| Path | Client read | Client write | Actual writer |
|---|---|---|---|
| `lessons/{lessonId}` | ✅ (public) | ❌ | Admin SDK (seed script; admin Function in Phase 3) |
| `users/{uid}` — `settings` field | ✅ (owner) | ✅ (owner) | Client (own uid) |
| `users/{uid}` — every other field, incl. `progressSummary` | ✅ (owner) | ❌ | Cloud Functions (Admin SDK) |
| `users/{uid}/practiceSessions/{sessionId}` | ✅ (owner) | ❌ | Cloud Functions (Admin SDK), `getExerciseFeedback` |
| `users/{uid}/aiMemory/summary` | ✅ (owner) | ❌ | Cloud Functions (Admin SDK) |

## 7. Dependencies

- `docs/SECURITY/AUTHORIZATION.md` §4.3 — the policy matrix this ruleset compiles.
- `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7, §4.11, §4.15 — exact field shapes referenced by the `isValidSettingsUpdate()` variant.
- `docs/DATA/FIRESTORE_SCHEMA.md` (parallel batch) — canonical full collection list; this file's collection set must track it, especially if Phase 2 collections (`songs`, `AIConversations`) land.
- `@firebase/rules-unit-testing` and the Firebase Local Emulator Suite — the only way these rules are actually verified (`docs/PRODUCT_SPEC.md` §26).

## 8. Security considerations

- **Default-deny is structural here, not just stated:** the file ends in an unconditional `match /{document=**} { allow read, write: if false; }`. Any new collection added later that forgets its own explicit rule block is denied by this catch-all, not silently allowed — this is what makes SEC-05 true by construction rather than by discipline.
- **The rules file alone is not sufficient for score integrity.** Denying client writes to `practiceSessions`/`progressSummary`/`aiMemory` closes the *client* side of the tampering vector; it says nothing about whether the Function that *does* write them computed the right value. `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 1 is explicit that both layers — rules denial and Function-side recomputation — must hold together. This document only owns the first.
- **`diff().affectedKeys().hasOnly([...])` fails closed.** If a future field is added to `UserProfile` and a developer forgets it should be Function-only, the *default* behavior of this rule is still to reject any client write that touches it (since it's not `'settings'`) — the failure mode of forgetting to update this file is "a legitimate settings-adjacent feature doesn't work" (loud, caught in QA), not "a new field becomes silently client-writable" (quiet, a real vulnerability). This asymmetry is deliberate and is the point of listing the *allowed* key (`settings`) rather than a *denied* list.

## 9. Performance considerations

- The illustrative cooldown pattern in §5.5 uses `get()` inside a rule, which counts as an extra document read against quota/billing for every evaluation of that rule — one more reason the authoritative check belongs in the Function (which already has to read Firestore for other reasons in the same request) rather than in rules, where it would be a dedicated extra read purely for this check.
- `lessons/**` being `allow read: if true` means no per-request auth check overhead for the app's most frequently read, least sensitive collection.

## 10. Error handling

A denied rule produces a `PERMISSION_DENIED` error from the Firestore SDK. Client code that never attempts a denied write (i.e., correctly-written client code only ever touches `settings`) never encounters this in normal operation — a `PERMISSION_DENIED` surfacing in production logs on `users/{uid}`, `practiceSessions`, or `aiMemory` is itself a signal worth alerting on, since it means either a client bug or an active tampering attempt.

## 11. Testing requirements

Per `docs/PRODUCT_SPEC.md` §26 and §29, using `@firebase/rules-unit-testing` against the Local Emulator Suite:

- A client (any authenticated identity) **cannot** write to `lessons/**` — any document, any field.
- An unauthenticated caller **can** read `lessons/**`.
- The owning uid **can** update `users/{uid}.settings` alone.
- The owning uid **cannot** update `users/{uid}.progressSummary`, `.xp`, `.streak`, `.currentLevel`, `.currentLessonId`, `.displayName`, `.email`, `.isAnonymous`, or `.createdAt` — individually, or bundled into the same write as a valid `settings` change (the bundled case is the one most likely to regress silently and needs its own explicit test, not just the single-field case).
- No client identity can `create` or `delete` a `users/{uid}` document.
- The owning uid can **read** their own `practiceSessions/**` and `aiMemory/**`; **no** client identity — owner included — can write to either, under any field.
- A non-owner authenticated identity is denied read/write on every path scoped to a different uid.
- If the §5.5 illustrative cooldown pattern is ever activated: a test asserting a second `practiceSessions` create within the cooldown window is denied, and one after it elapses is allowed.

## 12. Acceptance criteria

Verbatim, `docs/PRODUCT_SPEC.md` §29:
- "A client cannot write its own `practiceSessions`/progress/AI-memory documents directly (verified by a rules test, not a manual check)"
- "A client cannot write to `lessons` at all"

## 13. Out of scope

- Rules for Phase 2+ collections (`songs`, `AIConversations`) — not specified until `docs/DATA/FIRESTORE_SCHEMA.md` ratifies their shape; when added, follow the same default-deny-then-explicit-allow pattern established here.
- Firestore composite index configuration (`firestore.indexes.json`) — mentioned in `docs/PRODUCT_SPEC.md` §17 (`(userId, completedAt desc)` on `practiceSessions`) but not a security concern, so not covered here.
- Firebase Storage rules — not used in MVP (`docs/PRODUCT_SPEC.md` §16).
- App Check enforcement toggles — configured outside `firestore.rules`, at the console/CLI level — `docs/SECURITY/APP_CHECK.md`.

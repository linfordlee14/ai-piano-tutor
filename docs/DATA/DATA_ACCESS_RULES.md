# Data Access Rules

## 1. Purpose

A read/write authorization matrix: for every Firestore collection and document in the MVP schema, who can read it and who can write it.

## 2. Scope

Every path listed in `FIRESTORE_SCHEMA.md`. Excludes App Check (a request-origin filter, not a per-document authorization mechanism) and Cloud Function-internal input validation (owned by the relevant Function's own spec).

## 3. Alignment Notice — Read Before Relying on This Document

**This document could not be verified against `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` because that file does not yet exist** — it is being written in a parallel batch of work. Everything below is extracted directly from `docs/PRODUCT_SPEC.md` §17 and §19 (cross-checked against the field-ownership table in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15), which is the authoritative product-level statement of these rules. **Once `FIRESTORE_SECURITY_RULES.md` exists, re-diff this document against it** — that file's actual `rules_version = '2'` syntax is the implementation; this document is the product-level statement the implementation must match. Treat any divergence as a bug in one document or the other, not as two independently valid answers.

## 4. The Core Rule

Stated once, because everything else in this document is an elaboration of it (`PRODUCT_SPEC.md` §17, §19; `TYPES_AND_CONTRACTS.md` §4.15):

> `settings` is the only client-writable field under `users/{uid}`. Everything else — `progressSummary`, every `practiceSessions` document, the `aiMemory/summary` document, and all other top-level fields on `users/{uid}` (`displayName`, `email`, `isAnonymous`, `createdAt`, `currentLevel`, `currentLessonId`, `streak`, `xp`) — is Cloud-Functions-only. `lessons` is public-read and never client-writable, by anyone, under any circumstance.

This is the mechanism that closes the "a modified client self-reports a perfect score" cheating vector (§17/§19) and is the concrete implementation of the non-negotiable rule governing this whole documentation batch: **Gemini never determines correctness/scores/XP/streaks — all persisted scoring data is written exclusively by Cloud Functions after deterministic server-side validation, never by the client, never by Gemini.**

## 5. Read/Write Matrix

**Read access note:** `PRODUCT_SPEC.md` §17's security preview states read access uniformly for the whole `users/{uid}` document and everything under it: "everything under `users/{uid}/**` is readable only by that uid." §19's field-by-field breakdown differentiates *write* access, not read access — there is no stated case where a sub-path under a given user is readable by anyone other than that user. The matrix below reflects that: read is uniform-per-owner across the whole document tree; write is what differs field by field.

| Path | Read | Write | Source |
|---|---|---|---|
| `users/{uid}` (document, all fields except `settings`) | Owner (`request.auth.uid == uid`) only | Cloud Functions only (Admin SDK) | `PRODUCT_SPEC.md` §17, §19; `TYPES_AND_CONTRACTS.md` §4.15 |
| `users/{uid}.settings` (`keyboardMapping`, `volume`, `sustainDefault`) | Owner only | Owner (client), own `uid` only | `PRODUCT_SPEC.md` §17, §19; `TYPES_AND_CONTRACTS.md` §4.11, §4.15 |
| `users/{uid}.progressSummary` (embedded field) | Owner only | Cloud Functions only | `PRODUCT_SPEC.md` §17, §19; `TYPES_AND_CONTRACTS.md` §4.7 |
| `users/{uid}/practiceSessions/{sessionId}` | Owner only | Cloud Functions only (`getExerciseFeedback`) — never the client, not even the owner | `PRODUCT_SPEC.md` §17, §19; `TYPES_AND_CONTRACTS.md` §4.6 |
| `users/{uid}/aiMemory/summary` | Owner only | Cloud Functions only | `PRODUCT_SPEC.md` §17, §19; `TYPES_AND_CONTRACTS.md` §4.8 |
| `lessons/{lessonId}` | **Public** — `allow read: if true`, no auth required at all | Admin SDK only (seed script today, admin Function in Phase 3) — no client write, ever | `PRODUCT_SPEC.md` §17, §19 |

`lessons` is the one genuinely public path in the schema — the only read that doesn't require authentication at all, not even an anonymous session. Every other path requires the requester's `uid` to match the document's owner.

## 6. Illustrative Rules Shape

`PRODUCT_SPEC.md` §19 states the rules pattern directly (default deny, then explicit allow); reproduced here as the product-level statement of intent, not as a finished `firestore.rules` file:

```
// lessons — public read, no client write, ever
match /lessons/{lessonId} {
  allow read: if true;
  allow write: if false;
}

// users/{uid} — owner-read everywhere; owner-write only on settings
match /users/{uid} {
  allow read: if request.auth != null && request.auth.uid == uid;
  allow write: if false; // Cloud Functions (Admin SDK) bypass this — see §3 above

  match /practiceSessions/{sessionId} {
    allow read: if request.auth != null && request.auth.uid == uid;
    allow write: if false;
  }

  match /aiMemory/{docId} {
    allow read: if request.auth != null && request.auth.uid == uid;
    allow write: if false;
  }
}
```

This illustrates intent only. Field-level write splitting on `users/{uid}` (allowing `settings` writes while denying everything else on the same document) needs Firestore rules' field-diffing functions (e.g. comparing `request.resource.data` against `resource.data` for all keys except `settings`) — the exact expression is an implementation detail for `FIRESTORE_SECURITY_RULES.md`, not restated here.

## 7. Open Question — Abuse-Prevention Rule Conflicts With the Write-Ownership Model

`PRODUCT_SPEC.md` §19 states, under "Abuse prevention beyond AI": *"a Firestore rule rejecting a new `practiceSessions` write if the learner's previous session completed less than a few seconds ago, blocking naive script-driven XP farming."*

This is in tension with the write-ownership rule stated two paragraphs earlier in the same section: `practiceSessions` is written "only by Cloud Functions (using the Admin SDK, **which bypasses client rules**)." A Firestore Security Rule cannot be the enforcement mechanism for a write path that never goes through client-facing Firestore rules in the first place — Admin SDK writes are not evaluated against `firestore.rules` at all. And per §4/§5 above (and `TYPES_AND_CONTRACTS.md` §4.6's explicit "the client never writes to this collection directly, under any circumstance"), there is no client write path to `practiceSessions` for such a rule to guard.

**Flagging as IMPORTANT, not silently resolving it.** The functional intent is clear and worth keeping — reject a burst of rapid-fire session completions — but the stated mechanism (a Firestore rule) cannot deliver it given the stated write path (Admin-SDK-only). The evident fix is to move this check *inside* `getExerciseFeedback` itself (the Function reads the learner's most recent `practiceSessions.completedAt` before writing a new one, and rejects/short-circuits if it's below the threshold) rather than expressing it as a `firestore.rules` clause. This document proposes that resolution but does not treat it as decided — `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`, once written, should either implement the check in-Function and note why it isn't a rules clause, or explain a write path this document is missing.

## 8. Out of Scope

- The actual `firestore.rules` file and its exact rules-language implementation — `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` (parallel batch).
- App Check configuration (a request-origin filter, applied alongside these rules, not a per-document authorization mechanism) — `PRODUCT_SPEC.md` §16/§19.
- Rate-limiting/quota logic for AI calls specifically (a Cloud Function-internal counter, not a Firestore rule) — `PRODUCT_SPEC.md` §19.
- Retention and deletion-cascade access implications — `DATA_RETENTION.md`.

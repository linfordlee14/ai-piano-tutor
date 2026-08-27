# Privacy

**Status:** Draft v1.0 — security documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §6, §17, §18, §20, §25, §29
**Companion documents:** `docs/SECURITY/SECURITY_REQUIREMENTS.md` (SEC-23, SEC-24, SEC-25), `docs/SECURITY/AUTHORIZATION.md`, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`, `docs/AI/AI_CONTEXT_MODEL.md` (owns the tutor-conversation retention fact this document states as established from `docs/PRODUCT_SPEC.md` directly)

## 1. Purpose

Specify what personal data AI Piano Tutor collects, what it deliberately does not collect or retain, the scope and mechanism of data export and account deletion, and the legal context (POPIA, GDPR) `docs/PRODUCT_SPEC.md` names without a completed legal review — flagged here explicitly rather than assumed handled.

## 2. Scope

In scope: the concrete data inventory (what's in `UserProfile`, `PracticeSession`, `AIMemorySummary`), the analytics privacy boundary, `deleteAccount`/`exportUserData` scope and ordering, and the two flagged legal/completeness gaps.

Out of scope: the Firestore rules that enforce who can *read* this data day-to-day (`docs/SECURITY/FIRESTORE_SECURITY_RULES.md`) — this document covers the data's existence, retention, and subject-rights handling, not its access control.

## 3. Requirements

- Account deletion and data export are **non-negotiable day-one MVP scope**, not later polish — `docs/PRODUCT_SPEC.md` §18 is explicit about this, citing POPIA (the founder is South Africa-based) and a realistic chance of GDPR exposure (EU/UK users).
- A Cloud Function deletes a user's full data subtree on request (`deleteAccount`).
- A Cloud Function exports a user's full data subtree on request (`exportUserData`).
- Analytics collects product-usage events only — screen views, lesson starts/completions, taps — and never tutor conversation content or the specific notes a learner played.

Full requirement statements with IDs: `docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-23, SEC-24, SEC-25.

## 4. Architecture

### 4.1 What "full data subtree" concretely means

`docs/PRODUCT_SPEC.md` §18/§20 says `deleteAccount`/`exportUserData` act on "a user's full data subtree" without enumerating it. Walking `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`'s contracts and `docs/PRODUCT_SPEC.md` §17's collection list, that subtree is concretely:

| Data | Location | Included in export/deletion scope |
|---|---|---|
| `UserProfile` (root doc: `displayName`, `email`, `isAnonymous`, `createdAt`, `currentLevel`, `currentLessonId`, `streak`, `xp`, `settings`, `progressSummary`) | `users/{uid}` | Yes |
| Practice session history | `users/{uid}/practiceSessions/**` | Yes |
| AI memory summary | `users/{uid}/aiMemory/summary` | Yes |
| Firebase Auth identity record (email, provider links, uid, account-creation date) | Firebase Authentication (not Firestore) | Yes — see §4.2, a design decision stated explicitly since `docs/PRODUCT_SPEC.md` doesn't spell it out |
| Analytics events associated with this user | Firebase/Google Analytics | Open question — see §7 |
| Lesson content (`lessons/**`) | `lessons/{lessonId}` | No — not the user's data; shared content |

### 4.2 Design decision: the Firebase Auth record is in scope too

`docs/PRODUCT_SPEC.md` describes the export/deletion scope as "a user's full data subtree," without explicitly saying whether that includes the Firebase Auth identity record itself (email address, linked-provider IDs, account creation timestamp) or only the Firestore documents. **This document treats the Auth record as in scope for both operations**, stated as a deliberate completion of intent rather than left as a fully open question: a "delete my account" request that leaves a live Firebase Auth user with a real email address existing indefinitely is not a genuine account deletion by any reasonable reading of that phrase or of POPIA/GDPR's data-subject-rights intent; an "export my data" request that omits the person's own email and signup date from their own export is an incomplete export by the same standard. This is a low-controversy inference, not a policy fork — noted here for implementer clarity, not raised as one of the open questions in §7.

### 4.3 `deleteAccount` — recommended operation order

Not specified by `docs/PRODUCT_SPEC.md` (`docs/ARCHITECTURE/DATA_FLOW.md` §2 explicitly excludes this flow from its trace as "straightforward request/response, not architecturally interesting enough" — this document is where the ordering detail that flow doesn't cover belongs). Recommended order, and the reasoning for it:

1. **Delete subcollections first:** `users/{uid}/practiceSessions/**`, then `users/{uid}/aiMemory/summary`. Firestore does not cascade-delete subcollections when a parent document is deleted — deleting the root document first would silently orphan these, leaving data behind that the operation claims to have removed.
2. **Delete the root `users/{uid}` document** once its subcollections are confirmed gone.
3. **Delete the Firebase Auth record last**, only once the Firestore-side deletion above has completed. Reasoning: while the Auth record still exists, the caller still holds a valid ID token and can prove ownership to retry the operation if it partially fails partway through steps 1–2. Deleting the Auth record first would strand a retry — the caller would lose the ability to authenticate as the very account whose leftover data needs cleaning up.
4. The Function should be **idempotent/retry-safe** — safe to invoke again if it partially failed (e.g., subcollections deleted but the root document delete failed) — since a user-facing "delete my account" action should not require support intervention to finish cleanly.

Deleting a `practiceSessions` subcollection with many documents is not a single atomic Firestore operation — it requires batched/paginated deletes (read a batch of document IDs, delete them, repeat until the subcollection is empty), a standard pattern for bulk-deleting a Firestore subcollection, worth flagging because it's an easy detail to under-scope in a first implementation pass.

### 4.4 `exportUserData` — scope and format

Same data scope as §4.1 (Firestore subtree plus the Auth identity record). `docs/PRODUCT_SPEC.md` does not specify the export **format** (a downloadable file? an emailed link? a direct callable-function response payload?). **PROPOSED:** a structured JSON document mirroring the shapes already defined in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` (`UserProfile`, an array of `PracticeSession`, the `AIMemorySummary`, plus the Auth fields from §4.2) — every underlying record is already JSON-shaped Firestore data, so JSON is the lowest-effort, most complete, and most portable choice; it's also directly useful for a POPIA/GDPR-style "provide the data in a commonly used, machine-readable format" expectation without extra transformation work. This is a proposal for the implementer to ratify, not a fact `docs/PRODUCT_SPEC.md` states.

## 5. Data structures

### 5.1 What's collected

| Category | Fields/shape | Source |
|---|---|---|
| Account/profile | `UserProfile` — `displayName`, `email`, `isAnonymous`, `createdAt`, `currentLevel`, `currentLessonId`, `streak`, `xp`, `settings`, `progressSummary` | `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15 |
| Practice history | `PracticeSession` records — per-attempt `verdict` (accuracy/timing/completeness/consistency breakdown), `deviceType`, timestamps | §4.6 |
| AI-derived learner model | `AIMemorySummary` — `skillLevel`, `weakNotes`, `weakChords`, `strengths`, `currentGoal` | §4.8 |
| Product-usage analytics (aggregate/event-level, not content) | Screen views, lesson starts/completions, taps, AI tutor call *counts* | `docs/PRODUCT_SPEC.md` §25 |

### 5.2 What's NOT collected or retained

| NOT collected/retained | Why |
|---|---|
| Raw tutor conversation transcripts | `AIMemorySummary` is "rewritten in place each time it's recomputed — never a growing transcript" (`docs/PRODUCT_SPEC.md` §17/§31); `TutorResponse` is "never persisted as a growing log... nothing beyond the `AIMemorySummary` aggregate" (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.10). A learner's question text and the tutor's answer exist only for the duration of that single request/response — see `docs/AI/AI_CONTEXT_MODEL.md` (parallel batch, not yet written) for the full retention-model treatment; this document states the fact as already established by `docs/PRODUCT_SPEC.md`/`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` directly, independent of that document landing. |
| Specific notes played, sent to analytics | `docs/PRODUCT_SPEC.md` §25's privacy boundary, verbatim below (§8) — note/chord data lives only in Firestore under the user's own record, never duplicated into the analytics stream. |
| Tutor conversation content, sent to analytics | Same boundary. |
| Raw `NoteEvent[]` streams, retained beyond scoring | `NoteEvent` is explicitly "never persisted directly" (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1) — what's retained is the computed `ExerciseVerdict`, not the raw input that produced it. |

## 6. Interfaces/contracts

`deleteAccount` and `exportUserData` (`docs/PRODUCT_SPEC.md` §20): both are callable Cloud Functions taking no meaningful input beyond the auth context itself (`context.auth.uid` identifies the target — there is no "delete a different user's account" parameter, because no such capability exists in MVP, §7 below). Both follow the standard `{ ok: true, data }` / `{ ok: false, error }` response contract (`docs/PRODUCT_SPEC.md` §20); for `exportUserData`, `data` carries the export payload itself (or a reference to it, depending on the format decision in §4.4); for `deleteAccount`, `data` is a simple confirmation.

## 7. Dependencies

- `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — the exact shapes of everything enumerated in §5.1.
- `docs/AI/AI_CONTEXT_MODEL.md` (parallel batch, not yet written) — the fuller treatment of what "AI memory" retains; this document's §5.2 claim is sourced directly from `docs/PRODUCT_SPEC.md`/`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` and should be checked for consistency once that document lands.
- `docs/SECURITY/AUTHORIZATION.md` — confirms only the owning `context.auth.uid` can trigger `deleteAccount`/`exportUserData` for themselves; no admin override exists in MVP (§4.5 of that document) — there is intentionally no "support agent deletes/exports a user's account on their behalf" path in MVP. If that capability is ever needed, it requires its own authorization design, not an extension of this one.

## 8. Security considerations

**Analytics privacy boundary, verbatim (`docs/PRODUCT_SPEC.md` §25):** *"collect product-usage events only (screen views, lesson starts/completions, taps) — never send tutor conversation content or the specific notes a learner played into the general analytics stream. That data already lives correctly-scoped in Firestore under the user's own record; duplicating it into an analytics tool would only create a second, less-protected copy."*

Only `context.auth.uid` matching the target account may trigger deletion/export of that account's data — enforced the same way as every other per-user Cloud Function (`docs/SECURITY/AUTHENTICATION.md` §4.5/§6).

### ⚠ Open question — is analytics event data itself in the export/deletion scope? (IMPORTANT, not BLOCKING)

`docs/PRODUCT_SPEC.md` §25 is explicit that analytics events don't carry sensitive *content* (§8 above), but says nothing about whether analytics events are **linked to the user's Firebase uid** (e.g., via Firebase Analytics' `setUserId()`). If they are — which is a very ordinary thing to do for basic funnel/retention analysis, and nothing in §25 rules it out — then those events plausibly count as personal data under a POPIA/GDPR-style regime even though their *content* is harmless, purely because they're identifiable to a specific person. `docs/PRODUCT_SPEC.md`'s export/deletion Cloud Functions are scoped to "the Firestore subtree" (§4.1 above) with no mention of also purging/exporting linked analytics events, and Google Analytics does have its own data-deletion API surface for exactly this kind of request, separate from Firestore. This document does not resolve whether analytics data needs to be included — it flags that the question exists and is currently unaddressed by `docs/PRODUCT_SPEC.md`. **Classification: IMPORTANT** — related to, and should be resolved alongside, the broader legal-review recommendation in §9, not before M0, but before a real launch with live analytics collection.

## 9. Performance considerations

Bulk-deleting a `practiceSessions` subcollection with many documents requires batched/paginated deletes, not a single call (§4.3) — for a long-tenured learner this could mean dozens or hundreds of documents; the deletion Function should be written expecting that scale from the start rather than as a later fix once someone's account is slow to delete.

## 10. Error handling

- **Partial deletion failure:** see §4.3's ordering and idempotency requirement — a `deleteAccount` retry after a partial failure must be safe to run again without erroring on already-deleted subcollections.
- **Export requested for an account with no data yet** (e.g., an anonymous session that signed up but hasn't played anything): the export should still succeed, returning an otherwise-empty-but-structurally-complete payload (empty `practiceSessions` array, default-valued `UserProfile`), not an error — an empty result is a valid answer to "export my data," not a failure state.

## 11. Testing requirements

- Rules/emulator test: only the owning uid can trigger `deleteAccount`/`exportUserData` for their own account; no identity can trigger either for a different uid.
- A test asserting deletion actually removes `practiceSessions` and `aiMemory`, not just the `users/{uid}` root document — this is a real, easy-to-miss bug class ("deleted the summary doc but forgot the subcollections") worth its own explicit assertion rather than trusting the root-doc deletion to imply the rest.
- A test asserting a retried `deleteAccount` call after a simulated partial failure completes cleanly rather than erroring.
- A test asserting an `exportUserData` call for a fresh/empty account succeeds with an empty-but-valid payload rather than failing.

## 12. Acceptance criteria

Derived from `docs/PRODUCT_SPEC.md` §18's "non-negotiable day-one scope, not later polish" framing:
- Account deletion and data export ship as part of MVP, no later than the M5 hardening milestone (`docs/PRODUCT_SPEC.md` §28) — not deferred to Phase 2+.
- A deletion request removes 100% of the user's Firestore subtree (root document, `practiceSessions`, `aiMemory`) and the Firebase Auth record.
- An export request returns 100% of the same scope, in a complete, structured format.
- An audit of every analytics event call site confirms no event payload contains note/chord/theory-play content or tutor conversation text (mirrors `docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-25's acceptance criterion).

## 13. Out of scope

- A full legal compliance review/certification of POPIA and GDPR applicability — **flagged, not performed, by this document.**

  ### ⚠ Open question — POPIA/GDPR legal review (IMPORTANT, not BLOCKING)

  `docs/PRODUCT_SPEC.md` §18 names both regimes as applicable — POPIA because the founder/business is South-Africa-based, GDPR as a realistic exposure given a plausible EU/UK user base — and treats data export/deletion as the non-negotiable technical response to that exposure. What §18 does not include, and what this document does not attempt to supply, is an actual legal review confirming that the technical measures in this document set (export, deletion, the collection/retention boundaries in §5) are *sufficient* for either regime — things like lawful-basis documentation, a privacy policy with the specific disclosures either regime requires, breach-notification procedures, or a Data Processing Agreement posture if any third-party processor (Google/Firebase, Gemini) requires one. **Recommendation:** commission a legal review before a real (public, user-data-collecting) launch — not before M0, since no real user data exists yet at that milestone and the review's findings could still meaningfully change downstream design. **Classification: IMPORTANT, not BLOCKING** — this does not block starting engineering work on M0 through M4, but should be resolved before M5 (hardening & launch readiness) is called complete.

- A formal cookie-consent/DPA-style GDPR mechanics layer beyond the data-subject-rights (access/deletion) work already covered here.
- CCPA or other jurisdiction-specific regimes beyond POPIA/GDPR — `docs/PRODUCT_SPEC.md` names only these two.
- The Firestore rules that gate day-to-day read access to this data — `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.

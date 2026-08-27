# Data Retention

## 1. Purpose

Covers three areas `PRODUCT_SPEC.md` does not fully specify: (1) whether the append-only `practiceSessions` log has a retention/archival policy, (2) what happens to an anonymous user's data if they never sign up, and (3) exactly what an account-deletion cascade must touch to be complete. Recommendations here are marked **PROPOSED** where `PRODUCT_SPEC.md` is silent; facts are cited to their source section.

## 2. Scope

Data lifecycle after creation: how long it lives, what deletes it, and what "delete" must actually reach. Excludes the collection layout itself (`FIRESTORE_SCHEMA.md`), who can read/write while data exists (`DATA_ACCESS_RULES.md`), and the entity relationships driving what a cascade must follow (`DATA_MODEL.md`).

## 3. PracticeSession Retention — PROPOSED, Open

**`PRODUCT_SPEC.md` specifies no retention or archival policy for `practiceSessions`.** It is described only as an "append-only log" (§17) that "powers Progress/Stats and weak-area detection" — nothing in the document addresses whether it should ever be pruned, archived, or capped.

**Proposal (PROPOSED — needs product sign-off, not extracted from spec):** no automatic deletion in MVP. Expected volume is low — a few hundred early users (`PRODUCT_SPEC.md` §31's own framing), each producing at most a handful of small documents per practice session, in short irregular sessions (`PRODUCT_SPEC.md` §4). At that scale, Firestore storage cost is negligible, consistent with §31's general framing that Firebase costs outside Gemini spend are "likely near-zero" at MVP scale. Revisit only if/when volume becomes a real storage-cost concern.

**Trigger for revisiting (PROPOSED):** don't wait for a scheduled calendar review — extend the spend-instrumentation practice `PRODUCT_SPEC.md` §31 already recommends for Gemini ("instrument spend from day one... not just a monthly bill review") to cover Firestore storage/read volume too (see `docs/OPERATIONS/MONITORING.md` and `docs/OPERATIONS/COST_MONITORING.md`). Treat a meaningful, sustained rise in that number — not a specific document count picked in advance without usage data — as the signal to design an actual retention/archival policy (e.g. summarizing and cold-archiving sessions older than N months once N and the storage math are grounded in real numbers).

## 4. Anonymous, Never-Linked User Data — OPEN QUESTION, Not Addressed by PRODUCT_SPEC.md

`PRODUCT_SPEC.md` supports an anonymous try-first session (§6, §18) and describes the anonymous-to-linked upgrade path (`linkWithCredential`, §18) in detail. **It never addresses the case where that upgrade never happens** — a visitor plays the piano anonymously and simply never returns, never signs up, never links.

That anonymous session still has a real Firebase Auth `uid` (§18/§19 are explicit that anonymous users "carry a real Firebase uid" and the same rules apply uniformly) and, depending on how far the visitor got, a real `users/{uid}` document, possibly `practiceSessions`, possibly an `aiMemory/summary`. With no cleanup mechanism, that full subtree persists indefinitely with no confirmed owner ever coming back to claim, use, or consent to retaining it.

**Why this matters, concretely:**

- **Cost.** An unbounded, ever-growing set of orphaned anonymous subtrees is a different cost shape than "a few hundred early users" (§31) — it scales with every visitor who ever tries the piano and leaves, not with actual signups.
- **Privacy/compliance.** `PRODUCT_SPEC.md` §18 is explicit that this product is built "for South African users (POPIA applies) with a realistic chance of EU/UK users too (GDPR)," and treats account deletion/export as "non-negotiable day-one scope" specifically because of that. An orphaned-anonymous-data policy is the same category of concern — indefinite retention of data tied to a persistent device/session identifier, with no retention justification once it's clear the person isn't returning — and `PRODUCT_SPEC.md` gives it no treatment at all. **Cross-reference `docs/SECURITY/PRIVACY.md`** (does not yet exist in this documentation batch) — this needs to be resolved in coordination with that document once it's written; this document should be re-checked against it at that time.

**Candidate resolutions (PROPOSED — presented as options, not a decision):**

| Option | Mechanism | Trade-off |
|---|---|---|
| A. No automatic cleanup in MVP | Same default as §3's `practiceSessions` proposal — do nothing until volume/compliance pressure is real | Simplest; defers the privacy-posture question rather than answering it — the weaker option given §18's own POPIA/GDPR framing |
| B. Scheduled cleanup Cloud Function | A scheduled (cron) Cloud Function — a capability already explicitly named in the architecture (`PRODUCT_SPEC.md` §16: "Cloud Functions (2nd gen) — Gemini proxy **+ scheduled jobs**") — that deletes anonymous `UserProfile` subtrees with no activity for N days | Directly addresses both cost and privacy exposure; needs a concrete inactivity threshold N, which `PRODUCT_SPEC.md` gives no basis for choosing today |

**Flagged as an OPEN QUESTION requiring an explicit product decision before a real (non-internal-testing) launch** — not something this document resolves on its own authority. Given §18's framing that this exact category of concern ("non-negotiable... not later polish") is a stated priority for this product, treat this as needing resolution before public launch, in the same spirit as (though not with the identical urgency of) account deletion/export itself.

## 5. Account Deletion Cascade

`PRODUCT_SPEC.md` §18 states, as fact, not proposal: *"a Cloud Function that deletes or exports a user's full data subtree on request. This is non-negotiable day-one scope."* §20 names the callable functions: `deleteAccount`, `exportUserData`.

### 5.1 What "Full Data Subtree" Must Include

Based on the schema established in `FIRESTORE_SCHEMA.md`, a complete `deleteAccount` must reach every one of the following:

| Path | Included in cascade? | Why |
|---|---|---|
| `users/{uid}` (document, including embedded `settings` and `progressSummary`) | Yes | The root profile document |
| `users/{uid}/practiceSessions/*` (all documents) | Yes | User data, per-user subcollection |
| `users/{uid}/aiMemory/summary` | Yes | User data |
| `lessons/*` | **No** | Public content, not part of any user's subtree — correctly excluded |
| The Firebase Auth user record itself (not a Firestore path) | Yes — see note below | Not part of "Firestore subtree" literally, but required for the deletion to actually be complete |

**Note (implementation clarification, not a literal spec quote):** "full data subtree" as `PRODUCT_SPEC.md` states it is naturally read as the Firestore data. A `deleteAccount` implementation that stops there is incomplete: the Firebase Auth user record (`admin.auth().deleteUser(uid)`) must also be removed, or the same `uid` continues to be a valid, authenticatable identity with no data behind it — an inconsistent, and for export/deletion-on-request purposes, arguably non-compliant state.

**Operational gotcha worth flagging explicitly:** Firestore does not recursively delete subcollections when a parent document is deleted. Deleting `users/{uid}` alone does not remove `practiceSessions/*` or `aiMemory/summary` — those become orphaned documents at a path whose parent no longer exists, still fully readable/queryable by anyone who already knows or can query for the path pattern, and still billed for storage. A correct `deleteAccount` implementation must explicitly delete each subcollection's documents (a recursive-delete utility, or the `firebase-tools` recursive-delete helper, or manual batched deletes) *before or alongside* deleting the parent document — never rely on parent deletion alone. This is not stated in `PRODUCT_SPEC.md` and is exactly the kind of detail that could silently fail a POPIA/GDPR deletion request if missed — flagged here as IMPORTANT for whoever implements `deleteAccount`.

### 5.2 Proposed Operation Order (PROPOSED — PRODUCT_SPEC.md Does Not Specify Sequencing)

1. Verify caller identity via Firebase ID token (standard for every callable Function, per `PRODUCT_SPEC.md` §18/§19).
2. Recursively delete the `practiceSessions` subcollection.
3. Delete the `aiMemory/summary` document.
4. Delete the `users/{uid}` document.
5. Delete the Firebase Auth user record.
6. Return success to the caller.

Steps 2–4 delete Firestore data before the Auth record so that, if the Function fails partway through, the `uid` is still valid and the operation can be safely retried against a known, identifiable (if partially deleted) subtree, rather than leaving orphaned Firestore data with no Auth record left to retry against. The Function should be written idempotently — safe to call again if a prior attempt partially completed — rather than assuming a single atomic pass always succeeds; Firestore has no cross-collection transaction primitive that would make steps 2–5 atomic as one unit.

## 6. Out of Scope

- The exact retention/deletion implementation (Cloud Function code, recursive-delete utility choice) — this is documentation, not the implementation itself.
- The broader privacy/compliance posture beyond data lifecycle (consent flows, data export format) — `docs/SECURITY/PRIVACY.md` (parallel batch, does not yet exist).
- Cost-monitoring mechanics referenced as the trigger in §3 — `docs/OPERATIONS/COST_MONITORING.md`, `docs/OPERATIONS/MONITORING.md`.

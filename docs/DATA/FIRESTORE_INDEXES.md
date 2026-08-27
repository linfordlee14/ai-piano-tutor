# Firestore Indexes

## 1. Purpose

States the one composite index MVP needs, why it's needed, and the process for adding more as query patterns grow — explicitly so nobody pre-builds speculative indexes "just in case."

## 2. Scope

Composite index requirements only. Firestore's automatic single-field indexes are not covered here — there's nothing to configure; Firestore creates them automatically for every field, ascending and descending, unless explicitly exempted.

## 3. The MVP Composite Index

`PRODUCT_SPEC.md` §17: *"a single composite index on `practiceSessions` for `(userId, completedAt desc)` covers the stats screen; Firestore's automatic single-field indexes handle the rest at this scale."*

| Collection | Fields | Order | Purpose |
|---|---|---|---|
| `practiceSessions` | `userId`, `completedAt` | `userId` ascending, `completedAt` descending | Progress/Stats screen (`PRODUCT_SPEC.md` §17, §25) |

This is defined in `firestore.indexes.json` (see project structure, `PRODUCT_SPEC.md` §27) and deployed via the Firebase CLI alongside `firestore.rules` — see `docs/OPERATIONS/DEPLOYMENT.md`.

### 3.1 Why a Composite Index, Specifically

Firestore requires a composite index whenever a query combines an equality filter on one field with a sort (`orderBy`) on a different field — exactly the shape of "this user's sessions, most recent first." Firestore's automatic single-field indexes cannot serve that combination on their own; that's precisely the case explicit composite-index configuration exists for.

### 3.2 Why `userId` Is a Field on the Document at All, When the Path Already Scopes It

`PracticeSession` documents already live at `users/{uid}/practiceSessions/{sessionId}` — the path itself scopes every document to one user. A direct per-user subcollection read (`users/{uid}/practiceSessions`, ordered by `completedAt`) would need only Firestore's automatic single-field index on `completedAt`, with no composite index and no `userId` field required at all.

`TYPES_AND_CONTRACTS.md` §4.6 nonetheless defines `userId` as a required field directly on the `PracticeSession` document, redundant with its own path. The composite index `PRODUCT_SPEC.md` §17 calls for only makes sense against that redundant field if the actual query is (or may need to be) a **collection-group query** — `collectionGroup('practiceSessions').where('userId', '==', uid).orderBy('completedAt', 'desc')` — which flattens across every user's `practiceSessions` subcollection and therefore needs a stored field to filter on, since collection-group queries can't rely on the parent path alone.

**This isn't stated explicitly as the query mechanism anywhere in `PRODUCT_SPEC.md` or `TYPES_AND_CONTRACTS.md`** — it's the most coherent explanation connecting the redundant `userId` field to the specified composite index, offered here so the implementer understands *why* the index has this shape rather than treating it as an arbitrary instruction. Whether the stats screen actually issues a collection-group query or a direct subcollection query (which would make this specific composite index unused, though harmless to have defined) is an implementation decision, not one this document prescribes. Define the index as specified either way — it costs nothing idle and `PRODUCT_SPEC.md` calls for it explicitly.

## 4. Everything Else: Automatic Single-Field Indexes

Firestore creates a single-field index automatically for every field in every document, in both ascending and descending order, with no configuration required. At MVP's query patterns and data volume, this covers every other read path in the schema — `lessons` queries (by `level`, for the linear lesson-path ordering), `users/{uid}` document-by-ID lookups, and the `aiMemory/summary` fixed-path read. None of these need a composite index.

## 5. Process for Adding New Indexes

**Do not pre-build speculative indexes.** Firestore's own failure mode for a missing composite index is self-documenting: a query that needs an index not yet defined fails with an error containing a direct link to create exactly that index, pre-filled, in the Firebase console. The correct process, as query patterns grow beyond MVP:

1. Write the query the feature actually needs.
2. If it fails locally or in the emulator with a missing-index error, follow the provided link (or replicate the equivalent entry) into `firestore.indexes.json`.
3. Deploy the updated index alongside the next release via the Firebase CLI.
4. Do not add a composite index for a query pattern that doesn't exist yet, no matter how likely it seems — this is the same "don't add infrastructure before there's a reason" discipline `PRODUCT_SPEC.md` applies elsewhere (e.g. §21's "choose only if needed" for state management, §32's "deliberately excluded" dependency list).

## 6. Out of Scope

- The read/write authorization matrix — `DATA_ACCESS_RULES.md`.
- Deployment mechanics for `firestore.indexes.json` itself — `docs/OPERATIONS/DEPLOYMENT.md`.
- Query-level performance beyond index existence (e.g. a pagination strategy for a growing `practiceSessions` log) — not specified in `PRODUCT_SPEC.md`; the log's unbounded growth itself is covered as an open question in `DATA_RETENTION.md`.

# Authentication

**Status:** Draft v1.0 — security documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §5, §6, §10, §18; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15
**Companion documents:** `docs/SECURITY/SECURITY_REQUIREMENTS.md` (SEC-01, SEC-02, SEC-03), `docs/SECURITY/AUTHORIZATION.md`, `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`, `docs/ARCHITECTURE/DATA_FLOW.md`

## 1. Purpose

Define how AI Piano Tutor establishes and verifies learner identity: which providers exist, how a try-first anonymous session becomes a real linked account without losing progress, and how every server-side operation trusts that identity. This document owns **identity** — who someone is. `docs/SECURITY/AUTHORIZATION.md` owns **what they can then do** with that identity.

## 2. Scope

In scope: Firebase Auth provider configuration, the anonymous-to-linked upgrade flow, server-side ID token verification, provider extensibility, and the unresolved linking-conflict edge case.

Out of scope: what an authenticated identity is authorized to read/write (`AUTHORIZATION.md`, `FIRESTORE_SECURITY_RULES.md`), App Check (`APP_CHECK.md`, a separate attestation layer that runs alongside auth, not part of it), account deletion/export (`PRIVACY.md`).

## 3. Requirements

- **Providers (MVP):** Anonymous (try-first), Google Sign-In, Email/Password — `docs/PRODUCT_SPEC.md` §5, §6, §18.
- **Anonymous-to-linked upgrade:** `linkWithCredential`, so a try-first visitor's in-session progress survives signup rather than resetting to zero — §18.
- **Server-side trust:** every Cloud Function verifies the caller's Firebase ID token; a client-supplied `uid` is never trusted on its own — §18, §19.
- **Provider extensibility:** Firebase Auth's provider list is additive by default — no extra design work is needed now to add a provider later (e.g. Apple Sign-In) — §18.
- **Account deletion/export are day-one MVP scope**, not part of this document — see `docs/SECURITY/PRIVACY.md` for the requirement, owned there because it's a privacy concern first and an auth-adjacent mechanism second.

Full requirement statements with IDs: `docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-01, SEC-02, SEC-03.

## 4. Architecture

### 4.1 Provider summary

| Provider | Firebase Auth method | Learner-facing entry point | Notes |
|---|---|---|---|
| Anonymous | `signInAnonymously()` | Landing screen, "Try the piano now" (`docs/PRODUCT_SPEC.md` §10) | Issues a **real** Firebase uid immediately. `isAnonymous: true` on the resulting `UserProfile` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15). This is not a guest mode with no identity — it is a full identity with a temporary recoverability profile. See `docs/SECURITY/AUTHORIZATION.md` for what this means for read/write parity. |
| Google Sign-In | `GoogleAuthProvider` + `signInWithPopup`/`signInWithRedirect`, or `linkWithCredential` from an anonymous session | Sign in / Sign up screen, one-tap | Additive provider — no schema impact. |
| Email/Password | `EmailAuthProvider` + `createUserWithEmailAndPassword` / `signInWithEmailAndPassword`, or `linkWithCredential` | Sign in / Sign up screen | Same as above. |
| *(Future, additive)* Apple Sign-In, etc. | Standard Firebase Auth provider registration | Not designed against yet — no MVP screen references it | `docs/PRODUCT_SPEC.md` §18 explicitly calls this out as requiring no data-model change when it happens. |

### 4.2 The try-first anonymous flow

1. A new visitor lands on the app (pre-auth) and taps **"Try the piano now"** (`docs/PRODUCT_SPEC.md` §10 — this is the primary landing action, by design, so the section 2 north-star metric — a first 5-note melody within the first session — is never gated behind a signup form).
2. Client calls `signInAnonymously()`. Firebase Auth issues a real Firebase UID and an ID token; `isAnonymous: true`.
3. A Cloud Function (an Auth `onCreate` trigger, using the Admin SDK) provisions the initial `users/{uid}` document with default `UserProfile` field values (`currentLevel: 1`, `xp: 0`, `streak: {count: 0, ...}`, default `settings`, `displayName: null`, `email: null`, empty `progressSummary`). Document creation itself is Function-written, consistent with every other field on this document being Function-written except `settings` — there is no client-side "bootstrap" write path, even for the very first document. See `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` §4.1 for the rules-side rationale for denying client `create` on this path.
4. The visitor plays piano, attempts lessons. Every score-affecting write flows through the normal Cloud-Function-written path (`docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-10/SEC-11) — anonymous users are **not** a special case for data rules (`docs/SECURITY/AUTHORIZATION.md`).

### 4.3 The anonymous-to-linked upgrade flow

This is the mechanism that makes SEC-02 true — the specific ordering matters.

1. The still-anonymous, still-signed-in visitor chooses to sign up (Google or email/password) from the Sign in / Sign up screen.
2. Client obtains an `AuthCredential` for the chosen provider — `GoogleAuthProvider.credential(idToken)` for Google, or `EmailAuthProvider.credential(email, password)` for email/password — **without signing into that provider as a fresh, separate session.**
3. Client calls `currentUser.linkWithCredential(credential)` on the **currently-signed-in anonymous user object.** This is the critical step: linking is initiated *from* the anonymous session, not by signing into the new provider first and reconciling identities afterward.
4. On success: the Firebase **uid does not change.** This is the entire mechanism by which progress "survives" linking — there is no data migration, because every document the visitor's session already touched (`users/{uid}`, `users/{uid}/practiceSessions/**`, `users/{uid}/aiMemory/**`) was always keyed by this same uid. `isAnonymous` flips to `false`; `email`/`displayName` populate from the linked provider. (Which Function updates `isAnonymous`/`email`/`displayName` on the `UserProfile` document, and by what trigger, is an implementation detail for `functions/src/account/` — not specified further here since it doesn't change the security posture: it is Function-written either way, per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15.)
5. On failure with a credential-already-in-use error (`auth/credential-already-in-use` for OAuth providers like Google, `auth/email-already-in-use` for email/password): this is the **linking conflict** case. See §7 below — this is explicitly unresolved, not silently picked.

### 4.4 Returning-user sign-in

A learner who already linked a provider on a previous device/session signs in directly (`signInWithPopup`/`signInWithRedirect` for Google, `signInWithEmailAndPassword` for email) — no linking step, since they're not starting from an anonymous session. Their existing `uid` and all its data resume exactly as left off (`docs/PRODUCT_SPEC.md` §5: "As a returning learner, I want to resume exactly where I left off").

### 4.5 Server-side token verification — the precise mechanism

`docs/PRODUCT_SPEC.md` §18 states this as "every Cloud Function verifies the caller's Firebase ID token via the Admin SDK." Stated precisely, to avoid two plausible misreadings:

- AI Piano Tutor's entire server API surface is **Callable Cloud Functions** (`onCall`), not raw HTTPS functions (`docs/PRODUCT_SPEC.md` §20). The Callable Functions framework **automatically** verifies the caller's Firebase ID token — using the Admin SDK under the hood — before handler code ever runs, and populates `context.auth` with the verified identity. A handler does not need to, and must not, hand-roll its own `admin.auth().verifyIdToken(...)` call on a token pulled out of the request payload — that would be redundant work solving an already-solved problem, and it invites the exact mistake this requirement exists to prevent (see next bullet).
- What handler code **must** do, and what actually satisfies SEC-03, is two things: (a) treat `context.auth.uid` as the *only* source of truth for "who is calling this," and (b) explicitly guard on `context.auth != null` as the very first thing the handler does, rejecting immediately if it's absent. **A handler must never read a `uid` field out of the request payload (`data`) and act on it** — even "just to be explicit" or "to make testing easier" — because that field is fully attacker-controlled and defeats the entire point of token verification. This is the concrete form SEC-03's "a client-supplied uid is never trusted on its own" takes in code.

### 4.6 Provider extensibility

Firebase Auth's provider list is additive: adding a new provider (Apple Sign-In is the named future candidate) means registering it in the Firebase console/config and adding a corresponding sign-in/link button — it does not require a `UserProfile` schema change, a new Firestore rule, or a new Cloud Function, because every provider ultimately resolves to the same thing this document already handles: a Firebase uid, optionally linked from an anonymous session. No additional design work is needed now to keep this door open (`docs/PRODUCT_SPEC.md` §18).

## 5. Data structures

No new types. References `UserProfile.isAnonymous`, `UserProfile.email`, `UserProfile.displayName` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15 — all three are explicit-null-capable for a pre-linking anonymous user, never omitted).

## 6. Interfaces/contracts

**Every callable Function's auth contract**, restated as an implementation checklist (this is the concrete form of SEC-03/SEC-04):

1. Reject if `context.auth` is null/undefined.
2. Use `context.auth.uid` as the identity for every subsequent Firestore read/write in this invocation.
3. Never read an identity field from `data` (the request payload).
4. This check is identical for every Function, regardless of whether the caller is anonymous or linked — there is no branch on `context.auth.token.firebase.sign_in_provider` that relaxes or tightens this guard.

## 7. Security considerations

### ⚠ Open question — anonymous-to-linked conflict resolution policy (unresolved, not silently picked)

`docs/PRODUCT_SPEC.md` §10 names the state that must be handled — *"linking conflict if the email is already registered"* — but does not specify the resolution **policy**. Firebase Auth itself does not resolve this either: `linkWithCredential` simply throws (`auth/credential-already-in-use` / `auth/email-already-in-use`) and performs no linking or merging on its own — the app must decide what happens next. This document deliberately does not pick an answer. The candidate policies, laid out for whoever ratifies this:

| Option | Behavior | Trade-off |
|---|---|---|
| **A — Keep the anonymous session's progress** | Reject the link; tell the learner this email already has an account; offer "sign in to that account instead" (anonymous progress is then abandoned unless a manual merge is built) or "keep browsing as guest" | Simple, but a returning user with a real account may be confused why their history doesn't show up, and the anonymous session's work is lost if they proceed to the existing account |
| **B — Keep the existing account's progress** | Sign the learner into the existing account; discard/abandon the anonymous uid and its subtree (candidate for scheduled deletion per data-retention policy) | Simplest technically, but user-hostile if the visitor did meaningful work in the anonymous session before hitting "sign up" — the exact scenario the north-star metric (§2) is built around |
| **C — Merge the two progress states** | A Cloud Function reconciles both subtrees — e.g. union of completed lessons, max `bestScore` per lesson, max/sum of `xp`, later `streak` | Most user-friendly, but merge semantics for `AIMemorySummary` (whose `weakNotes`/`weakChords`/`currentGoal` wins, or do they merge?) are themselves ambiguous and would need their own design pass |
| **D — Ask the learner** | Surface the conflict explicitly in the UI and let the learner choose between B and (a data-loss-accepting version of) A | Most transparent, but is a product/UX decision as much as a technical one, and still needs B or a version of C as its underlying mechanism once chosen |

**Regardless of which policy is chosen, error handling for the specific `auth/credential-already-in-use` / `auth/email-already-in-use` codes must exist** — even doing nothing here would surface a raw SDK error to the learner, which violates the "never a dead end" principle (`docs/PRODUCT_SPEC.md` §10, §34). Only the *policy* of what happens after catching that error is open.

**Classification:** IMPORTANT — this does not block starting engineering on M0–M2 (piano, engine, first lesson, none of which touch linking), but it must be resolved before the Sign in/Sign up screen (`docs/PRODUCT_SPEC.md` §10) and the M2–M4 auth-adjacent work ship, since the linking button cannot be built without knowing what it does on conflict.

### Other considerations

- Anonymous users are a genuine identity, not a lesser one — see `docs/SECURITY/AUTHORIZATION.md` for the explicit statement that anonymous and linked users have identical read/write parity on their own uid's data. The only difference is *recoverability*: an anonymous session tied to one device/browser storage has no credential to sign back in with if that storage is lost.
- Token verification happening automatically inside the Callable Functions framework is a platform guarantee, not a per-Function implementation choice — but the discipline in §4.5/§6 (never trust a payload-supplied `uid`) is not automatic and must be code-reviewed for on every new Function.

## 8. Dependencies

- Firebase Authentication SDK (client), Firebase Admin SDK (server, via Callable Functions' built-in verification).
- `docs/SECURITY/AUTHORIZATION.md` — what an authenticated identity can do.
- `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` — `request.auth.uid` is the primitive every rule in that document is built from.

## 9. Error handling

- **`linkWithCredential` conflict** (`auth/credential-already-in-use`, `auth/email-already-in-use`): caught explicitly; UX resolution is the open question in §7. Never surfaced as a raw/opaque SDK error.
- **Network failure during any auth operation:** the piano itself must keep working regardless (`docs/PRODUCT_SPEC.md` §2's "the learner is never blocked" principle) — a sign-in/link failure degrades the *account* features, not the already-loaded local session's ability to play piano and progress through the currently-loaded lesson content.
- **Expired/invalid ID token reaching a Cloud Function:** rejected by the Callable Functions platform before handler code runs; the client receives the typed `{ ok: false, error: { code, message } }` shape (`docs/PRODUCT_SPEC.md` §20), never a raw 401.

## 10. Testing requirements

- Firebase Local Emulator Suite's Auth emulator for all sign-in/link/sign-out flows, integrated with Firestore/Functions emulators for the full round trip (`docs/PRODUCT_SPEC.md` §26).
- An explicit test asserting that a Function payload containing a forged `uid` field is ignored — the Function acts only on `context.auth.uid`, even when the two differ.
- An end-to-end (Playwright) test of the full anonymous-try-the-piano-then-link flow: play as anonymous → accumulate some progress → link to Google/email → confirm the same uid, same progress, `isAnonymous: false` (`docs/PRODUCT_SPEC.md` §26).
- Once §7's open question is resolved: a rules/emulator test exercising the chosen conflict-resolution policy explicitly.

## 11. Acceptance criteria

Verbatim, `docs/PRODUCT_SPEC.md` §29:
- "An anonymous try-first session's progress survives linking to a real account"
- "Progress persists across sign-out/sign-in and across devices for the same account"

Additional, derived:
- A client-supplied `uid` in a Function payload is demonstrably never trusted (test, not code-review habit — mirroring the discipline §29 requires elsewhere for credential leakage).
- The linking-conflict policy (§7) is ratified and has passing tests before the Sign in/Sign up screen ships.

## 12. Out of scope

- What an identity is authorized to read/write once established — `docs/SECURITY/AUTHORIZATION.md`.
- Admin/moderator identity — does not exist in MVP (Phase 3, out of scope per `docs/PRODUCT_SPEC.md` §7/§34/§36).
- App Check attestation — a parallel, provider-agnostic layer, not part of identity verification (`docs/SECURITY/APP_CHECK.md`).
- Account deletion/export mechanics — `docs/SECURITY/PRIVACY.md`.

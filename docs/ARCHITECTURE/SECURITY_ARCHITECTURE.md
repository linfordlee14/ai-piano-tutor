# Security Architecture

**Status:** Draft v1.0 — architecture documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §19 (primary), also §17, §18, §20, §29
**Companion documents:** this document is the **architecture-level** companion to the more detailed `docs/SECURITY/` directory (written in parallel — `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` owns the literal rules syntax this document deliberately stops short of). Also see `docs/ARCHITECTURE/FIREBASE_ARCHITECTURE.md`, `docs/ARCHITECTURE/AI_ARCHITECTURE.md`, `docs/ARCHITECTURE/DATA_FLOW.md`, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`.

## 1. Purpose

State the overall security *philosophy* — default-deny, redundant enforcement, and the specific architectural decisions that close the two vectors this app is most exposed to: a modified client self-reporting a perfect score, and a modified client fabricating context to manipulate the AI tutor. Illustrative rule snippets appear where they clarify intent; this document is explicitly not the place for literal, complete Firestore Rules syntax — that belongs to `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.

## 2. Scope

In scope: the default-deny philosophy, the score-tampering closure, credential protection, App Check, rate limiting, prompt-injection hygiene, auth boundaries, and the field-level nuance in the `UserProfile` document that the rules doc needs to get right (§4.3).

Out of scope: literal `firestore.rules` syntax and its own test suite structure (→ `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`), Gemini prompt-template-level guardrails like exact length caps (→ `docs/AI/AI_GUARDRAILS.md`), the AI boundary itself in full (→ `AI_ARCHITECTURE.md`, though prompt-injection hygiene is covered here at the architecture level since it's a security topic first).

## 3. Requirements

- **R1 — Default deny.** Every Firestore path is unreadable and unwritable unless a rule explicitly allows it. No path is ever left to an implicit "probably fine."
- **R2 — The client cannot self-report success.** No path exists — not a bug to be avoided, but a path that structurally does not exist — by which a modified client can write a `PracticeSession`, a `ProgressSummary` entry, or an `AIMemorySummary` document that wasn't independently computed server-side.
- **R3 — Credentials never reach the client.** The Gemini API credential is never in client code, never in a public repo, never in a `NEXT_PUBLIC_*` environment variable.
- **R4 — Redundant enforcement, not single-point enforcement.** Every server-written-only contract is enforced *both* by Firestore Security Rules denying the client write path *and* by the writing Cloud Function independently re-validating/recomputing rather than trusting client input. Neither alone is sufficient (§4.2).
- **R5 — Privacy and deletion are day-one scope, not polish.** Account deletion and data export ship in MVP, not later (`docs/PRODUCT_SPEC.md` §18).

## 4. Architecture

### 4.1 Default-deny philosophy

Every Firestore Security Rule in this system starts from **deny**, with explicit, narrow **allow** rules layered on top — never the reverse. `docs/PRODUCT_SPEC.md` §19 gives the concrete top-level example, reproduced here as an *illustrative* snippet (literal, complete rules syntax is `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`'s job, not this document's):

```
// Illustrative only — see docs/SECURITY/FIRESTORE_SECURITY_RULES.md for the real rules file
match /lessons/{lessonId} {
  allow read: if true;
  allow write: if false;
}
```

Content changes to `lessons/**` happen only via the Admin SDK — a seed script today, an admin Function in Phase 3 — never from any client, at any privilege level. There is no "admin user" client-side write path; "admin" access is exclusively a server-side/Admin-SDK concept in this system, which is what makes `allow write: if false` correct rather than merely convenient.

### 4.2 The score-tampering vector — closed two ways, not one

This is the specific rule that closes an obvious cheating vector: **a tampered client cannot self-report a perfect score, because the client never writes scores at all.**

- `users/{uid}/practiceSessions/{sessionId}`, `users/{uid}/aiMemory/summary`: readable by the owning `uid`, **written only by Cloud Functions** (via the Admin SDK, which bypasses client-facing rules entirely).
- The Cloud Function that writes them (`getExerciseFeedback`) does not trust any client-supplied verdict, score, or accuracy value — it re-runs `packages/music-engine` server-side against the raw `NoteEvent[]` and the `Lesson`/`Exercise` definition it independently loads, and writes *that* result (`DATA_FLOW.md` §4, `MUSIC_ENGINE_ARCHITECTURE.md`).

**Why both mechanisms, redundantly (R4):** restated directly from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5, rule 1, because it's the correct and complete reasoning and shouldn't be paraphrased into something weaker — *"Firestore Security Rules deny the write path entirely, and Cloud Functions independently re-validate/recompute rather than trusting any client-supplied verdict. Neither mechanism alone is sufficient — rules-only leaves the Function itself as an unverified trust boundary if compromised; Function-only leaves a gap if a rules change regresses. Both must hold."* A rules-only defense assumes the Function is always correct and never compromised; a Function-only defense assumes the rules file never regresses. Neither assumption is one this architecture is willing to make alone.

### 4.3 The `users/{uid}` document — a field-level nuance the rules doc needs to get right

This is worth spelling out explicitly because it's a real, easy-to-miss gap between how `docs/PRODUCT_SPEC.md` §19 phrases the rule and how `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15 models the data.

`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15 is unambiguous that `UserProfile` (including its embedded `settings` object and its embedded `ProgressSummary`) is **one Firestore document** at `users/{uid}` — not `settings` and `progressSummary` as separate sub-paths. Its field-level ownership table is exactly this:

| Field | Client-writable? | Written by |
|---|---|---|
| `settings.*` | Yes | Client (own uid only) |
| everything else (`progressSummary`, `xp`, `streak`, `currentLevel`, `currentLessonId`, …) | No | Cloud Functions (Admin SDK) only |

**The architectural requirement this implies:** a Firestore rule that simply says *"a user may write their own `users/{uid}` document"* would be **wrong** — it would let a client overwrite `xp`, `streak`, or `progressSummary` in the same write that legitimately updates `settings.volume`, because Firestore document-level rules don't distinguish fields unless the rule is written to. The correct rule has to be **field-scoped on a single document**, using something equivalent to comparing the incoming write against the existing document and asserting that only the `settings` key differs — for example (illustrative only, not the literal implementation):

```
// Illustrative only — see docs/SECURITY/FIRESTORE_SECURITY_RULES.md for the real rules file
match /users/{uid} {
  allow read: if request.auth.uid == uid;
  allow write: if request.auth.uid == uid
    && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['settings']);
}
```

This is **the one documented exception** to "everything under `users/{uid}` beyond `settings` is Function-written only" (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.11) — and it only holds if the rule is written at this field-scoped granularity, not the coarser per-collection granularity that suffices for `practiceSessions`, `aiMemory`, and `lessons` (§4.2, §4.1), where an entire path can simply be `allow write: if false` because no field within it is ever client-writable.

By contrast, `users/{uid}/practiceSessions/{sessionId}` and `users/{uid}/aiMemory/summary` genuinely are separate documents/subcollections (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.6, §4.8), so a simple document-level `allow write: if false` is sufficient there — no field-level diffing needed. The nuance in this section applies specifically and only to the single `users/{uid}` document.

> This distinction is flagged as an **IMPORTANT** open question in the parent task report: `docs/PRODUCT_SPEC.md` §19's own prose phrases the rule as if `settings` might be its own sub-path (`users/{uid}/settings`) rather than a field on the main document, which would sidestep this nuance entirely if true. This document resolves the discrepancy in favor of `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`'s explicit `UserProfile` interface, per that document's own stated authority as the canonical source for data shapes — but `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`'s author should confirm this reading before implementing the rule, since a document-level rule written under the other assumption would silently reopen the score-tampering vector for `progressSummary`, `xp`, and `streak` specifically (all three sit on the same document as the legitimately client-writable `settings` field).

### 4.4 Gemini credential protection

- Never in client code, never in a public repo, never in a `NEXT_PUBLIC_*` env var (R3).
- Stored as a **Cloud Functions secret (Secret Manager)**, read only inside the Function's server-side execution.
- Verified as a CI check, not a code-review habit: the built client bundle is grepped for the credential as part of the acceptance criteria (§11) — a process check backing up the architectural guarantee, since "we didn't put it there" is not the same claim as "it cannot be there."

### 4.5 App Check

Enabled from day one (not retrofitted later) on:
- **Firestore** — blocks direct, non-app traffic from reading/writing even paths that are otherwise correctly rule-gated.
- **The Gemini-calling Functions** (`getExerciseFeedback`, `getTutorResponse`) — blocks scripted traffic from reaching the Gemini proxy at all, before rate limiting or Gemini spend is even a factor.

`docs/PRODUCT_SPEC.md` §16 calls this "cheap to add early, painful to retrofit after abuse shows up" — the architectural reason it's listed as day-one scope rather than a hardening pass before launch.

### 4.6 Per-user rate limiting

A per-user rate-limit counter, checked and atomically incremented **inside** the Cloud Function (e.g., a daily cap on tutor calls for free-tier users), gates the Gemini call — the check happens *before* Gemini is invoked, not after (`DATA_FLOW.md` §5.2, step 5), so a request that would exceed the cap never generates Gemini spend at all. On a rejected request, the Function returns the standard `{ ok: false, error: { code, message } }` shape (`SYSTEM_ARCHITECTURE.md` §6) rather than a raw exception, so the client can show a clear "you've reached today's limit" state instead of a generic failure.

> The exact numeric cap is **not specified** anywhere in `docs/PRODUCT_SPEC.md` — §19 and §29 both refer to it as "a daily cap" / "the configured cap" without a number. This is intentional as a tunable constant, but it must be set to a real value before launch. Flagged as **IMPORTANT** in the parent task report.

### 4.7 Prompt-injection hygiene

Basic hygiene, scoped to what actually matters given the Gemini call's limited capability surface:
- **Input validation and length caps** on the learner's free-text question, applied before it's dropped into a prompt (exact cap value: `docs/AI/AI_GUARDRAILS.md`).
- **No tool- or function-calling access.** The Function's Gemini call is scoped to plain text output only — a learner typing "ignore your instructions and…" into the tutor chat can, at worst, cause a strange or off-brand text response. It cannot extract the system prompt into an action, cannot trigger a tool call, and cannot reach any data or capability beyond generating text, because none is wired up. This is a capability-surface argument, not a prompt-wording argument: even a *successful* injection has nothing to escalate into.
- **The context injected into every prompt is server-assembled, not client-supplied** (`AI_ARCHITECTURE.md` §4.3) — this closes a *second*, easy-to-miss injection surface: a learner cannot inject fabricated "recent performance" or "memory" content into their own prompt, because they never get to supply those fields at all.

### 4.8 Authentication boundaries

- Every callable Function checks `context.auth` before doing anything user-specific (`docs/PRODUCT_SPEC.md` §19).
- Anonymous users still carry a real Firebase UID, so the same rules and Function-side checks apply uniformly — **there is no separate "unauthenticated" code path to accidentally leave unguarded** (`docs/PRODUCT_SPEC.md` §19, §18).
- A client-supplied `uid` (e.g., in a request body) is never trusted on its own — the Function derives the acting user's identity exclusively from the verified ID token (`docs/PRODUCT_SPEC.md` §18).

### 4.9 Abuse prevention beyond AI

A Firestore rule rejecting a new `practiceSessions` write if the learner's previous session completed less than a few seconds ago — blocking naive script-driven XP farming (`docs/PRODUCT_SPEC.md` §19). Since `practiceSessions` writes only ever originate from `getExerciseFeedback` (§4.2), this check is enforced inside that Function alongside the rules-level protection, consistent with R4.

> The exact cooldown ("a few seconds") is not a specified number in `docs/PRODUCT_SPEC.md`. Flagged as **NON-BLOCKING** — a tunable constant that doesn't threaten the architecture either way, but needs a concrete value before implementation.

### 4.10 Baseline hygiene

Secrets stay out of the repo (`.env.local` gitignored), all sensitive environment variables are server-side only, and dependency scanning (`npm audit`/Dependabot) runs as a matter of course — standard practice, stated explicitly here so it is not assumed away (`docs/PRODUCT_SPEC.md` §19).

### 4.11 Privacy: deletion and export as day-one scope

A Cloud Function (`deleteAccount`, `exportUserData` — `SYSTEM_ARCHITECTURE.md` §6) deletes or exports a user's full data subtree on request. This is **non-negotiable day-one scope**, not later polish (`docs/PRODUCT_SPEC.md` §18): the product is built for South African users (POPIA applies) with a realistic chance of EU/UK users too (GDPR). Both functions require the same auth-boundary treatment as every other callable Function (§4.8) — a user can only delete/export their *own* subtree, verified via the authenticated caller's UID, never a UID supplied in the request payload.

## 5. Data structures

The `UserProfile` field-level ownership table (§4.3) is the single most security-relevant data-structure fact in this system — it is the table every Firestore rule author and every Cloud Function author should check first before writing to (or authorizing a write to) any part of `users/{uid}`. Full type definitions: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`.

## 6. Interfaces / contracts

- `deleteAccount`, `exportUserData` — §4.11.
- Every callable function's error contract — `{ ok: true, data }` / `{ ok: false, error: { code, message } }` (`SYSTEM_ARCHITECTURE.md` §6) — is itself a security-relevant convention: it means an authorization failure, a rate-limit rejection, and a validation failure all surface through one predictable shape the client already knows how to render safely, rather than leaking raw error internals.

## 7. Dependencies

- Firebase Authentication (ID token issuance/verification, Admin SDK) — `FIREBASE_ARCHITECTURE.md`.
- Google Cloud Secret Manager — Gemini credential storage (§4.4).
- App Check (§4.5).
- `@firebase/rules-unit-testing` and the Firebase Local Emulator Suite — the test infrastructure this document's requirements are verified against (§10).

## 8. Performance considerations

The rate-limit check (§4.6) adds one Firestore read/increment per Gemini call. This is acceptable overhead precisely because Gemini calls are already infrequent by design — gated to explicit user action or exercise-end (`AI_ARCHITECTURE.md` §4.4) — so the rate-limit check's own cost never compounds into a hot path. The abuse-prevention cooldown check (§4.9) is similarly cheap: one comparison against the most recent `practiceSessions` write, already being read/written in the same Function call.

## 9. Error handling

- **Rate limit exceeded:** `{ ok: false, error: { code: 'RATE_LIMITED', message } }` (illustrative code name) — the client shows a specific "you've reached today's limit" state, not a generic failure (§4.6).
- **Auth check failure:** the Function returns before touching any data — no partial write, no partial read, on an unauthenticated or mismatched-UID call (§4.8).
- **Cooldown rejection (§4.9):** surfaces through the same typed error contract, distinguishable from a genuine validation failure so the client doesn't present script-driven throttling as if it were a user-facing mistake.

## 10. Testing requirements

Per `docs/PRODUCT_SPEC.md` §26, §29 — these are the two rules where a silent regression becomes a real cheating or content-integrity bug, so they are tested directly rather than trusted to "the emulator didn't complain":
- **Firestore rules unit tests** (`@firebase/rules-unit-testing`), asserting explicitly:
  - A client **cannot** write its own score/progress/AI-memory data (covering the field-level nuance in §4.3, not just the simpler collection-level cases in §4.2).
  - A client **cannot** write lesson content.
- **Integration tests** against the Firebase Local Emulator Suite (Firestore + Functions + Auth), covering the write-only-via-Function pattern end to end, not just the rules file in isolation.
- **CI check:** grep the built client bundle for the Gemini credential — a process check, not a manual review step (§4.4, §11).

## 11. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29:
- [ ] A client cannot write its own `practiceSessions`/progress/AI-memory documents directly (verified by a rules test, not a manual check).
- [ ] A client cannot write to `lessons` at all.
- [ ] No Gemini credential appears anywhere in the built client bundle — grepped as a CI check, not a code-review habit.
- [ ] Per-user rate limiting demonstrably blocks a scripted burst of tutor calls beyond the configured cap.

## 12. Out of scope

- Literal, complete Firestore Rules file → `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`
- Gemini prompt-template-level guardrails (exact length caps, structured-output schemas) → `docs/AI/AI_GUARDRAILS.md`
- The AI deterministic/Gemini boundary itself (what Gemini is allowed to decide) → `AI_ARCHITECTURE.md`
- Firebase service selection rationale (why Firestore, why not Storage for audio) → `FIREBASE_ARCHITECTURE.md`
- Field-by-field Firestore schema → `docs/DATA/FIRESTORE_SCHEMA.md`

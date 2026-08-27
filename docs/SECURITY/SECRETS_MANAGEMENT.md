# Secrets Management

**Status:** Draft v1.0 — security documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §16, §19, §29
**Companion documents:** `docs/SECURITY/SECURITY_REQUIREMENTS.md` (SEC-12, SEC-13, SEC-14, SEC-22), `docs/SECURITY/GEMINI_SECURITY.md`, `docs/SECURITY/APP_CHECK.md`

## 1. Purpose

State exactly where every secret AI Piano Tutor depends on lives, who and what can read it, and the mechanical (not habit-based) checks that keep secrets out of places they must never be — most critically, out of any code path that ends up in the client bundle.

## 2. Scope

In scope: the Gemini API credential, the Cloud Functions/Admin SDK's own service-account-based access, local development secret files, the `NEXT_PUBLIC_*` boundary, dependency scanning, and the secrets inventory for anything added later.

Out of scope: App Check's own configuration (a provider setup, not a "secret" in this document's sense — `docs/SECURITY/APP_CHECK.md`), Firebase project configuration values that are *meant* to be public (the Firebase web config object — API key, project ID, etc. — is not a secret; it identifies the project, not a credential, and Firebase's own security model does not treat it as sensitive).

## 3. Requirements

- Gemini API credentials are never in client-side code, never in a public repo, never in a `NEXT_PUBLIC_*` environment variable — **full stop, no exceptions, including local/manual testing.**
- Gemini API credentials are stored as a Cloud Functions 2nd-gen secret parameter (Secret Manager-backed), read only inside that Function's server-side execution.
- `.env.local` is gitignored.
- Dependency scanning (`npm audit`/Dependabot) runs as baseline hygiene, not an occasional manual check.
- No Gemini credential appears anywhere in the built client bundle — mechanically checked in CI, not a code-review habit.

Full requirement statements with IDs: `docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-12, SEC-13, SEC-14, SEC-22.

## 4. Architecture

### 4.1 How a secret flows

1. The Gemini API credential is provisioned once, out-of-band (Google AI Studio / Google Cloud console), and stored in Google Cloud Secret Manager via Firebase's 2nd-gen Cloud Functions secret-parameter mechanism (`defineSecret()`, bound at deploy time via the `firebase functions:secrets:set` CLI flow).
2. At deploy time, each Cloud Function that needs the credential explicitly declares that secret as a dependency of its own definition — this is an explicit, per-Function opt-in, not a project-wide environment variable every Function automatically inherits. Only `getTutorResponse` and `getExerciseFeedback` (the two Gemini-calling Functions, `docs/PRODUCT_SPEC.md` §20) declare it; every other Function (`deleteAccount`, `exportUserData`, the Auth `onCreate` trigger) has no access to it at all, because it never asked for it.
3. At invocation time, the secret's value is made available to that Function's execution context only — conceptually equivalent to a scoped environment variable, but sourced from Secret Manager rather than a static `.env` file baked into a deployment artifact.
4. The credential is used to construct the Gemini client for that single invocation and is never logged, cached to a client-visible store, persisted to Firestore, or echoed back in any response — including error responses (an error message that includes the credential value, even truncated, is itself a leak and must be avoided in error-handling code).
5. Nothing about this flow ever touches `apps/web` — the Next.js client build. There is no code path, correct or buggy, by which this secret could end up inside the client bundle *unless* someone manually copies it into a `NEXT_PUBLIC_*` variable or into client-reachable source — which is exactly the non-negotiable rule in §3.

### 4.2 Why `NEXT_PUBLIC_*` is a structural boundary, not a style preference

Next.js inlines any environment variable prefixed `NEXT_PUBLIC_` directly into the client-side JavaScript bundle at build time — it is not a runtime lookup that happens to be reachable from the browser, it is literally string-substituted into the shipped code. Using that prefix for the Gemini credential would not be a *risk* of exposure; it would *be* exposure, unconditionally, to every visitor who loads the app, the moment the build runs. This is why the requirement in §3 has no "just for testing" exception: there is no testing scenario where a `NEXT_PUBLIC_*` secret doesn't already mean the secret has shipped.

## 5. Data structures

### 5.1 Secrets inventory

`docs/PRODUCT_SPEC.md` names only the Gemini API credential explicitly. This table also documents the Admin SDK's own access mechanism (referenced constantly elsewhere in this doc set as "the Admin SDK bypasses rules," so it's worth stating precisely what that access actually is) and a placeholder row for anything added later, so a future integration has an obvious slot to fill rather than inventing a new pattern.

| Secret | Purpose | Storage location | Who/what can read it | Rotation | Notes |
|---|---|---|---|---|---|
| Gemini API credential (name illustrative, e.g. `GEMINI_API_KEY`) | Authenticates server-side calls to the Gemini API from `getTutorResponse` and `getExerciseFeedback` | Cloud Functions 2nd-gen secret parameter (Google Cloud Secret Manager-backed), via `defineSecret()` | Only `getTutorResponse` and `getExerciseFeedback`, at runtime, inside their own execution context; Google Cloud IAM principals with Secret Manager access at the project level (restricted to project owners and the CI/CD deploy service account — not any application runtime identity beyond the two Functions above) | No fixed schedule specified by `docs/PRODUCT_SPEC.md` — **PROPOSED:** rotate immediately on suspected compromise; review at least annually otherwise | Never in client code, public repo, or `NEXT_PUBLIC_*` (§3, non-negotiable). Never in `.env.local` committed to source. CI-grepped from the built bundle (§8). |
| Firebase Admin SDK service-account credential | Lets every Cloud Function use the Admin SDK to bypass Firestore Security Rules for server-side reads/writes (score writes, account deletion/export, lesson seeding) | Managed automatically by the Cloud Functions 2nd-gen runtime (a Google-managed service account attached to the Functions runtime environment) | The Cloud Functions runtime itself; Google Cloud IAM at the project level | Managed by Google Cloud, not manually rotated by this project | Included for completeness — this is the mechanism behind every "Admin SDK bypasses rules" statement elsewhere in this document set (`docs/SECURITY/AUTHORIZATION.md` §4.4). Not a developer-managed secret in the typical `.env` sense, but still a credential worth naming in an inventory. |
| *(Future)* any third-party service API key | Not currently named by `docs/PRODUCT_SPEC.md` — no second external service exists in MVP | Would follow the identical pattern: Cloud Functions 2nd-gen secret parameter, scoped to only the Function(s) that need it | Whichever specific Function(s) explicitly declare it | Same policy as the Gemini row, until a service-specific requirement (e.g. a vendor's own rotation mandate) says otherwise | Placeholder row. The day a second external service is integrated, add a row here — do not invent a parallel secrets-handling pattern for it. |

## 6. Interfaces/contracts

The `defineSecret()` binding pattern (Firebase Functions 2nd-gen) is the mechanism by which a Function declares "I need this secret" as part of its own definition, and by which the Firebase CLI's `functions:secrets:set` command writes a new value to Secret Manager without that value ever passing through a file that gets committed. This document states the pattern by name and by property (per-Function opt-in declaration; Secret-Manager-backed; never a project-wide implicit environment variable) rather than reproducing Cloud Functions implementation code, since this repository's documentation-only scope reserves literal code for `functions/` itself.

## 7. Dependencies

- Google Cloud Secret Manager (the storage layer Cloud Functions 2nd-gen secret parameters are backed by).
- Firebase Functions 2nd-gen (`defineSecret()`, `firebase functions:secrets:set`).
- Next.js's build-time environment variable convention (the `NEXT_PUBLIC_*` inlining behavior §4.2 depends on understanding correctly).
- `docs/SECURITY/GEMINI_SECURITY.md` — the Function-side usage of the secret this document stores.
- CI pipeline (whatever runs the production build) — the enforcement point for §8's grep check.

## 8. Security considerations

- **The non-negotiable rule, restated once more because it's the one that matters most:** Gemini credentials must never appear in client-side code, a public repo, or a `NEXT_PUBLIC_*` environment variable — no exceptions, including "just for local testing." A credential typed into a `NEXT_PUBLIC_*` variable for a five-minute local experiment and then "removed" has still shipped in every build made during those five minutes, and — if committed even once — persists in git history regardless of a later revert.
- **`.env.local` must never be committed**, including as a "temporary" commit meant to be reverted. Git history retains it. `.gitignore` must include `.env.local` (and any equivalent local-secret file) from the repository's first commit, not added reactively after a near-miss.
- **CI-enforced, not code-review-enforced.** `docs/PRODUCT_SPEC.md` §29 is explicit that the check for credential leakage into the built client bundle is "grepped as a CI check, not a code-review habit" — a human reviewer skimming a diff is not an acceptable substitute for a build step that fails the pipeline. See §11.
- **Dependency scanning is baseline hygiene, not optional polish.** `npm audit` (or Dependabot, or both) should run on every dependency change, catching known-vulnerable packages before they reach production — stated explicitly in `docs/PRODUCT_SPEC.md` §19 "so it's not assumed away."

## 9. Error handling

If the Gemini secret is missing or misconfigured at deploy/runtime (wrong secret name, not bound to the Function, Secret Manager access denied), the affected Function must **fail closed** — refuse to serve a Gemini-backed response — rather than degrading to any insecure fallback (e.g., never fall back to a hardcoded or client-suppliable credential, and never silently proceed with an unauthenticated Gemini call, which would simply fail anyway but should fail with a clear, typed error rather than an ambiguous one). This failure mode routes into the same existing "Gemini unavailable" fallback UX already specified for any Gemini outage (`docs/PRODUCT_SPEC.md` §10's "temporarily unavailable, here's what to practice instead" card) — a misconfigured secret is, from the learner's perspective, indistinguishable from Gemini itself being down, and should be handled by the exact same degraded-state path rather than a new one.

## 10. Testing requirements

- Firebase Local Emulator Suite test confirming a Function correctly reads its bound secret in a local/emulated context (using the emulator's local secret-value override, not the real production secret).
- A test (or CI step) confirming that removing/invalidating the secret causes the affected Function to fail closed with a typed error, not an unhandled exception or a silent fallback.
- See §11 for the client-bundle-scanning check specifically.

## 11. Acceptance criteria

Verbatim, `docs/PRODUCT_SPEC.md` §29:
- "No Gemini credential appears anywhere in the built client bundle — grepped as a CI check, not a code-review habit"

Implementation note for that CI check: grepping for the literal secret *value* in the client build is one layer (effective only if the actual value is available to CI, which it should be as a deploy-time secret) — an equally important, value-independent layer is asserting that no `NEXT_PUBLIC_*` variable name resembling a Gemini/AI credential exists in the environment configuration at all, and that `apps/web`'s client bundle never imports anything from `functions/src/`. Both checks should exist; neither alone is sufficient (the first catches an accidental literal leak, the second catches the structural mistake before a value even needs to leak).

Additional, derived:
- `.gitignore` contains `.env.local` from the first commit (ties to `docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-22).
- Dependabot (or equivalent) is configured on the repository before the M0 milestone is called complete.

## 12. Out of scope

- Key management process for a future third-party integration beyond noting it follows the identical pattern (§5.1's placeholder row) — no such integration exists yet to design against.
- Historical git-history secret scanning (e.g. a tool that scans all prior commits for accidentally-committed secrets) — worth recommending as a one-time hardening pass before a real launch, but not specified by `docs/PRODUCT_SPEC.md` and not built out here.
- App Check's own credential/provider setup — a distinct mechanism, `docs/SECURITY/APP_CHECK.md`.

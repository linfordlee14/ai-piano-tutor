# App Check

**Status:** Draft v1.0 — security documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §16, §19, §28
**Companion documents:** `docs/SECURITY/SECURITY_REQUIREMENTS.md` (SEC-15, SEC-16), `docs/SECURITY/GEMINI_SECURITY.md`, `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`

This is a **new document** — `docs/PRODUCT_SPEC.md` mentions App Check as a line item in §16's service table and again in §19's AI-request-protection bullet, but does not give it dedicated treatment. This document consolidates both mentions and specifies the mechanism.

## 1. Purpose

Specify Firebase App Check's role in AI Piano Tutor: what it verifies, where it's enforced, what provider it uses, and — the property that makes it worth a dedicated document rather than a footnote — exactly what happens to a request that fails it.

## 2. Scope

In scope: App Check's purpose and threat model, its enforcement scope (Firestore, the Gemini-calling Cloud Functions), provider selection for the web client, debug-token handling for development/CI, and rejected-request behavior.

Out of scope: the rate-limiting and request-shape controls that run *after* App Check inside a Gemini-calling Function (`docs/SECURITY/GEMINI_SECURITY.md`), Firestore Security Rules content (`docs/SECURITY/FIRESTORE_SECURITY_RULES.md` — App Check is a request-provenance gate that runs independently of and prior to those rules, not expressed inside `firestore.rules` syntax).

## 3. Requirements

- App Check is enabled on **Firestore** from day one.
- App Check is enabled on **the Gemini-calling Cloud Functions** (`getTutorResponse`, `getExerciseFeedback`) from day one.
- "Day one" means from the earliest milestone this is buildable at — not retrofitted after abuse is observed.
- A request that fails App Check is rejected before it reaches Firestore or Gemini — never silently allowed through.

Full requirement statements with IDs: `docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-15, SEC-16.

Source, verbatim (`docs/PRODUCT_SPEC.md` §16): *"App Check | Abuse prevention | Enabled from day one on Firestore *and* the Gemini-calling Functions — cheap to add early, painful to retrofit after abuse shows up."*

## 4. Architecture

### 4.1 What App Check verifies

App Check answers a different question than Firebase Auth. Auth answers "who is this request from" (a specific learner's identity, real or anonymous — `docs/SECURITY/AUTHENTICATION.md`). App Check answers **"is this request actually coming from our unmodified app"** — as opposed to a bot, a scraping script, a hand-crafted HTTP client replaying a captured request, or someone who has extracted the Firebase web config (which, per `docs/SECURITY/SECRETS_MANAGEMENT.md` §2, is not itself a secret) and is calling Firestore/Functions directly, bypassing the real client entirely. A request can carry a perfectly valid Firebase Auth token — a script driven by a real, legitimately-created anonymous account still has a real uid — and still fail App Check, because App Check is attesting to the *app*, not the *user*.

### 4.2 Request flow

1. The client app initializes the App Check SDK at startup, registering with the configured provider.
2. The provider issues a short-lived attestation token, obtained transparently — the Firebase client SDKs (Firestore SDK, Callable Functions SDK) attach this token to every outgoing request automatically once App Check is initialized; no per-call code is needed in application logic.
3. On the backend, Firestore and each App-Check-enforcing Cloud Function verify the attestation token before doing anything else.
4. **Missing or invalid token → the request is rejected outright**, before touching Firestore data or spending any Gemini quota (§7).

### 4.3 Provider choice for web

`docs/PRODUCT_SPEC.md` does not name an App Check provider. **Recommended default, explicitly flagged as a recommendation rather than a spec requirement:** reCAPTCHA v3, or reCAPTCHA Enterprise for a higher-assurance/higher-volume future need — these are Firebase App Check's standard providers for web clients (App Check's other providers — DeviceCheck, Play Integrity — are for native iOS/Android apps, which don't exist in this product's MVP or any phase currently scoped, per `docs/PRODUCT_SPEC.md` §8's Phase 4 "Native mobile app" entry being the first and only mention of one). reCAPTCHA v3 is the lower-friction starting choice (invisible to the user, score-based); reCAPTCHA Enterprise is a reasonable upgrade path if abuse patterns ever demand finer-grained control — this is a build-time decision for whoever provisions the Firebase project, not a fact this document is asserting `docs/PRODUCT_SPEC.md` already decided.

### 4.4 Recommended extension beyond the two named surfaces

`docs/PRODUCT_SPEC.md` §16 names Firestore and "the Gemini-calling Functions" specifically. **Recommendation, clearly distinguished from what's explicitly spec'd:** enable App-Check enforcement on every callable Cloud Function — including `getExerciseFeedback` (already covered, since it's Gemini-calling), and also `deleteAccount` and `exportUserData`, which are not Gemini-calling but are still exactly the kind of sensitive, abuse-worth-preventing operation App Check exists for (a scripted account-deletion or data-export flood is a real, if different, abuse scenario). The marginal cost of enabling `enforceAppCheck: true` on two more function definitions is close to zero, and it removes an inconsistency where some callable functions are app-attested and others aren't for no principled reason. This is a recommendation, not something `docs/PRODUCT_SPEC.md` explicitly requires — treat it as a proposal for the implementer to ratify, not as already-decided scope.

### 4.5 Enforced mode, not monitor-only mode

Firebase App Check supports an unenforced "monitoring" mode, useful when retrofitting App Check onto an app that already has real production traffic (it lets you observe how much traffic *would* be rejected before flipping enforcement on and breaking anyone). AI Piano Tutor has no such legacy-traffic problem — `docs/PRODUCT_SPEC.md` §16 calls for App Check "from day one," meaning there is no pre-App-Check client version to worry about breaking. MVP should launch directly in **enforced** mode on every surface it covers; monitoring mode is not a step this project needs to pass through.

## 5. Interfaces/contracts

- **Firestore:** App Check enforcement is a per-database toggle, configured via the Firebase console or CLI — not expressed as conditions inside `firestore.rules` syntax. `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` and this document are complementary, independent layers: a request must pass App Check *and* the rules file's authorization checks; neither substitutes for the other.
- **Cloud Functions:** enforcement is configured per-function via the `enforceAppCheck` option on each callable function's definition (§4.4's recommendation is exactly a call to set this option consistently on all four MVP functions, not just the two Gemini-calling ones).
- **Debug tokens:** local development, the Firebase Local Emulator Suite, and CI all run outside a real browser/App-Check-attestable context. App Check provides a debug-token mechanism specifically for this — a token registered in the Firebase console that's accepted in place of a real provider attestation. **CI and local dev must be unblocked via debug tokens, never by disabling enforcement** — disabling enforcement for convenience anywhere is exactly the kind of "just for testing" exception the credential-protection non-negotiable rule (`docs/SECURITY/SECRETS_MANAGEMENT.md` §3) warns against in spirit, even though App Check tokens aren't credentials in the same sense.

## 6. Dependencies

- Firebase App Check SDK (client).
- reCAPTCHA v3 or reCAPTCHA Enterprise (external Google service, distinct from and unrelated to the Gemini API itself — no shared credential, no shared quota).
- `docs/SECURITY/GEMINI_SECURITY.md` §4.1 — where App Check sits as the first platform-layer gate in the Gemini-calling request pipeline.

## 7. Security considerations

**What happens to a request that fails App Check, stated explicitly per the task this document responds to:** it is **rejected before reaching Firestore or Gemini** — never silently allowed through, never logged-and-permitted, never degraded to a lesser-trust path. This is the same fail-closed posture as every other control in this document set (default-deny rules, fail-closed secret misconfiguration handling). An App-Check failure and a real outage are handled by the *product* the same way (both route into the existing degraded-state UX, §9 below) but they are handled by the *backend* completely differently: an outage still lets a legitimate, attested request through to fail naturally against an unavailable service; an App Check failure never lets the request reach the service at all.

**Why this matters specifically for the Gemini-calling Functions, beyond general abuse prevention:** see `docs/SECURITY/GEMINI_SECURITY.md` §4.2 — App Check is a cost control there, not just a security one, because every request that reaches Gemini costs money regardless of whether it "succeeds" at anything malicious.

## 8. Performance considerations

App Check token attachment and verification happen automatically via the Firebase SDKs and add negligible latency — no additional round trip is introduced beyond what the SDK already does to attach the token to a request it was making anyway. This is not a place with a meaningful performance trade-off to weigh.

## 9. Error handling

`docs/PRODUCT_SPEC.md` does not specify UX for a genuine end-user (not an abuser) whose App Check attestation itself fails — e.g., a corporate/school network that blocks Google's reCAPTCHA domains, or a browser extension interfering with the attestation flow. **Resolution, following the existing pattern rather than inventing new UX:** App Check failure should route into the same degraded-state banner and fallback UX `docs/PRODUCT_SPEC.md` §10 already specifies for "Firestore or Gemini unreachable" ("Playing offline — your progress will sync later" / the AI Tutor's "temporarily unavailable" card). This follows directly from §4.3's Property 1 in `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md`: the piano itself has zero Firestore/Gemini dependency, so an App Check attestation failure — which only ever blocks Firestore sync and Gemini calls, never local piano play — is functionally indistinguishable, from the learner's perspective, from any other reason those two services became unreachable. No new fallback UX needs to be designed for this case specifically; it's an additional *cause* feeding an *existing* effect.

## 10. Testing requirements

- Emulator/debug-token setup so App Check enforcement does not block CI or local development — verified as part of the CI pipeline itself (a CI run that starts failing broadly the moment App Check enforcement is turned on is a signal the debug-token wiring is missing, not that App Check should be disabled).
- A test that replays a request via a raw HTTP client (not the real app, no valid attestation) against an App-Check-enforced Firestore path and against an App-Check-enforced callable Function, asserting rejection in both cases.
- A test confirming enforcement mode is **enforced**, not **monitor-only**, on every covered surface — this is a configuration assertion, not a behavioral one, and easy to silently regress if a monitoring-mode setting is ever used temporarily during debugging and not reverted.

## 11. Acceptance criteria

`docs/PRODUCT_SPEC.md` §29 does not name App Check by a dedicated bullet; the following are derived directly from §16's "day one" instruction and §7 above:
- App Check is configured in enforced (not monitor-only) mode on Firestore and on every Gemini-calling Cloud Function no later than the milestone each surface first becomes reachable (Firestore: M0; the Gemini-calling Functions: M4, per `docs/PRODUCT_SPEC.md` §28) — not retrofitted at M5 hardening.
- A request without a valid App Check token, replayed via a raw HTTP client, is rejected server-side on both surfaces.
- CI and local development are unblocked via App Check debug tokens, never via disabled enforcement.

**Implementation-planning note:** `docs/PRODUCT_SPEC.md` §28's M0 acceptance criterion explicitly names only "security rules default-deny from commit one" — it does not separately name App Check, even though §16 calls for App Check "from day one" in the same spirit. Recommend adding explicit App Check enablement to M0 (for Firestore) and M4 (for the Gemini-calling Functions, which don't exist before M4) acceptance criteria, so "day one" is actually checked at a milestone gate rather than left to be remembered. Classification: NON-BLOCKING — the instruction to build it day-one is unambiguous in §16; this is a milestone-tracking gap, not a policy gap.

## 12. Out of scope

- Native mobile App Check providers (DeviceCheck, Play Integrity) — no native app exists in MVP or any currently-scoped phase (`docs/PRODUCT_SPEC.md` §8, Phase 4).
- reCAPTCHA's own bot-detection scoring/tuning internals — a Google-managed implementation detail, not a project-specific security decision.
- Rate limiting and request-shape validation that runs after App Check passes — `docs/SECURITY/GEMINI_SECURITY.md`.

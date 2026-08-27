# Security Testing

## 1. Purpose

Defines the tests that verify AI Piano Tutor's two structural security guarantees hold: (1) a client cannot fabricate its own progress/scores/AI-memory data, and (2) no Gemini credential ever reaches the built client bundle. Both are named, explicit acceptance criteria (`docs/PRODUCT_SPEC.md` §29), not general hardening advice — "the emulator didn't complain" is explicitly rejected as sufficient evidence (§26).

**Source of truth:** `docs/PRODUCT_SPEC.md` §16–19 (Firebase/Security Architecture), §26, §29. Contracts and field-level ownership: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.6–4.8, §4.15, §5. Rules implementation itself lives in `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` (parallel batch) and `firestore.rules` — this document specifies what must be *provably true* of that implementation, not the rules syntax itself.

## 2. Scope

Covers: `@firebase/rules-unit-testing` assertions on Firestore Security Rules for every server-written-only path, and a CI build-time check that greps the built client bundle for Gemini credential patterns. Overlaps intentionally with `INTEGRATION_TESTING.md` §4–6 (which tests the same rules indirectly, through real Functions and the emulator) — this document's tests are narrower and more mechanical: they exercise the rules directly, without going through a real Cloud Function, so a rules regression is caught even if the Function-level test happens to route around it.

## 3. Why Both This Suite *and* the Integration Suite Test the Same Rules

`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 1 states this explicitly: every server-written-only contract is enforced **redundantly** — rules deny the write path entirely, and Cloud Functions independently re-validate rather than trusting any client-supplied verdict. "Neither mechanism alone is sufficient — rules-only leaves the Function itself as an unverified trust boundary if compromised; Function-only leaves a gap if a rules change regresses. Both must hold." This document tests the rules layer directly and in isolation; `INTEGRATION_TESTING.md` tests the same guarantee through the full Function path. A change that breaks one suite but not the other is exactly the regression this redundancy is designed to catch — do not consolidate these into one suite even though the assertions look similar.

## 4. Rules-Unit-Testing Environment

- `@firebase/rules-unit-testing` against the Firestore emulator, loading the actual `firestore.rules` file the app deploys — never a hand-simplified copy of the rules used only for testing.
- Each test constructs an emulator client authenticated (or explicitly unauthenticated) as a specific test uid via the library's `initializeTestEnvironment`/`authenticatedContext` helpers, then issues the write/read directly against Firestore — no Cloud Function, no HTTP layer, no app UI involved.
- Rules tests reset emulator state between tests (`clearFirestore()` or equivalent) so no test's seeded data leaks into another's assertions.

## 5. Required Rules Assertions

### 5.1 `practiceSessions` — client cannot write, can read own

```
- allow read:  authenticated as {uid} reading users/{uid}/practiceSessions/**        → ALLOWED
- allow read:  authenticated as a DIFFERENT uid reading users/{uid}/practiceSessions/** → DENIED
- allow write: authenticated as {uid} writing (create/update/delete) ANY document
               under users/{uid}/practiceSessions/**, with any payload shape,
               including one crafted to exactly match a real ExerciseVerdict  → DENIED
- allow write: same as above, unauthenticated                                        → DENIED
```

### 5.2 `progressSummary` — a field on `users/{uid}`, not a separate path (see §6)

```
- allow write: authenticated as {uid} updating ONLY users/{uid}.progressSummary
               (no other field touched)                                              → DENIED
- allow write: authenticated as {uid} updating users/{uid}.settings.volume
               together with users/{uid}.progressSummary in the SAME request         → DENIED (whole write)
- allow write: authenticated as {uid} updating ONLY users/{uid}.settings.volume       → ALLOWED
```

### 5.3 `aiMemory/summary` — client cannot write, can read own

```
- allow read:  authenticated as {uid} reading users/{uid}/aiMemory/summary           → ALLOWED
- allow write: authenticated as {uid}, any payload                                   → DENIED
- allow read:  authenticated as a DIFFERENT uid                                      → DENIED
```

### 5.4 `users/{uid}` profile — Function-only fields vs. client-writable `settings`

Per the field-level ownership table in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15:

```
- allow write: {uid} updating settings.keyboardMapping, settings.volume,
               settings.sustainDefault                                               → ALLOWED
- allow write: {uid} updating xp                                                     → DENIED
- allow write: {uid} updating streak.count or streak.lastPracticeDate                → DENIED
- allow write: {uid} updating currentLevel or currentLessonId                        → DENIED
- allow write: {uid} creating the users/{uid} document directly on signup with
               arbitrary starting xp/streak/currentLevel values (rather than the
               Function-assigned defaults)                                           → DENIED
```

### 5.5 `lessons` — public read, zero client write

```
- allow read:  unauthenticated                                                       → ALLOWED
- allow read:  authenticated, any uid                                                → ALLOWED
- allow write: authenticated as ANY uid, create/update/delete                        → DENIED
- allow write: unauthenticated                                                       → DENIED
```

No exception path for any test uid, including one that "looks like" an admin — MVP has no client-facing admin write path at all (`docs/PRODUCT_SPEC.md` §19); if the rules ever grow an admin exception, this test suite must be updated deliberately alongside that change, never left passing by accident because the exception happened not to match the test's uid.

### 5.6 Abuse-prevention rule

```
- allow write: {uid} creating a new practiceSessions document (VIA THE FUNCTION'S
               Admin SDK context) less than the configured minimum interval after
               their previous session's completedAt                                  → DENIED
```

**Resolved:** `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` §5.5 (now written) confirms enforcement lives in `getExerciseFeedback` itself (`functions/src/scoring/`), not in `firestore.rules` — Admin SDK writes bypass rules by default, so a rules-language cooldown has nothing to attach to under the active `allow write: if false` rule on this collection (§5.4 of that document). That document's own rules-language sketch of the cooldown is explicitly illustrative/inert, kept only as a defense-in-depth backstop should a future regression ever open a client-authenticated write path here. Accordingly, this test belongs in `docs/TESTING/INTEGRATION_TESTING.md` §5 (which already asserts it as a `getExerciseFeedback` Function-level behavior: "two rapid-fire calls from the same user produce one success and one rejection") — not as a `@firebase/rules-unit-testing` assertion in this document's §5 list, since there is no rules-level condition to unit-test independent of the Function.

## 6. Why §5.2 Is Tested as a Combined-Write Case, Not Just a Standalone One

`ProgressSummary` is embedded as a map field on the same `users/{uid}` document as `settings` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7, §4.15) — it is **not** a separate document or subcollection, despite `docs/PRODUCT_SPEC.md` §19's rule sketch using slash-path notation (`users/{uid}/settings: readable/writable by request.auth.uid == uid only`) that reads like it might be one. A rules test that only checks "can the client write `progressSummary` alone" would miss the more realistic attack: a single `update()` call that legitimately touches `settings.volume` *and* illegitimately smuggles in a `progressSummary` or `xp` change in the same request. If the eventual rule is a simple per-uid `allow write` without a field-level `diff().affectedKeys()` check, this combined case is exactly what would slip through while the standalone case still (correctly) passes. Always include the combined case, not just the isolated one — `docs/TESTING/INTEGRATION_TESTING.md` §6 tests the same scenario at the Function-plumbing layer; this document tests it at the raw-rules layer.

## 7. CI Check — Gemini Credential Bundle Grep

Direct implementation of `docs/PRODUCT_SPEC.md` §29's acceptance criterion: *"No Gemini credential appears anywhere in the built client bundle — grepped as a CI check, not a code-review habit."*

**Requirements for this check:**
- Runs as a distinct CI step, after the production client build (`apps/web`'s build output — whatever Firebase App Hosting actually ships), not against source files (source-file scanning is a *different*, also-useful check for accidental `.env` commits, but does not prove the *built* bundle is clean — a leak could originate from a bundler inlining an env var even if the source file itself looks fine).
- Greps the built output for: the literal Gemini API key pattern/prefix Google issues, any environment variable name pattern that would indicate a server-only secret leaked into a `NEXT_PUBLIC_*`-prefixed (and therefore client-bundled) variable, and the literal string of the Secret Manager secret's name/path if that itself would be sensitive to expose.
- **Fails the build** (non-zero exit, blocks merge/deploy) on any match — this is a hard CI gate, not a warning.
- Runs on every PR that touches `apps/web`, and again as a pre-deploy gate, since a build produced in one CI run should not be assumed identical to what actually deploys without re-checking.

**Complementary source-level check (defense in depth, not a substitute):** a lint rule or pre-commit hook flagging any Gemini-related secret referenced from a file under `apps/web/` at all (as opposed to `functions/`), since per `docs/PRODUCT_SPEC.md` §19 the credential must never appear in client code, never in a `NEXT_PUBLIC_*` env var, and must live only in a Cloud Functions secret (Secret Manager) read inside server-side execution.

## 8. Error Handling

- A rules-test denial is asserted via the library's `assertFails()` helper (or equivalent), not by loosely checking "no document was written" — the test should confirm the operation was specifically rejected by rules (a `PERMISSION_DENIED`-class failure), distinguishing it from a test that accidentally passes because of an unrelated bug (e.g., a typo'd collection path that "fails" for the wrong reason).
- The bundle-grep CI step's failure output names the specific matched pattern and file, not just "check failed" — a security check that fails silently or unhelpfully gets bypassed under deadline pressure, which defeats its purpose.

## 9. Acceptance Criteria (Restated From `docs/PRODUCT_SPEC.md` §29)

- [ ] A client cannot write its own `practiceSessions`/progress/AI-memory documents directly — verified by a rules test, not a manual check.
- [ ] A client cannot write to `lessons` at all.
- [ ] No Gemini credential appears anywhere in the built client bundle — grepped as a CI check.
- [ ] The combined-write smuggling case (§6) is explicitly tested, not just the standalone field case.

## 10. Out of Scope

- Penetration testing / third-party security audit — not called for at MVP scale by `docs/PRODUCT_SPEC.md`; revisit at `docs/ROADMAP/M5_HARDENING.md` sign-off if the launch scope grows.
- Dependency vulnerability scanning process itself (`npm audit`/Dependabot) — named as baseline hygiene in `docs/PRODUCT_SPEC.md` §19, but that's a CI/process concern owned by `docs/ENGINEERING/` tooling docs, not a "test" in the sense this document covers.
- App Check enforcement testing — App Check's abuse-prevention role (`docs/PRODUCT_SPEC.md` §16, §19) is exercised as part of `AI_TESTING.md`'s rate-limiting scenarios and `INTEGRATION_TESTING.md`'s Function tests, not duplicated here.

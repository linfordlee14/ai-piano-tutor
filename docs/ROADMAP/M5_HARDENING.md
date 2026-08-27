# M5 — Hardening & Launch Readiness

## 1. Objective

Turn a working demo (M1–M4, feature-complete) into something safe to point real users at. By the end of M5, the full `docs/PRODUCT_SPEC.md` §29 acceptance list passes as one connected whole: accessibility pass, security rules tests, account deletion/export, analytics wired, performance budget met, sample attribution credited (§28's literal M5 acceptance line). M5 does not add product features — it verifies, completes, and locks in guarantees that earlier milestones already promised.

**Source of truth:** `docs/PRODUCT_SPEC.md` §18 (Authentication — account deletion/export), §19 (Security), §24 (Accessibility), §25 (Analytics), §22 (Performance), §26, §28 (M5 row), §29 (the full list, in its entirety). Cross-referenced throughout: every `docs/TESTING/*.md` file's own "fully enforced by: M5" marker in `docs/TESTING/TESTING_STRATEGY.md` §5.

## 2. Prerequisites

**M1, M2, M3, and M4 must all be done.** Per `docs/PRODUCT_SPEC.md` §28's own dependency column ("M1–M4") and `docs/ROADMAP/MILESTONES.md` §4's dependency diagram, M5 is the join point after the M3/M4 fork — both the full curriculum (M3) and the AI tutor (M4) must be complete before hardening begins, since several of M5's own checks (the full E2E journey set, the complete §29 acceptance walkthrough) are only meaningful once every MVP feature actually exists to be hardened.

## 3. Exact Scope

- **Full accessibility pass** (`docs/TESTING/ACCESSIBILITY_TESTING.md`): automated axe-core confirmed green across **every** core screen named in that document's §4 list — including the M3/M4 screens (Practice Hub, Note ID Game, Onboarding, Progress/Stats, AI Tutor) that didn't exist when axe-core first started running at M1 — plus the one required **manual screen-reader pass** (mobile VoiceOver, desktop NVDA/VoiceOver per that document's §5), walking the full core interaction loop end to end, with every finding explicitly triaged into "blocks launch" vs. "known limitation, tracked for later" before this milestone can be signed off.
- **Full Firestore Security Rules test suite** (`docs/TESTING/SECURITY_TESTING.md` §5, in its entirety): every assertion in that document's §5.1–5.6 passing together against the real, deployed `firestore.rules` — including the combined-write smuggling case (§6 of that document) and the abuse-prevention cooldown (§5.6, confirmed as an actually chosen, documented interval value rather than the illustrative placeholder M2 introduced it with — see §4 below).
- **Account deletion and data export**, deferred here from M2's own out-of-scope note ("explicitly M5 scope"), even though `docs/PRODUCT_SPEC.md` §18 calls this capability itself "non-negotiable day-one scope" for the product's eventual launch (a statement about *launch readiness*, which is precisely what M5 delivers — not a claim that it must exist before M2):
  - `deleteAccount` and `exportUserData` Cloud Functions (`functions/src/account/`, `docs/PRODUCT_SPEC.md` §20), both verifying `context.auth` and operating only on the caller's own `uid`, never a client-supplied target.
  - Full-subtree cascade: `users/{uid}` profile, every `practiceSessions/**` document, `aiMemory/summary` (real as of M4), and the Firebase Auth record itself — all deleted together, or all included together in an export, with no silent partial omission (`docs/TESTING/INTEGRATION_TESTING.md` §7–8).
  - Settings-screen UI wiring for both (`docs/PRODUCT_SPEC.md` §10's Profile/Settings screen: "sign out, data export/delete").
- **Analytics wired** (§25): Firebase's built-in Google Analytics integration — DAU/WAU, lesson completion rate, session count/duration, funnel drop-off point, most-missed notes/chords (aggregated from already-existing `practiceSessions` data — "essentially free," §25), and AI tutor usage **volume only** (call counts, never content). The explicit privacy boundary from §25 is a hard requirement, not a preference: tutor conversation content and the specific notes a learner played are never sent to the analytics stream — that data already lives correctly scoped in Firestore under the user's own record.
- **Performance budget enforced as a hard CI gate**, not an advisory report (`docs/TESTING/PERFORMANCE_TESTING.md` §4): the Lighthouse CI budget on the Lesson Player route now blocks merge on regression, for the first time as a non-negotiable gate rather than a running-but-non-blocking check. The manual key-press-to-sound latency spot-check (first run at M1) is re-run at least once here against the now-feature-complete build, per that document's own §5 point 5 ("at minimum once before `docs/ROADMAP/M5_HARDENING.md` sign-off").
- **Sample attribution re-verified as actually shipped**: the Salamander Grand Piano / Alexander Holm CC BY 3.0 credit (first added at M1) is confirmed **visible in the shipped Settings/About screen**, not merely present as a `NOTICE` file in the repo — the license requires a visible credit, and M5 is where "visible in the actually-deployed UI" gets confirmed rather than assumed from the file's existence.
- **CI bundle-grep for leaked Gemini credentials** (`docs/TESTING/SECURITY_TESTING.md` §7) — live since M4, re-confirmed here as part of the full acceptance sweep, including as a pre-deploy gate (not just a PR-time check), since a build produced in one CI run should not be assumed identical to what actually deploys.
- **Full E2E suite, all scenarios together** (`docs/TESTING/E2E_TESTING.md` §4–5): the reference scenario, anonymous-to-linked, computer-keyboard parity, Gemini-unavailable degraded state, full lesson-sequence progression, and cross-device resume — all passing as one regression-guarding suite against the complete M1–M4 feature set, not individually against whichever milestone originally introduced each.
- **The complete `docs/PRODUCT_SPEC.md` §29 acceptance checklist**, walked as a single connected pass for the first time — Piano & Input, Music Engine, Lessons & Progress, AI Tutor, Security, and Accessibility sections, all together (see §4 below for why this differs from each item's original milestone of introduction).
- **Baseline hygiene confirmed**, not newly introduced: `npm audit`/Dependabot actually running in CI, `.env.local` (and equivalents) gitignored — both should already be true since M0, and M5 is where they're explicitly re-checked as part of launch sign-off rather than assumed still-true by inertia.

## 4. Why M5 Re-Verifies Rather Than Introduces

Most of the individual guarantees in §3 were already *required* starting at an earlier milestone — `docs/TESTING/TESTING_STRATEGY.md` §5's Test Suite Map marks several suites "first required" as early as M0 or M1. What's specific to M5, and is the actual reason it exists as its own milestone rather than being "already done," is captured by that same table's **"fully enforced by: M5"** column, repeated for five of the seven suites (`INTEGRATION_TESTING.md`, `E2E_TESTING.md`, `ACCESSIBILITY_TESTING.md`, `SECURITY_TESTING.md`, `PERFORMANCE_TESTING.md`). "First required" means the suite must exist and pass for the slice of the product that exists at that milestone; "fully enforced" means the **complete** suite — every scenario, every screen, every rule, as a hard CI gate — holds together, against the **complete** MVP feature set. A milestone can be internally consistent and fully pass its own acceptance criteria while still leaving a "fully enforced" item incomplete simply because the feature that item depends on (the full curriculum, the AI tutor) didn't exist yet. M5 is where those two states — "required and passing for what exists so far" and "fully enforced across the whole product" — become the same thing.

## 5. Files/Components Expected

```
functions/src/account/
  deleteAccount.ts               # full-subtree cascade delete + Auth record deletion
  exportUserData.ts               # full-subtree read + export payload assembly

apps/web/
  app/(app)/profile/                # extended: data export / account deletion UI, confirmation flow
  components/profile/
    DeleteAccountDialog.tsx
    ExportDataButton.tsx
    AttributionCredit.tsx           # Settings/About screen — Salamander/Alexander Holm CC BY 3.0 credit
  lib/analytics/                    # Firebase Analytics event wrapper (screen views, lesson starts/
                                     # completions, taps — product-usage events only, per §25's boundary)

.github/workflows/                   # (or equivalent) — Lighthouse CI now a blocking gate, not advisory;
                                      # bundle-grep re-confirmed as a pre-deploy gate, not just PR-time
firestore.rules                       # abuse-prevention cooldown: placeholder value replaced with the
                                       # actually-chosen, documented interval (see §4/§3)
```

## 6. Technical Requirements

- `deleteAccount`/`exportUserData` verify `context.auth` and act **only** on `request.auth.uid` — never a client-supplied target uid, matching the same authentication-boundary rule every other callable Function has followed since M2 (`docs/PRODUCT_SPEC.md` §19).
- The full-subtree cascade for both operations explicitly enumerates every collection a user's data can live in as of M4: `users/{uid}` profile fields, `users/{uid}/practiceSessions/**`, `users/{uid}/aiMemory/summary` — a hardcoded, reviewed list, not a "delete everything under this path" operation that could silently miss a collection added since this code was last touched.
- The Lighthouse CI budget's numeric thresholds (`docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md` PERF-2/PERF-5's flagged open numeric budgets) are set from **real measurement** against the now-complete build, per that document's own instruction to set them "from real Lighthouse CI / device data once M1 hardening begins" rather than an unmeasured guess — M5 is where that measurement finally happens against the full app, not a placeholder carried forward from M1.
- Analytics events capture screen views, lesson starts/completions, and taps only — no tutor conversation content, no specific notes/chords a learner played, ever, in the general analytics stream (§25) — this is a hard boundary to implement correctly, not a preference to lean toward.

## 7. Data Structures / Contracts Touched

No new shared contract is introduced in this milestone — M5 is the one milestone in this roadmap that deliberately touches zero new `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` shapes. `UserProfile`, `PracticeSession`, `AIMemorySummary` (§4.6, §4.8, §4.15) are read in full by `exportUserData` and deleted in full by `deleteAccount`, exactly as already defined — this milestone's job is completeness and correctness against the existing contracts, not new ones.

## 8. Tests Required

- **`docs/TESTING/INTEGRATION_TESTING.md` §7–8** — this milestone's own literal gate for account deletion/export: the full-subtree cascade test (profile + `practiceSessions` + `aiMemory` + the Auth record itself), and the export-completeness test (asserting on cardinality/key fields against seeded data, not just "some export object came back").
- **`docs/TESTING/SECURITY_TESTING.md`**, in full — every assertion in §5.1–5.6, run together against the deployed rules, plus the CI bundle-grep (§7) confirmed as a pre-deploy gate.
- **`docs/TESTING/ACCESSIBILITY_TESTING.md` §4–5**, in full — axe-core across every core screen, plus the completed and triaged manual screen-reader pass.
- **`docs/TESTING/PERFORMANCE_TESTING.md`**, in full — the Lighthouse CI budget enforced as a blocking gate (§4 of that document) and the manual Android/iPhone latency spot-check re-run against the complete build (§5).
- **`docs/TESTING/E2E_TESTING.md` §4–5**, all scenarios passing together as one suite.
- A final, literal walkthrough of every checkbox in `docs/PRODUCT_SPEC.md` §29, recorded as evidence (a CI run or a dated manual-check record per `docs/ROADMAP/DEFINITION_OF_DONE.md` §3) against each one — not re-asserted from memory that "this passed before."

## 9. Acceptance Criteria

Reproduced from `docs/PRODUCT_SPEC.md` §28 (M5 row), which is itself a pointer to the full §29 list — repeated here as this milestone's own checklist rather than only cross-referenced, since M5's entire job is this list:

- [ ] Accessibility pass complete: full keyboard-only navigation, axe-core green on all core screens, one manual screen-reader pass completed and triaged (§29).
- [ ] Security rules tests complete: a client cannot write its own `practiceSessions`/progress/AI-memory documents directly, and cannot write to `lessons` at all — both verified by rules tests (§29).
- [ ] Account deletion and data export both work, verified by an automated full-subtree test, not a manual check (§18, this milestone's own scope).
- [ ] Analytics wired, with the tutor-content/notes-played privacy boundary confirmed never violated (§25).
- [ ] Performance budget met: key press produces sound in under ~50ms on a mid-range Android device and a current iPhone; Lighthouse CI budget green and enforced as a blocking gate (§29, PERF-1/PERF-5).
- [ ] Salamander Grand Piano attribution is visible in the shipped Settings/About screen, and a `NOTICE` file exists in the repo (§12 licensing requirement, re-confirmed).
- [ ] No Gemini credential appears anywhere in the built client bundle, grepped as a CI check (§29).

## 10. Definition of Done

Checked against `docs/ROADMAP/DEFINITION_OF_DONE.md` in full — this is the first (and only) milestone where **every** item on that checklist is simultaneously live and load-bearing: item 6 ("no secrets exposed") has real Gemini-credential surface area to check since M4; item 5 ("security rules not weakened") applies to the complete rules file, not a subset; item 3 ("no Phase-2+ feature leaked in") is checked against the *entire* shipped app for the first time, not just one milestone's diff.

## 11. Out of Scope for This Milestone

- **Any new product feature.** M5 hardens what M1–M4 already built; it does not add MIDI input, additional lesson content, a song module, achievements beyond streak/XP, offline PWA caching, an admin CMS, Gemini-personalized daily practice, or microphone input — all Phase 2+ (`docs/PRODUCT_SPEC.md` §7, §8), and none of them belong in a hardening milestone regardless of how small or tempting "while we're in here" makes them look (`docs/ROADMAP/DEFINITION_OF_DONE.md` §3's warning applies with full force here).
- **A third-party security audit or penetration test** — not called for at MVP scale by `docs/PRODUCT_SPEC.md`; `docs/TESTING/SECURITY_TESTING.md` §10 explicitly defers this ("revisit at `docs/ROADMAP/M5_HARDENING.md` sign-off if the launch scope grows") — i.e., a decision to make *at* this milestone's sign-off if warranted, not a deliverable this milestone is presumed to include.
- **Resolving the rate-limit-counter open question** flagged in `docs/ROADMAP/M4_GEMINI_TUTOR.md` §4 — a documentation/decision-record item for the project lead (`docs/DECISIONS.md` OQ-15, per `docs/ROADMAP/DEVELOPMENT_METHODOLOGY.md` §4), not something M5's engineering scope resolves by fiat. (The `aiMemory` client-readability question this bullet used to also name was resolved during the documentation QC pass — see `docs/DECISIONS.md` D-01 — and no longer needs M5 or any other milestone to revisit it.)

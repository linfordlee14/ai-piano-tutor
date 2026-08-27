# Testing Strategy

## 1. Purpose

This is the master testing strategy for AI Piano Tutor. It states, in priority order, where testing investment matters most, defines the single canonical end-to-end scenario every other testing document in this repository treats as its reference flow, and maps each test category to its own detailed spec file in this directory.

**Source of truth:** `docs/PRODUCT_SPEC.md` §26 (Testing Strategy), §29 (Acceptance Criteria), §13 (Music Theory Engine — scoring rubric and worked example), §14 (AI/Gemini Architecture — the deterministic/AI boundary), §32 (Recommended Technology Stack — testing tooling row). Contract shapes referenced throughout come from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`. Where this document and a milestone spec in `docs/ROADMAP/` overlap on *when* a test category becomes required, `docs/ROADMAP/MILESTONES.md` and the individual `M*.md` files are authoritative on sequencing; this document is authoritative on *what* the test category must verify.

## 2. Scope

Covers: the rationale for the testing priority order, the canonical E2E reference scenario, a map of every test category to its detailed spec and primary tooling, cross-cutting CI expectations, and the coverage targets pulled from `docs/PRODUCT_SPEC.md` §29. It does not itself contain test code or step-by-step test cases — those live in the category-specific files listed in §5 below.

Out of scope for this document: acoustic pitch detection testing (microphone input is not in MVP — `docs/PRODUCT_SPEC.md` §13's explicit non-goal, §8 Phase 3), MIDI input testing (Phase 2, §8), and load/scale testing beyond the Lighthouse/latency checks in `PERFORMANCE_TESTING.md` (no meaningful concurrent-user scale exists pre-launch).

## 3. Why This Priority Order (Not an Arbitrary List)

`docs/PRODUCT_SPEC.md` §26 orders test categories by **value**, not by build sequence, and that ordering is a direct consequence of §14's non-negotiable boundary: Gemini never determines correctness, scores, XP, or streaks — that is deterministic `packages/music-engine` logic, server-verified. Every other claim this product makes about being trustworthy rests on that engine being *provably* correct. Testing investment should be allocated in the same order the product's trust argument depends on it:

| Priority | Category | Why it ranks here | Detail doc |
|---|---|---|---|
| 1 (highest value) | **Unit tests — music-engine** | Pure functions, zero mocking, near-100% coverage target on chord/scale/scoring logic. This is the literal proof that the AI-boundary argument in §14 holds — if the engine can be wrong, "Gemini never decides correctness" doesn't actually make the app trustworthy, it just relocates the risk. Also the fastest suite to run, so it doubles as the tightest CI feedback loop. | `UNIT_TESTING.md` |
| 2 | **Integration tests — Firebase Emulator Suite** | Verifies the write-only-via-Function pattern actually holds end-to-end (Function writes succeed, direct client writes to the same paths are rejected) and that account deletion/export cascade correctly. This is where the engine's correctness guarantee (priority 1) meets the security guarantee (priority 6) in a running system. | `INTEGRATION_TESTING.md` |
| 3 | **AI/prompt contract tests** | Not "is the answer good" — expensive and unfalsifiable to assert automatically — but "does it fit the schema, respect the length cap, and does the fallback trigger correctly." This is testable and load-bearing; response *quality* is a human-reviewed complement, not an automated gate. | `AI_TESTING.md` |
| 4 | **UI/Playwright tests** | Full user journeys through the piano and lesson UI, anchored on the reference scenario in §4 below. | `E2E_TESTING.md` |
| 5 | **Accessibility tests** | Automated axe-core in CI plus a manual screen-reader pass — automated tools catch roughly half of real accessibility issues (§26), so neither alone is sufficient. | `ACCESSIBILITY_TESTING.md` |
| 6 | **Security rules tests** | `@firebase/rules-unit-testing` assertions that a client cannot self-report a score or write lesson content. Ranked here rather than higher because it's a narrower, more mechanical check than the integration suite above (which already exercises the same rules indirectly) — but it's still an explicit, dedicated suite because a silent regression here is a real cheating/content-integrity bug, not just a UX bug. | `SECURITY_TESTING.md` |
| 7 | **Performance tests** | Lighthouse CI budget on the Lesson Player route, plus a manual key-press-to-sound latency check on real Android hardware — a number that does not show up in Lighthouse at all. Last in the list not because it matters least to the user (audio latency is arguably the single biggest "does this feel like a real piano" risk per `docs/PRODUCT_SPEC.md` §30) but because it's the category most dependent on the others already being correct — there's no point Lighthouse-budgeting a Lesson Player whose scoring is still wrong. | `PERFORMANCE_TESTING.md` |

**Reading this table correctly:** "priority order" governs where engineering time and CI strictness go when trade-offs must be made, not a strict gate where category 2 cannot start until category 1 is "done." In practice, per `docs/ROADMAP/MILESTONES.md`, categories are introduced incrementally as the milestones that need them land (see §5's Test Suite Map).

## 4. The Reference E2E Scenario

Every testing document in this batch — `E2E_TESTING.md` above all, but also the acceptance-criteria sections of `INTEGRATION_TESTING.md` and every milestone spec in `docs/ROADMAP/` — treats the following flow as **the** canonical scenario. It is the operationalized, step-by-step form of the E2E test named in `docs/PRODUCT_SPEC.md` §26 ("user opens lesson → identifies note → presses piano key → receives result → progress is saved") and §28's M2 acceptance row (same flow, same wording, shorthand). This document states it once, in full, and every other document references it rather than restating a slightly different version:

1. **Visitor opens app** — lands on the pre-auth landing screen (`docs/PRODUCT_SPEC.md` §10), no account required.
2. **Starts lesson** — either via "Try the piano now" → an anonymous Firebase Auth session is created silently, or as an already-authenticated returning learner continuing from the dashboard.
3. **Receives note prompt** — the Lesson Player (§10) shows the current lesson's instruction and demonstrates the expected note(s).
4. **Plays key** — via mouse, touch, or mapped computer-keyboard input; the Input Layer (§11) normalizes this into one or more `NoteEvent` objects (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1).
5. **Engine evaluates** — the client's `packages/music-engine` produces an optimistic local verdict for instant visual/audio feedback, then the full `NoteEvent[]` is submitted as an `ExerciseAttempt` (§4.4) to the `getExerciseFeedback` Cloud Function, which independently re-runs the same engine server-side and computes the authoritative `ExerciseVerdict` (§4.5) — the client-side verdict is provisional and is overwritten by the server's response.
6. **UI displays result** — correct/incorrect key state, the specific miss (e.g. `missedNotes`), and the composite score, rendered from the server's `ExerciseVerdict`.
7. **Session completes** — the lesson's pass/fail state resolves (`ExerciseVerdict.passed`), and if passed, the next lesson unlocks per its `prerequisiteLessonId` chain.
8. **Progress is saved** — the Function writes `PracticeSession` (§4.6) and updates `ProgressSummary` (§4.7) server-side; the client never writes these paths directly (`docs/PRODUCT_SPEC.md` §19).

This scenario first becomes fully automatable at **M2** (`docs/ROADMAP/M2_FIRST_LESSON.md`), once one real lesson and its scoring Function exist — `docs/PRODUCT_SPEC.md` §28 makes this M2's explicit acceptance criterion. `E2E_TESTING.md` names the additional scenarios (anonymous-to-linked, computer-keyboard input, Gemini-unavailable degraded state) layered on top of this one as the curriculum and AI tutor land in M3/M4.

## 5. Test Suite Map

| Doc | Covers | Primary tooling | First required at | Fully enforced by |
|---|---|---|---|---|
| `UNIT_TESTING.md` | `packages/music-engine`: note/chord/scale detection, scoring math | Vitest (or Jest — see `docs/PRODUCT_SPEC.md` §32) | M1 | M1, continuously thereafter |
| `INTEGRATION_TESTING.md` | Cloud Functions against Firebase Local Emulator Suite; write-only-via-Function pattern; account deletion/export cascade | Firebase Emulator Suite (Firestore + Functions + Auth) | M2 (`getExerciseFeedback`) | M5 (account deletion/export) |
| `AI_TESTING.md` | Gemini response schema/length/fallback contract tests; small human-reviewed eval set | Vitest/Jest + mocked Gemini client | M4 | M4 |
| `E2E_TESTING.md` | Full user journeys, anchored on §4's reference scenario | Playwright | M2 (reference scenario) | M5 (full journey set) |
| `ACCESSIBILITY_TESTING.md` | Automated contrast/ARIA checks; manual screen-reader pass | axe-core in CI + manual pass | M1 (baseline keyboard nav) | M5 (full pass + manual review) |
| `SECURITY_TESTING.md` | Firestore rules unit tests; CI bundle grep for leaked Gemini credentials | `@firebase/rules-unit-testing`; CI grep step | M0 (default-deny skeleton) | M5 (full rules + bundle-grep pass) |
| `PERFORMANCE_TESTING.md` | Lighthouse CI budget on Lesson Player; manual Android latency check | Lighthouse CI + manual device test | M1 (latency spot-check) | M5 (budget enforced in CI) |

## 6. Tooling Summary

Per `docs/PRODUCT_SPEC.md` §32:

- **Unit:** Vitest or Jest (final pick belongs to `docs/ENGINEERING/CODING_STANDARDS.md`, not this document — both are acceptable per the spec).
- **E2E/UI:** Playwright.
- **Integration/security:** Firebase Emulator Suite + `@firebase/rules-unit-testing`.
- **Accessibility:** axe-core (automated), manual screen-reader pass (NVDA/VoiceOver — see `ACCESSIBILITY_TESTING.md`).
- **Performance:** Lighthouse CI + manual real-device latency check.

## 7. Coverage Targets (Summary)

Pulled from `docs/PRODUCT_SPEC.md` §29; each is expanded with concrete assertions in its category doc:

- Unit test coverage on chord/scale/scoring logic: **≥90%** (`UNIT_TESTING.md` §7).
- All 12 major and all 12 minor triads correctly identified regardless of inversion or note order (`UNIT_TESTING.md` §6).
- The canonical worked example (expected C-D-E-F-G, played C-D-E-G) returns exactly `{missedNotes: ['F'], accuracy: 0.8}` (`UNIT_TESTING.md` §5).
- A client cannot write its own `practiceSessions`/`progressSummary`/`aiMemory` documents, verified by a rules test, not a manual check (`SECURITY_TESTING.md` §5).
- No Gemini credential appears anywhere in the built client bundle, grepped as a CI check (`SECURITY_TESTING.md` §7).
- Full keyboard-only navigation of every non-piano screen; automated axe-core passes on all core screens; one manual screen-reader pass completed pre-launch (`ACCESSIBILITY_TESTING.md`).
- Key press produces sound in under ~50ms on a mid-range Android device and a current iPhone (`PERFORMANCE_TESTING.md` §5, cross-referencing `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md`).
- Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly (`E2E_TESTING.md` §5, `AI_TESTING.md` §6).

## 8. CI Integration Expectations

- **Every pull request:** unit tests (music-engine), rules tests, AI contract tests, axe-core automated checks, and the CI bundle-grep for leaked credentials all run and block merge on failure.
- **Every pull request touching `apps/web` or `functions/`:** the relevant Playwright suite (at minimum the reference scenario in §4) runs against the Firebase Emulator Suite, not production.
- **Lighthouse CI:** runs on the Lesson Player route on every PR touching `apps/web`; budget regressions block merge (`PERFORMANCE_TESTING.md` §4).
- **Manual gates (cannot be automated, must be tracked as explicit pre-launch tasks, not skipped):** the Android latency spot-check (`PERFORMANCE_TESTING.md` §6) and the manual screen-reader pass (`ACCESSIBILITY_TESTING.md` §5) — both required once, before M5 sign-off, and re-run whenever the Lesson Player's core interaction changes materially.
- **Test data:** integration, security, and E2E suites run exclusively against the Firebase Local Emulator Suite. No test ever touches a production Firestore project or a live Gemini API key — `AI_TESTING.md` and `INTEGRATION_TESTING.md` both mock/stub the Gemini client and seed the emulator from fixtures, never from a production export.

## 9. Out of Scope

- Load/concurrency testing at scale — no meaningful concurrent-user number exists pre-launch; revisit post-M5 if usage data suggests a need.
- Cross-browser visual regression testing beyond what Playwright's own assertions cover — not called for by `docs/PRODUCT_SPEC.md`.
- Acoustic pitch detection / microphone input testing — explicit Phase 3 non-goal (`docs/PRODUCT_SPEC.md` §13, §8).
- MIDI input testing — Phase 2 (`docs/PRODUCT_SPEC.md` §8, §11); the `NoteEvent` shape is designed to support a future `MIDIAdapter`, but no MIDI-specific test exists in any MVP milestone.
- Testing Gemini response *quality*/tone as an automated gate — explicitly a human-in-the-loop, periodically-reviewed eval set instead (`AI_TESTING.md` §7), not a CI-blocking check.

## 10. Cross-References

- `docs/ROADMAP/MILESTONES.md` and each `M*.md` file — when each suite becomes required, and that milestone's specific acceptance criteria.
- `docs/ROADMAP/DEFINITION_OF_DONE.md` — "tests pass" as a cross-milestone gate, referencing this document's suites by name.
- `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — every contract shape (`NoteEvent`, `ExerciseAttempt`, `ExerciseVerdict`, `PracticeSession`, `ProgressSummary`) named in §4 above.
- `docs/MUSIC/MUSIC_ENGINE_SPEC.md` (parallel batch) — engine implementation detail underlying `UNIT_TESTING.md`.
- `docs/AI/AI_GUARDRAILS.md` (parallel batch) — the validation rules `AI_TESTING.md`'s contract tests assert against.
- `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md` (parallel batch) — the authoritative latency budget `PERFORMANCE_TESTING.md` tests against.
- `docs/UX/ACCESSIBILITY.md` (parallel batch) — the accessibility commitments `ACCESSIBILITY_TESTING.md` verifies, including the honest piano-interaction caveat.

# End-to-End Testing — Playwright

## 1. Purpose

Defines Playwright-driven full-user-journey testing, anchored on the canonical reference scenario defined once in `docs/TESTING/TESTING_STRATEGY.md` §4. This document lists every E2E scenario implied by `docs/PRODUCT_SPEC.md`, not just the one named test — the reference scenario is the backbone every other scenario in this suite extends or degrades from.

**Source of truth:** `docs/PRODUCT_SPEC.md` §5 (User Stories), §9 (UX Architecture — core interaction loop), §10 (Screen-by-Screen Spec), §11 (Input Layer — keyboard mapping), §14 (AI Architecture — Gemini call boundary), §18 (Authentication — anonymous-to-linked upgrade), §26 (testing strategy), §29 (acceptance criteria).

## 2. Scope

Covers browser-driven journeys through the rendered app: piano interaction (all input methods), lesson progression, sign-up/linking, the AI tutor's happy and degraded paths, and dashboard state after a session. Does not cover: engine correctness in isolation (`UNIT_TESTING.md`), Firestore rules or Function plumbing tested directly against the emulator without a browser (`INTEGRATION_TESTING.md`), or Gemini response schema/fallback logic at the contract level (`AI_TESTING.md` — this suite verifies the *UI's* behavior when Gemini is degraded, not the Function's internal fallback mechanics).

## 3. Environment

- Runs against the app served locally (or in CI) with Firebase Emulator Suite backing it — never production Firestore, Auth, or a live Gemini key. Where a scenario needs Gemini "down," the test disables/mocks the Function at the emulator/network layer rather than calling a real, rate-limited API.
- Audio assertions do not verify actual sound output (Playwright cannot "hear"); they verify the app-level signal that audio played — e.g., `Tone.js`/`AudioContext` state transitions, or a UI state class change gated on the sample having triggered. Actual audible correctness and latency are `PERFORMANCE_TESTING.md`'s manual-device job.

## 4. The Reference Scenario (Required — This Is the M2 Gate)

Implemented exactly as defined in `docs/TESTING/TESTING_STRATEGY.md` §4. This is the automated form of `docs/PRODUCT_SPEC.md` §26's named canonical test and §28's M2 acceptance row — it must pass before M2 (`docs/ROADMAP/M2_FIRST_LESSON.md`) is considered done, and it stays in the suite permanently thereafter as a regression guard.

```
1. Visit the app (unauthenticated) → landing screen renders, "Try the piano now" visible.
2. Tap "Try the piano now" → anonymous Firebase Auth session created; lands on the lesson
   ("Find Middle C" in M2; the current lesson from progress thereafter).
3. Lesson Player shows the instruction/prompt for the expected note.
4. Click/tap the correct piano key (or use the mapped computer-keyboard input — see §6.2).
5. Assert: a network request to getExerciseFeedback fires with a NoteEvent[] payload.
6. Assert: the UI transitions to a correct/incorrect state matching the Function's response
   verdict (not an optimistic-only local guess — wait for the server round-trip).
7. Assert: on lesson completion, a "lesson complete" state renders.
8. Assert: navigating to the Dashboard shows updated progress (streak/XP/completion state
   reflecting this session) — i.e., progress was actually saved, not just displayed
   optimistically in the Lesson Player and lost on navigation.
```

Step 6's "wait for the server round-trip, don't just assert the optimistic state" distinction matters specifically because `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5 is explicit that any client-side preview verdict is provisional and gets overwritten by the server's response — a test that only checks the instant optimistic UI state would pass even if the server-authoritative path were completely broken.

## 5. Additional E2E Scenarios

### 5.1 Anonymous try-first → sign up → progress retained

```
1. As an anonymous session (per §4 steps 1–2), complete at least one exercise attempt
   so a practiceSessions record and progressSummary entry exist for that (anonymous) uid.
2. Initiate sign-up (Google Sign-In or email/password).
3. Assert linkWithCredential succeeds and the resulting authenticated uid is the SAME
   uid the anonymous session used (docs/PRODUCT_SPEC.md §18) — not a fresh account that
   happens to inherit copied data.
4. Assert the Dashboard reflects the pre-signup progress: the completed lesson/exercise,
   any XP/streak state accrued anonymously, is all present post-link, not reset to zero.
5. Additional state to cover per §10: the email-already-registered linking-conflict path —
   assert a clear, non-dead-end error state, not a silent failure or a duplicate account.
```

This scenario is the automated proof behind the §29 acceptance line "An anonymous try-first session's progress survives linking to a real account" and directly serves the product's stated north-star metric (`docs/PRODUCT_SPEC.md` §2) — a first-session melody should never be gated behind, or lost to, a signup form.

### 5.2 Computer-keyboard input produces correct piano behavior

```
1. On the Piano Free Play screen (or a Lesson Player awaiting input), send the mapped
   physical key for Middle C — 'A' per the default mapping (docs/PRODUCT_SPEC.md §11)
   — via Playwright's keyboard.press().
2. Assert this produces the identical engine/audio behavior as clicking the same key:
   the same NoteEvent shape is generated (source: 'keyboard' — diagnostic-only field,
   per TYPES_AND_CONTRACTS.md §4.1, must not change engine behavior), the same visual
   key-active state, the same recognition result if a chord is formed.
3. Repeat for at least one black-key mapping (e.g. 'W' for C#) and one full triad played
   via three simultaneous/near-simultaneous keyboard keys (e.g. A, D, J for a C major
   triad per the default mapping table) to prove multi-key chord entry works via keyboard,
   not just single notes.
4. Assert remapped keys (if the settings screen's remapping feature has shipped by the
   milestone this test runs in) are honored — a key pressed under a custom mapping
   produces the remapped note, not the default one.
```

This is the automated proof behind the §29 line "Mouse, touch, and computer-keyboard input all produce identical engine and audio behavior for the same note," and directly serves the user story "As a learner on a laptop, I want to play using my computer keyboard, so I don't need hardware to practice" (`docs/PRODUCT_SPEC.md` §5).

### 5.3 Gemini-unavailable degraded state — "the learner is never blocked"

```
1. Mock/disable the Gemini-calling Cloud Function (getTutorResponse, and the AI-explanation
   portion of getExerciseFeedback) so calls fail or time out.
2. Repeat the FULL reference scenario (§4) end to end.
   Assert: every step still passes. Scoring, correctness feedback, progress saving —
   none of it depends on Gemini being reachable (docs/PRODUCT_SPEC.md §14: the engine
   computes the verdict; Gemini only explains it after the fact).
3. Navigate to the AI Tutor screen specifically.
   Assert: a clear "temporarily unavailable, here's what to practice instead" fallback
   card renders, built from deterministic data (e.g. the same next-lesson/weak-area logic
   the dashboard already has) — never a blank screen, an infinite spinner, or a raw error
   (docs/PRODUCT_SPEC.md §10).
4. Assert the persistent degraded-service banner appears where specified (§10) and that
   dismissing/navigating away from it does not also break the piano or lesson flow.
5. Re-enable the mock's success path mid-session (simulating recovery) and assert normal
   tutor function resumes without requiring a full page reload.
```

This is the browser-level proof behind the §29 line "Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly, not assumed," and the architecture-level claim in `docs/PRODUCT_SPEC.md` §2 and §33 that Input → Engine → Audio/UI is a closed local loop with no Gemini (or even Firestore) anywhere in its critical path. `AI_TESTING.md` §6 covers the same principle from the Function/contract side (does the fallback trigger correctly on a malformed Gemini response); this scenario covers it from the rendered-UI side (does the *app* actually stay usable) — both are required, neither substitutes for the other.

### 5.4 Full lesson-sequence progression (once M3 content exists)

```
1. Starting from Level 1, complete each MVP lesson in sequence.
2. Assert each lesson's completion unlocks exactly the next lesson per its
   prerequisiteLessonId chain (docs/ENGINEERING/TYPES_AND_CONTRACTS.md §4.2) —
   no skipped unlocks, no dead ends, no lesson reachable out of order.
3. Assert the Note Identification game is reachable from the Practice Hub independent
   of lesson linear progress (docs/PRODUCT_SPEC.md §10 — Practice Hub has "no lesson
   scaffolding").
```

This is the automated proof behind the §29 line "A learner can complete the full MVP lesson sequence with no dead end or broken step." It only becomes runnable once M3's curriculum lands (`docs/ROADMAP/M3_MVP_CURRICULUM.md`) — see `docs/TESTING/TESTING_STRATEGY.md` §5's Test Suite Map for sequencing.

### 5.5 Cross-device / cross-session resume

```
1. Complete a lesson as an authenticated (non-anonymous) user in one browser context.
2. Open a fresh browser context (simulating a different device), sign in as the same
   account.
3. Assert progress (current lesson, streak, XP) matches exactly what the first context
   left off with.
```

Automated proof behind "Progress persists across sign-out/sign-in and across devices for the same account" (§29).

## 6. Error Handling in This Suite

- Every scenario that expects a specific failure mode (linking conflict, Gemini unavailable, rejected write) asserts on the actual user-facing state, not just "no crash" — a silently blank screen is a failing test even if Playwright doesn't throw.
- Flaky-prone waits (audio loading, network round-trips to `getExerciseFeedback`) use explicit assertions on state transitions (loading → ready, pending → correct/incorrect), never fixed `sleep()` delays.

## 7. Acceptance Criteria

- [ ] The reference scenario (§4) passes as an automated Playwright test — this is M2's named acceptance gate (`docs/PRODUCT_SPEC.md` §28).
- [ ] Anonymous-to-linked progress retention passes (§5.1).
- [ ] Computer-keyboard input parity with mouse/touch is proven, not assumed (§5.2).
- [ ] The Gemini-unavailable degraded state leaves the rest of the app fully functional (§5.3).
- [ ] Full lesson-sequence progression has no dead end (§5.4, once M3 content exists).
- [ ] Cross-device resume works for authenticated accounts (§5.5).

## 8. Out of Scope

- Actual audible sound verification and latency — `PERFORMANCE_TESTING.md`'s manual real-device job; Playwright cannot assert on perceived audio quality or hear latency in a way that's meaningful for the ~50ms target.
- MIDI-input journeys — Phase 2 (`docs/PRODUCT_SPEC.md` §8, §11); no MIDI E2E scenario exists in any MVP milestone.
- Gemini response schema/fallback-trigger mechanics at the contract level — `AI_TESTING.md`.
- Visual pixel-diff regression testing — not called for by `docs/PRODUCT_SPEC.md`; this suite asserts on state and content, not screenshots.

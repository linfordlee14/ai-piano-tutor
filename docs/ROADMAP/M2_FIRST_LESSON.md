# M2 — First Guided Lesson, End to End

## 1. Objective

Prove the full loop for one lesson before authoring the rest of the curriculum. By the end of M2, a learner can open the one seeded lesson ("Find Middle C"), get evaluated by server-verified deterministic scoring, see the result, have it persist to Firestore, see it reflected on the Dashboard, and — new in this milestone — sign up from an anonymous session without losing any of that. The canonical reference E2E scenario (`docs/TESTING/TESTING_STRATEGY.md` §4) passes as an automated Playwright test; this is this milestone's literal, named gate.

**Source of truth:** `docs/PRODUCT_SPEC.md` §10 (Lesson Player, Sign in/Sign up screens), §17 (Firestore Data Model), §18 (Authentication), §19 (Security Architecture), §20 (API Architecture), §26/§28 (canonical E2E test), §29. Contracts: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 (`Lesson`), §4.4–4.7 (`ExerciseAttempt`, `ExerciseVerdict`, `PracticeSession`, `ProgressSummary`), §4.15 (`UserProfile`).

## 2. Prerequisites

**M1** must be done: `packages/music-engine` is fully implemented and unit-tested (chord/scale/scoring logic, ≥90% coverage), the piano UI/audio/input stack works standalone in Free Play. M2 is the first milestone that calls the M1 engine from server-side code and wraps it in real lesson content — it does not re-implement or duplicate any engine logic.

## 3. Exact Scope

- **One lesson, seeded as static content:** "Find Middle C" (`docs/PRODUCT_SPEC.md` §5's own framing — "the app to show me where Middle C is and confirm when I find it"), authored as static JSON in `content/lessons/`, loaded into Firestore's `lessons` collection by a seed script (§17) — not hand-written per-document in the console. Content shape and validation (including the `ScoringWeights` sum-to-1.0 check, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14) is owned by `docs/CONTENT/LESSON_CONTENT_SCHEMA.md` (parallel batch); this milestone consumes that schema, it does not define it.
- **Lesson Player screen**, all named states (`docs/PRODUCT_SPEC.md` §10): demonstrating (animated + audio demo on the same piano instance the learner will use) / awaiting input / correct / incorrect-with-hint / lesson-complete. The "ask the tutor why" entry point is present as a UI affordance but is inert/hidden in M2 — it has nothing to open until M4 builds the Tutor screen (see §10 below).
- **`getExerciseFeedback` Cloud Function — deterministic half only** (see §4 below for why this is stated so precisely): verifies the caller's Firebase ID token, accepts an `ExerciseAttempt`, independently loads the `Lesson` definition from Firestore, re-runs `packages/music-engine` server-side against the submitted `NoteEvent[]` (never trusting any client-computed verdict, per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4), computes the authoritative `ExerciseVerdict`, and writes `PracticeSession` + updates `UserProfile.progressSummary` — this is where scoring actually happens, not on the client (`docs/PRODUCT_SPEC.md` §20).
- **Sync Layer** (`apps/web`): buffers the exercise's `NoteEvent[]` locally, computes an optimistic client-side preview verdict for instant feedback, submits the full `NoteEvent[]` as an `ExerciseAttempt` on exercise completion (batched, not per-note — `docs/PRODUCT_SPEC.md` §11, §22), and overwrites the optimistic preview with the server's authoritative response once it arrives (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5).
- **Firestore Security Rules — first feature-specific rules beyond M0's default-deny root:**
  - `lessons/**`: `allow read: if true; allow write: if false;` (`docs/PRODUCT_SPEC.md` §19).
  - `users/{uid}/practiceSessions/**`, and the `progressSummary` field on `users/{uid}`: readable by the owning uid, writable only via the Admin SDK (Cloud Functions) — never by any client SDK call, including the owning user's own (§19). See `docs/TESTING/SECURITY_TESTING.md` §6 for the specific combined-write test this rule must survive, given `progressSummary` shares a document with client-writable `settings`.
  - Abuse-prevention rule: reject a new `practiceSessions` write if the learner's previous session completed less than the configured minimum interval ago (§19) — the exact interval value is not specified in `docs/PRODUCT_SPEC.md`; use an explicit, documented placeholder (e.g. a small number of seconds) and flag it the same way `docs/TESTING/UNIT_TESTING.md` §6.5 handles the unratified `passThreshold`.
- **Dashboard — real completion state:** the "Continue lesson" CTA and completion display now reflect actual `ProgressSummary` data instead of M0's static empty shell. With only one lesson seeded, "continue" reasonably resolves to a "lesson complete — more lessons coming" state; a full lesson-path progression UI is M3 scope, once there's a real path to progress through.
- **Authentication UI** (`docs/PRODUCT_SPEC.md` §10, §18) — placed in this milestone because M2 is specifically the milestone that proves progress persistence end-to-end, and an anonymous session's progress surviving signup is core to that proof, not a hardening afterthought:
  - Sign in / Sign up screen: Google Sign-In, email/password.
  - `linkWithCredential` wiring so a "try-first" anonymous session's uid — and everything already written under it (`practiceSessions`, `progressSummary`) — carries over on signup rather than starting over (§18).
  - The email-already-registered linking-conflict state, surfaced clearly, never a silent failure (§10).
  - A minimal Profile/Settings screen: account info, the keyboard-mapping remapping UI (the mapping *data structure* already exists from M1; M2 adds the settings-screen UI to edit it), volume/sustain defaults, sign out. **Data export/account deletion UI and Functions are explicitly M5 scope, not M2** — see §10 below.

## 4. Why `getExerciseFeedback` Is Scoped Precisely to "Deterministic Half Only" Here

`docs/PRODUCT_SPEC.md` §20 names one Cloud Function, `getExerciseFeedback`, and describes it as doing two things at once: serving the Exercise Feedback **AI** prompt template (§15), *and* validating submitted note events server-side and writing the session/progress docs. Read in isolation, that single line could suggest building the Gemini call into this Function as part of M2. **That reading would violate `docs/PRODUCT_SPEC.md` §14's non-negotiable rule** (the deterministic engine and lesson content ship before Gemini is added on top) and would contradict M4's own stated dependency on M2 specifically *because* M2 produces "real scored exercises to give feedback on" — implying the scoring half exists and is already provable before the AI half is added.

**The resolution this milestone spec uses, stated explicitly so it's not silently guessed at differently by whoever builds M2 versus M4:** `getExerciseFeedback` is introduced in M2 scoped to *only* its deterministic responsibility (§3 above) and returns the `ExerciseVerdict`. **It does not call Gemini, and has no dependency on any Gemini credential, in M2.** The AI-generated-feedback-text portion of this same Function's eventual full contract is *added* in M4 (`docs/ROADMAP/M4_GEMINI_TUTOR.md` §3), as an additive extension of the already-shipped, already-tested M2 behavior — not a rebuild, and not something M2 needs to anticipate beyond leaving the Function's response envelope open to an additional optional field later. This is flagged as an inferred resolution, not an explicit statement in `docs/PRODUCT_SPEC.md` itself, in this batch's delivery report.

## 5. Files/Components Expected

```
content/lessons/
  l01-find-middle-c.json        # the one MVP-launch lesson for this milestone; exact id/slug
                                 # convention owned by docs/CONTENT/LESSON_CONTENT_SCHEMA.md
scripts/seed-lessons.ts          # (or functions/ equivalent) — loads content/lessons/*.json into Firestore

functions/src/scoring/
  validateAttempt.ts              # loads Lesson, re-runs packages/music-engine, produces ExerciseVerdict
functions/src/getExerciseFeedback.ts  # the callable itself — auth check, calls validateAttempt,
                                       # writes PracticeSession + progressSummary, returns {ok, data}

apps/web/
  app/(app)/learn/                # lesson path route — single-item list in M2
  app/(app)/learn/[lessonId]/     # Lesson Player route
  app/(auth)/sign-in/              # Sign in / Sign up route
  app/(app)/profile/                # minimal Profile/Settings route
  components/lesson-player/          # LessonPlayer shell, PromptDisplay, HintButton, LessonCompleteCard
  components/auth/                    # SignInForm, GoogleSignInButton, LinkingConflictNotice
  hooks/useLessonSession.ts            # Sync Layer: buffers NoteEvents, submits ExerciseAttempt,
                                        # reconciles optimistic vs. server verdict
  hooks/useAuth.ts                      # anonymous session + linkWithCredential wiring

firestore.rules                          # updated: lessons public-read, practiceSessions/progressSummary
                                          # Function-write-only, abuse-prevention rule
```

## 6. Technical Requirements

- Every callable Function checks `context.auth` before doing anything user-specific (`docs/PRODUCT_SPEC.md` §19) — `getExerciseFeedback` rejects an unauthenticated call outright.
- App Check enforcement extends to `getExerciseFeedback` now that it's a real, abusable callable Function — §16's "enabled from day one" principle applies as soon as there's something to protect, not narrowly only once Gemini calls exist.
- Server never trusts client-supplied `attemptNumber` or `noteEvents` timestamps for anything that affects scoring (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4).
- The client-side optimistic verdict is explicitly provisional in the UI's own state model — the UI must be able to represent "pending server confirmation" distinctly from "server-confirmed," not just show the optimistic result as if it were final (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5).
- Every Function response conforms to `{ ok: true, data }` or `{ ok: false, error: { code, message } }` (`docs/PRODUCT_SPEC.md` §20) — this is the first milestone where this contract needs a real implementation, and it should be built as a shared helper/wrapper other Functions (M4, M5) reuse rather than each Function inventing its own response shape.

## 7. Data Structures / Contracts Touched

`Lesson`, `ExpectedNote` (§4.2) — first real content authored against this shape. `ExerciseAttempt` (§4.4) — first real client-to-server payload. `ExerciseVerdict` (§4.5) — first server-computed instance backed by real persisted data. `PracticeSession` (§4.6), `ProgressSummary`/`LessonProgress` (§4.7) — first real Firestore documents of these shapes. `UserProfile` (§4.15) — `currentLevel`/`currentLessonId` start actually changing based on real progress; `isAnonymous`/`displayName`/`email` transition from anonymous defaults to real values on linking.

## 8. Tests Required

- **`docs/TESTING/E2E_TESTING.md` §4 — the reference scenario is this milestone's literal gate.** It must pass as an automated Playwright test before M2 is done; it remains in the suite permanently afterward as a regression guard.
- **`docs/TESTING/E2E_TESTING.md` §5.1** (anonymous → sign up → progress retained) — also this milestone's scope, per §3/§4 above's placement of Auth UI here.
- **`docs/TESTING/INTEGRATION_TESTING.md` §4–6** — the write-only-via-Function pattern is first fully testable now that `getExerciseFeedback` and its rules exist; both the positive half (Function writes succeed) and negative half (direct client writes rejected, including the `progressSummary`/`settings` combined-write case) must pass.
- **`docs/TESTING/SECURITY_TESTING.md` §5.1–5.2, §5.5–5.6** — the rules-unit-test equivalents of the same guarantees, tested directly against `firestore.rules` without going through a Function.
- **`docs/TESTING/UNIT_TESTING.md`** continues to gate any change to `packages/music-engine` made in this milestone (there should be few or none — M2 *consumes* the M1 engine, it should not need to modify it).

## 9. Acceptance Criteria

Reproduced and expanded from `docs/PRODUCT_SPEC.md` §28 (M2 row) and §29:

- [ ] The reference E2E scenario passes as an automated Playwright test (§28's literal M2 acceptance line; full scenario in `docs/TESTING/TESTING_STRATEGY.md` §4).
- [ ] A client cannot write its own `practiceSessions`/progress documents directly — verified by a rules test (§29, first enforceable at this milestone).
- [ ] An anonymous try-first session's progress survives linking to a real account (§29).
- [ ] Progress persists across sign-out/sign-in for the same account (the cross-*device* half of this §29 line is more naturally exercised once there's more than one lesson to distinguish states across — `docs/TESTING/E2E_TESTING.md` §5.5 — but the same-device sign-out/sign-in case is fully testable now and should pass in M2).
- [ ] `getExerciseFeedback` returns the typed `{ok, data|error}` shape on every path, including auth failure and malformed input (§20).

## 10. Definition of Done

Checked against `docs/ROADMAP/DEFINITION_OF_DONE.md` — item 5 ("security rules not weakened") is fully live starting this milestone: this is the first milestone that introduces real client-facing Firestore rules beyond M0's bare root, and it is exactly the kind of change `docs/PRODUCT_SPEC.md` §26 warns "a silent regression here would become a real cheating... bug." Item 6 ("no secrets exposed") remains vacuous — no Gemini credential exists yet, by design (§4 above).

## 11. Out of Scope for This Milestone

- **Any Gemini call, from `getExerciseFeedback` or anywhere else** — see §4 above; this is the single most important exclusion in this milestone.
- **The remaining ~11–14 MVP lessons, the Note Identification game, streak/XP logic, the Onboarding/Skill-Check screen** — all M3. The Onboarding screen specifically is deferred here because it exists to set a *starting lesson index* (`docs/PRODUCT_SPEC.md` §10), which is meaningless with only one lesson seeded; it becomes a real feature once there's a path to skip into.
- **Data export and account deletion** (`deleteAccount`, `exportUserData` Functions and their Settings-screen UI) — explicitly M5 scope per this documentation batch's task framing, even though `docs/PRODUCT_SPEC.md` §18 calls this "non-negotiable day-one scope" for the *product's eventual launch readiness* — M2 does not need to build it to prove the lesson loop, and M5 is where the full privacy-compliance surface is verified together.
- **MIDI input** — Phase 2 (`docs/PRODUCT_SPEC.md` §7, §8, §11), unrelated to this milestone regardless.
- **Progress/Stats sub-screen** (accuracy over time, weak notes/chords, practice minutes) — M3, once there's enough real session history across multiple lessons for it to be meaningful; the Dashboard's own completion display in M2 is sufficient for this milestone's acceptance criteria.

# User Flows

## 1. Purpose

Step-by-step flows for the key journeys implied by `docs/PRODUCT_SPEC.md` §9 (UX Architecture) and §10 (Screen-by-Screen Specification). This document does not exist in the original spec — it is written to make the screen transitions and system behavior at each step explicit enough to implement and test against, using only screen names and states already named in §10 (and §9's core interaction loop). Where the spec does not specify a transition trigger, this document says so explicitly rather than inventing one silently — see the flagged points inline and the open-questions summary delivered with this batch.

## 2. Scope

Four flows, as specified:
1. First-time anonymous visitor → plays first note → signs up → progress retained.
2. Returning learner → resumes lesson → completes it → sees feedback.
3. Learner asks the AI tutor a question mid-lesson.
4. Learner hits a Gemini-unavailable degraded state.

## 3. Flow 1 — First-Time Anonymous Visitor to Signed-Up, Progress Retained

**Goal this flow serves:** §2's north star — a first-time visitor plays a recognizably correct note/melody within their first session, unassisted, and that moment survives them creating an account.

| # | Step | Screen / State | System behavior |
|---|---|---|---|
| 1 | Visitor opens the app for the first time | **Landing (pre-auth)** | Value proposition renders; primary CTA is "Try the piano now"; secondary is "Sign in." No auth state exists yet. |
| 2 | Visitor taps "Try the piano now" | Landing → transitioning | An anonymous Firebase Auth session is created invisibly (no form, no visible step). This is the zero-friction entry point named in §10/§18. |
| 3 | App loads the piano | **Piano Free Play** (also serves as the try-before-signup surface, §10) | Piano samples for the initial octave range load; if the browser blocks audio pre-gesture, a "tap to enable sound" affordance is shown per §12/PR-PIANO-14 — this is not a separate screen, it's a state of this screen. |
| 4 | Visitor presses a key | Piano Free Play, active | `NoteEvent` fires from the input adapter → Audio Layer plays sound locally (no network) → Engine Core recognizes the note/chord → a recognition toast renders (e.g., "You played: C4") per §10's Free Play toast pattern. This is the moment that satisfies the north-star "first correctly-identified note," entirely client-side, zero Firestore/Gemini dependency. |
| 5 | Visitor is invited to start structured learning | Piano Free Play → **Learn (Lesson Path)** or **Onboarding / Quick Skill Check** | §10 does not specify the exact trigger that moves a Free-Play visitor into onboarding/lessons (a persistent CTA vs. a prompt after N key presses vs. manual nav). **Flagged as an open question** — this document assumes a persistent, non-blocking "Start learning" entry point is available from Free Play/Landing at all times, consistent with "never a dead end," but the precise UI trigger is not specified in `PRODUCT_SPEC.md`. |
| 6 | Visitor answers or skips the skill check | **Onboarding / Quick Skill Check** | 3–4 skippable questions; skipping sets starting lesson index to Level 1 (default). This step is never gatekeeping — visitor can skip entirely (§10). |
| 7 | Visitor begins the first lesson | **Lesson Player** (demonstrating → awaiting input) | Same anonymous session carries through; no auth is required to attempt lessons. Progress so far (any completed exercises, `PracticeSession` records under this anonymous `uid`) accumulates under the anonymous Firebase uid. |
| 8 | Visitor decides to create an account | **Sign in / Sign up** | Reached via a persistent Profile/nav entry, or an in-context prompt — again, §10 does not specify a forced trigger point for this transition, consistent with try-first being genuinely optional, not a timed paywall. |
| 9 | Visitor completes Google or email/password sign-up | Sign in / Sign up → transitioning | `linkWithCredential` runs: the anonymous session's Firebase uid is linked to the new credential rather than creating a fresh uid. All Firestore data already written under the anonymous uid (profile, any `practiceSessions`, `progressSummary`) is now reachable under the newly-authenticated identity — nothing is copied or migrated, because it was never a different uid to begin with. |
| 10 | Linking conflict case (not the default path) | Sign in / Sign up, conflict state | If the email used is already registered to a *different*, pre-existing account, the flow surfaces a resolvable conflict rather than silently discarding the anonymous session's progress or silently overwriting the existing account (§10 explicit state to handle). |
| 11 | Visitor is now a signed-up learner with continuous history | **Dashboard (Home)** | Dashboard reflects the same streak/XP/progress that started accruing back in step 7 — nothing resets. If this is still the learner's very first completed exercise, the Dashboard empty state ("Start your first lesson") may still apply per §10; otherwise normal Dashboard state renders. |

**Non-negotiable rule surfaced in this flow:** every note-correctness judgment in steps 4 and 7 comes from `packages/music-engine`, running fully client-side with no network call — Gemini is not invoked anywhere in this flow unless the visitor separately opens the AI Tutor (see Flow 3).

## 4. Flow 2 — Returning Learner Resumes, Completes a Lesson, Sees Feedback

**Goal this flow serves:** §5's "resume exactly where I left off" story and the full core interaction loop (§9: Learn → Demonstrate → Practice → Feedback → Repeat → Progress).

| # | Step | Screen / State | System behavior |
|---|---|---|---|
| 1 | Learner opens the app, already signed in | **Dashboard (Home)** | Firebase Auth session persists from a prior visit — no sign-in screen shown. Dashboard renders greeting, streak, XP, "Continue lesson" CTA, today's practice card, one weak-area callout, quick links to Piano/Tutor, sourced from `UserProfile` (one read, per §17/§22's flattened-doc optimization). |
| 2 | Learner taps "Continue lesson" | Dashboard → **Lesson Player** | The Lesson Player opens directly at `UserProfile.currentLessonId`, at the specific step the learner last reached — not restarted from the lesson's beginning. |
| 3 | Lesson opens in Demonstrate state | Lesson Player, `demonstrating` | Animated + audio demo plays on the piano itself (§9); progress-through-lesson indicator shows position within this lesson. |
| 4 | Learner attempts the exercise | Lesson Player, `awaitingInput` → active play | Keys highlight live as the learner plays (§9); each `NoteEvent` triggers sound and visual key state (idle/hover/active/correct/incorrect) immediately, client-side, via the Audio and UI layers. |
| 5 | Learner finishes the attempt | Lesson Player, submitting | The client assembles an `ExerciseAttempt` (`noteEvents`, `startedAt`, `submittedAt`, `deviceType`, `attemptNumber`) and calls the `getExerciseFeedback` Cloud Function. Any client-side "preview" verdict shown instantly for responsiveness is explicitly provisional. |
| 6 | Server computes the authoritative verdict | (server-side, no dedicated screen) | The Function re-runs `packages/music-engine` against the independently-loaded `Lesson` definition — it does not trust the client's `noteEvents` timestamps or any client-computed verdict. It writes `PracticeSession` and updates `ProgressSummary`/streak/XP via the Admin SDK, then returns the `ExerciseVerdict` (and, if requested, `getExerciseFeedback`'s prose) to the client. |
| 7a | Attempt passed the lesson's `passThreshold` | Lesson Player, `correct` / `lessonComplete` | Deterministic verdict renders first (accuracy, missed/extra notes, composite score) — this is Feedback in the core loop. XP/streak updates and a "what's next" nudge render — this is Progress in the core loop (§9). The next lesson unlocks per `prerequisiteLessonId`/`passThreshold` gating. |
| 7b | Attempt did not pass | Lesson Player, `incorrectWithHint` | Verdict still renders (what was right first, the specific miss, one concrete next step — mirroring the Exercise Feedback prompt's output contract, §15), followed by an escalated hint and an immediate retry option — this is Repeat in the core loop. No hard stop. |
| 8 | Learner optionally asks why | Lesson Player → **AI Tutor** (pre-loaded) | See Flow 3 — this is optional, not required to see the deterministic feedback, since deterministic feedback already rendered in step 7a/7b before any Gemini call happens. |
| 9 | Learner returns to Dashboard or continues to the next lesson | **Dashboard** or **Learn (Lesson Path)** | Streak/XP reflect the just-completed session; "Continue lesson" now points at the next lesson in sequence. |

**Non-negotiable rule surfaced in this flow:** step 6 is the only place a score, verdict, or completion state is *decided* — and it is decided entirely by `packages/music-engine` running inside a Cloud Function, never by Gemini. Step 8's tutor, if invoked, only explains the verdict already fixed in step 6/7.

## 5. Flow 3 — Learner Asks the AI Tutor a Question Mid-Lesson

**Goal this flow serves:** §5's "ask 'why does my chord sound wrong' and get an answer grounded in what I actually just played" story, and §10's "ask the tutor why" entry point.

| # | Step | Screen / State | System behavior |
|---|---|---|---|
| 1 | Learner is in an active or just-scored exercise | **Lesson Player**, `incorrectWithHint` or `correct`/`lessonComplete` | The "ask the tutor why" entry point is visible on the Lesson Player (§10). |
| 2 | Learner taps "ask the tutor why" | Lesson Player → **AI Tutor** | The AI Tutor opens pre-loaded with this exercise's context — the transition itself carries a reference to the just-completed `ExerciseVerdict`/`PracticeSession`, not just a blank chat. |
| 3 | Tutor screen renders | AI Tutor, loaded | Chat interface with quick-suggestion chips (e.g., "What should I practice today?"); a persistent, subtle indicator shows the learner's recent-session context is loaded (§10) — the tutor does not present as a blank generic chatbot. |
| 4 | Learner types a free-form question (or taps a chip) | AI Tutor, composing | E.g., "why does my chord sound wrong." Input is length-capped client-side before submission (server re-validates regardless, §19). |
| 5 | Client calls the tutor Cloud Function | AI Tutor, `thinking` | The client-facing signature accepts only `{ learnerQuestion: string }` — nothing else is trusted from the client (`TYPES_AND_CONTRACTS.md` §4.9). A "thinking" state renders; 1–3 seconds is an accepted latency here since this call is already gated to an explicit user action (§22). |
| 6 | Server assembles full tutor context | (server-side) | The Function independently loads `recentExerciseContext` (from the most recent `PracticeSession`), `learnerMemorySummary` (`AIMemorySummary`), and `skillLevel`, then constructs the full `TutorRequest` — the learner cannot fabricate or override any of this by manipulating client input. |
| 7 | Gemini call (mid tier for open-ended Q&A) | (server-side) | The Tutor Q&A prompt template is used (§15); the system instruction forbids inventing note/chord/theory facts not present in the structured input. |
| 8 | Response validated | (server-side) | Checked against the output contract (≤150 words unless the learner asked for more, per §15). On failure, a deterministic canned message is substituted — the client never receives a broken or malformed response. |
| 9 | Answer renders in chat | AI Tutor, response shown | The answer references the specific miss from step 6's context (e.g., names the actual missed note), calibrated to `skillLevel`. `TutorResponse.tier` and `.cached` are used for cost monitoring only — never shown to the learner. |
| 10 | Learner returns to the lesson | AI Tutor → **Lesson Player** | Navigation back preserves the learner's exact position in the lesson — asking the tutor a question is never a dead end that loses lesson state. |

**Non-negotiable rule surfaced in this flow:** step 7's Gemini call receives the `ExerciseVerdict` as read-only input (already computed in Flow 2, step 6) and only produces prose. No step in this flow allows Gemini to change `accuracy`, `missedNotes`, `passed`, or any other verdict field.

## 6. Flow 4 — Learner Hits a Gemini-Unavailable Degraded State

**Goal this flow serves:** the "learner is never blocked" principle (§2), realized concretely for the one dependency that genuinely cannot work offline — Gemini itself (§23: "not realistic, and not attempted: offline AI tutoring... the right fallback is deterministic content stands in when AI is unreachable").

This flow has two entry points, both worth documenting because §10 and §29 describe them slightly differently.

### 6.1 Entry point A — Learner opens or is already in the AI Tutor

| # | Step | Screen / State | System behavior |
|---|---|---|---|
| 1 | Learner opens AI Tutor, or has already asked a question (continuing from Flow 3, step 5) | AI Tutor, `thinking` | Client calls the tutor Cloud Function as normal. |
| 2 | The Gemini call fails or times out | (server-side) | The Function's error contract returns `{ ok: false, error: { code, message } }` — never a raw exception the client has to guess how to handle (§20). |
| 3 | Tutor screen shows its specific fallback | **AI Tutor**, degraded state | A clear "temporarily unavailable, here's what to practice instead" card renders, built entirely from deterministic data (e.g., the same next-incomplete-lesson + last-3-misses logic used for the daily practice card) — never a dead end, never a spinner that never resolves (§10). |
| 4 | Learner dismisses or acts on the fallback card | AI Tutor → **Lesson Player** / **Learn** / **Dashboard** | The learner can act on the deterministic suggestion immediately; nothing about lesson play, scoring, or progress is affected by Gemini's unavailability — only the conversational/explanatory layer is missing. |
| 5 | Gemini becomes reachable again | AI Tutor, next attempt | The next tutor call (new question, or the learner retrying) succeeds normally; no special "recovery" state is needed since each call is independent and stateless from the client's point of view. |

### 6.2 Entry point B — Exercise feedback's Gemini explanation is unavailable

| # | Step | Screen / State | System behavior |
|---|---|---|---|
| 1 | Learner completes an exercise (continuing from Flow 2, step 5) | Lesson Player, submitting | `getExerciseFeedback` is called — this Function does two things: (a) computes the authoritative `ExerciseVerdict` via the engine, always, and (b) optionally generates a Gemini prose explanation of that verdict. |
| 2 | Gemini's portion of the call fails; the engine's portion does not | (server-side) | Per §29's explicit acceptance criterion — "disabling the Gemini Function entirely leaves every other screen fully functional" — the `ExerciseVerdict` computation and the `PracticeSession`/`ProgressSummary` writes are not coupled to Gemini succeeding. |
| 3 | Lesson Player renders the deterministic verdict regardless | Lesson Player, `correct` / `incorrectWithHint` / `lessonComplete` | Accuracy, missed/extra notes, composite score, pass/fail, and progress/XP/streak updates all render normally. Only the natural-language explanation is missing or replaced with a deterministic canned line. |
| 4 | App-wide indication that a service is degraded | Any screen, banner | If the underlying cause is a broader outage (not just this one call), the persistent, non-alarming degraded-service banner (§10: "Playing offline — your progress will sync later") may also be visible — this banner's wording as given in §10 is phrased around Firestore-reachability specifically; this document treats it as the same visual pattern reused for "a backend service is degraded" generally, since §10 groups Firestore-unreachable and Gemini-unreachable under one "Error / Degraded-service states" heading without fully separating their banner copy. **Flagged as a minor spec ambiguity** — exact banner copy per service is a content detail, not a structural gap. |

**Non-negotiable rule surfaced in this flow:** the fact that scoring and progress-writing are structurally independent of the Gemini call (§29, §33's architecture-diagram commentary: "Input → Engine → Audio/UI is a closed local loop with no Firebase or Gemini anywhere in the critical path") is what makes this entire flow possible without a special-cased recovery path — there's nothing to roll back, because nothing scoring-related depended on Gemini in the first place.

## 7. Cross-References

- `docs/UX/SCREEN_SPECIFICATIONS.md` — full per-screen detail (loading/empty/success/error/degraded states) for every screen named above.
- `docs/UX/UX_PRINCIPLES.md` — the core interaction loop and "learner never blocked" principle these flows implement.
- `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — `NoteEvent`, `ExerciseAttempt`, `ExerciseVerdict`, `TutorRequest`, `TutorResponse`, `PracticeSession`, `UserProfile`.

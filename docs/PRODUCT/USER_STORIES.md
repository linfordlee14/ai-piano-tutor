# User Stories

## 1. Purpose

Expands `docs/PRODUCT_SPEC.md` §5 (User Stories) with per-story acceptance criteria. The stories, epics, and `[MVP]`/`[Future]` tags below are unchanged from §5 — this document adds a short list of testable conditions under each, since the original spec tagged scope but did not specify how a story's completion would be verified.

## 2. Scope

Every epic and story in §5, in the same order, with the same tags. `[Future]` stories keep their acceptance criteria for specification completeness (so a future implementer has something to build against) but are **not** built in MVP — do not read their presence here as a scope change. See `docs/PRODUCT/MVP_SCOPE.md` for the authoritative boundary.

## 3. Onboarding & Auth

### 3.1 As a new visitor, I want to try the piano before creating an account, so I can decide if this is for me before committing. `[MVP]`

**Acceptance criteria:**
- Landing screen offers "Try the piano now" as the primary action, requiring no form fields before interaction.
- Selecting it creates an anonymous Firebase Auth session invisibly — no visible signup step.
- The visitor reaches a playable piano (Piano Free Play) and can produce sound on a key press within that same load.
- No account, email, or password is requested at any point in this flow.
- Progress/state generated during this anonymous session is retained if the visitor later signs up (see 3.2).

### 3.2 As a returning learner, I want to sign in with Google in one tap, so I don't re-enter credentials on a new device. `[MVP]`

**Acceptance criteria:**
- A "Sign in with Google" control completes authentication in a single tap/click, with no secondary form.
- On success, the learner's existing Firestore-backed profile and progress load on the new device with no manual data re-entry.
- If the Google account's email is already linked to an existing account, the flow surfaces the linking-conflict state (§10) rather than silently creating a duplicate profile or silently overwriting data.
- The session persists across app restarts on that device without requiring sign-in again.

## 4. Learning the Keyboard

### 4.1 As a beginner, I want the app to show me where Middle C is and confirm when I find it, so I have an anchor point on the keyboard. `[MVP]`

**Acceptance criteria:**
- The lesson visually highlights (glows) the key corresponding to Middle C (MIDI 60) during the Demonstrate phase.
- Pressing the correct key (MIDI 60) is confirmed with a distinct "correct" visual/audio state, not a generic key-press state.
- Pressing any other key during this step is a normal retry (per 4.2), not an error/failure screen; the hint/glow remains available.
- Confirmation happens with zero network round-trip — the local engine validates the note against the lesson's `expectedSequence` immediately.

### 4.2 As a beginner, I want a wrong key press treated as a normal part of learning, not a failure state, so I don't get discouraged. `[MVP]`

**Acceptance criteria:**
- An incorrect key press never renders error styling implying failure (no blocking modal, no punitive copy); it shows a neutral "try again" affordance.
- After a wrong key press, the same exercise stays active and is immediately retryable — no forced navigation, no lesson restart.
- Repeated incorrect attempts escalate the hint level (§9's Repeat stage) rather than repeating an identical, unhelpful prompt.
- Attempt count (`LessonProgress.attempts`) increments internally but is never surfaced to the learner in a shaming way.

## 5. Note Identification Game

### 5.1 As a learner, I want a quick game that asks me to find notes by name with a running accuracy score, so I build recall speed. `[MVP]`

**Acceptance criteria:**
- The game prompts by note name (e.g., "Which key is C?") and accepts a key press as the response.
- A running accuracy score (correct ÷ attempted) updates visibly after each round.
- A running streak counter (consecutive correct) is visible, per §10.
- Each round's correctness is determined by `packages/music-engine`, never by the AI Tutor — the non-negotiable engine/AI boundary applies inside this game too.

### 5.2 As a learner, I want the game's difficulty to increase as I improve, so it stays challenging. `[MVP]`

**Acceptance criteria:**
- Difficulty (`Exercise.difficulty`) increases after a defined run of correct answers and decreases after a defined run of misses; exact thresholds are exercise metadata, not hardcoded logic.
- Every difficulty change is visibly indicated the moment it happens — never inferred silently by the learner (§10: "difficulty step-up/step-down is visible, not silent").
- Difficulty state is scoped to the current exercise pool (`Exercise.pool`), consistent with `Exercise` being static config rather than a Firestore-persisted entity.

## 6. Scales & Chords

### 6.1 As a learner, I want to see and hear a scale demonstrated before attempting it, so I know what correct sounds like. `[MVP]`

**Acceptance criteria:**
- Before the "awaiting input" state, the Lesson Player plays the full scale sequence on the piano itself, audibly and with in-order visual key highlighting.
- The demonstration plays at the lesson's `tempoBpm` when one is defined.
- The learner can replay the demonstration on request — it does not disappear irrecoverably after the first play.

### 6.2 As a learner, I want to know specifically which note I missed in a scale attempt, not just "wrong," so I know what to fix. `[MVP]`

**Acceptance criteria:**
- After an attempt, `ExerciseVerdict.missedNotes` is computed and displayed by name (e.g., "F"), not just a pass/fail indicator.
- The spec's own worked example holds structurally for any scale: expected C-D-E-F-G, played C-D-E-G → `{missedNotes: ['F'], accuracy: 0.8}` (§13, §29).
- Extra/wrong notes played (`ExerciseVerdict.extraNotes`) are also named, not silently dropped.
- This feedback is available even if Gemini is unreachable — it is produced by the deterministic verdict, not the tutor.

### 6.3 As a learner, I want to build from single notes to simple triads gradually, so chords don't feel like a wall. `[MVP]`

**Acceptance criteria:**
- The MVP lesson sequence places single-note and melody lessons before the first chord/triad lesson, per §7's stated ordering (keyboard orientation → note names → octaves → melodies → rhythm → intro to triads).
- The first chord lesson introduces one major triad (e.g., C major) in root position only — not multiple chords or inversions simultaneously (§7).
- `Lesson.prerequisiteLessonId` + `passThreshold` gating makes the chord lesson unreachable before the preceding single-note/melody lessons are passed.

## 7. AI Tutor

### 7.1 As a learner, I want to ask "why does my chord sound wrong" in plain language and get an answer grounded in what I actually just played, so the help feels relevant, not generic. `[MVP]`

**Acceptance criteria:**
- The learner can type a free-form question into the Tutor Q&A interface.
- The Cloud Function assembles `TutorRequest.recentExerciseContext` server-side from the learner's most recent `PracticeSession`/`ExerciseVerdict` — the client supplies only `learnerQuestion` (`TYPES_AND_CONTRACTS.md` §4.9).
- The tutor's answer references the specific miss already present in that `ExerciseVerdict` (e.g., names the actual missed/extra note), not a generic abstract explanation of chords.
- The tutor never independently re-judges whether the chord was correct — it explains a verdict the engine already computed. This is the non-negotiable architecture rule and is testable directly: no code path exists where a Gemini response can overwrite or bypass `ExerciseVerdict`.
- If no recent exercise context exists yet, `recentExerciseContext` is `null` and the tutor still answers without fabricating exercise history.

### 7.2 As a learner, I want the tutor to suggest what to practice today based on my recent mistakes, so I don't have to plan my own practice. `[MVP]`

**Acceptance criteria:**
- A "What should I practice today?" quick-suggestion chip is available in the Tutor UI (§10).
- The suggestion comes from the deterministic daily-practice generator (next incomplete lesson + review of last 3 misses), not a live Gemini call — per §7's explicit MVP pushback against AI-personalized plans before there's usage data.
- The suggestion updates as the learner's `PracticeSession` history changes; it is not a static, one-time recommendation.
- This flow behaves identically whether or not Gemini is reachable, since the underlying plan is deterministic.

### 7.3 As a returning learner, I want the tutor to already know I'm weak on F and B without re-explaining, so sessions don't restart from zero. `[MVP]`

**Acceptance criteria:**
- `AIMemorySummary.weakNotes`/`weakChords` is read server-side and included in every `TutorRequest` automatically — the client never needs to supply it.
- The tutor never prompts the learner to state their weak areas from scratch when `AIMemorySummary` already has them populated.
- `AIMemorySummary` is rewritten in place on each recomputation from recent `PracticeSession` records — never a growing, ever-larger transcript (§17, §31).
- A persistent, subtle UI indicator on the Tutor screen shows that recent-session context is loaded (§10) — the tutor never presents as a blank generic chatbot to a learner with history.

## 8. Practice & Progress

### 8.1 As a learner, I want one daily practice session generated for me, so I don't have to decide what to do each time I open the app. `[MVP]`

**Acceptance criteria:**
- Opening the Dashboard shows a single generated "today's practice" session (next incomplete lesson + review of last 3 misses) without requiring the learner to choose from a menu.
- The plan is generated deterministically from `ProgressSummary`/recent `PracticeSession` data — not by a live Gemini call.
- The plan updates day to day as progress changes; it is not static or one-time.

### 8.2 As a learner, I want a streak and basic stats visible, so I have a reason to come back tomorrow. `[MVP]`

**Acceptance criteria:**
- Streak count and XP are visible on the Dashboard once the learner has at least one completed session.
- Streak increments are calendar-day based (`UserProfile.streak.lastPracticeDate` as `YYYY-MM-DD`), not session-count based.
- Lessons completed, practice minutes, and accuracy trend are reachable from Progress/Stats under Profile.
- All streak/XP/stats values shown come from the server-written `UserProfile` document — never a client-side computed guess (this is what closes the self-reporting cheating vector, §19, and must be verified by a rules test per §29, not a manual check).

### 8.3 As a returning learner, I want to resume exactly where I left off, so nothing is lost between sessions. `[MVP]`

**Acceptance criteria:**
- `UserProfile.currentLessonId` drives the "Continue lesson" CTA and reflects the true furthest-incomplete point — not "Level 1" after every sign-in.
- Progress persists across sign-out/sign-in and across devices for the same account (§29 — verified by an automated test, not a manual check).
- An anonymous session's progress survives linking to a real account without resetting (see 3.1/3.2, PR-AUTH-04).
- If Firestore is briefly unreachable, in-progress local session state is not lost — it queues and syncs once reachable (PR-PLATFORM-04/05).

## 9. Piano Interaction

### 9.1 As a learner on a laptop, I want to play using my computer keyboard, so I don't need hardware to practice. `[MVP]`

**Acceptance criteria:**
- The default computer-keyboard mapping (A/W/S/E/D/F/T/G/Y/H/U/J/K) is active with no setup step required.
- A computer-keyboard key press produces the same `NoteEvent` shape, sound, and engine verdict as the equivalent mouse/touch press on the corresponding key (§29: "identical engine and audio behavior for the same note").
- The mapping is changeable from Settings, and the changed mapping persists (`UserProfile.settings.keyboardMapping`, client-writable).

### 9.2 As a learner on my phone, I want the piano to stay usable on a small screen without keys becoming unplayably tiny. `[MVP]`

**Acceptance criteria:**
- Touch targets are ≥44px per key on phone-sized viewports.
- The visible range is 1–1.5 octaves on phones with horizontal pan to reach the rest of the keyboard, rather than shrinking every key to fit more octaves (§9). **This exact range is flagged as unvalidated** — see `docs/UX/RESPONSIVE_DESIGN.md` for the open question on breakpoints/octave counts before it is locked.
- Panning does not interrupt in-progress audio or lose the learner's place in an active exercise.

### 9.3 As a learner with a MIDI keyboard, I want to connect it and have the app recognize what I actually play. `[Future — MIDI, Phase 2]`

**Acceptance criteria** (specified now for completeness; not built in MVP — see `docs/PRODUCT/MVP_SCOPE.md`):
- `navigator.requestMIDIAccess()` is feature-detected before any MIDI-related UI is shown.
- A connected device's `MIDIMessageEvent` note-on/note-off, velocity, and timestamp normalize into the same `NoteEvent` shape used by every other input adapter — the Engine Core requires no MIDI-specific branching.
- On a browser without Web MIDI support (verified: Safari, macOS and iOS), the app shows "MIDI isn't available in this browser" rather than a silent failure or an inert "Connect MIDI" control.

## 10. Songs

### 10.1 As a learner who has the basics, I want to learn a real (public-domain) song section by section, so I have something to show for my practice. `[Future]`

**Acceptance criteria** (specified now for completeness; not built in MVP):
- Only public-domain or originally-composed songs are used as content, pending a separate licensing conversation (§8, §30).
- A song is broken into sections that can be attempted and scored independently, using the same `ExerciseVerdict`/scoring engine as lessons — no separate, unaudited scoring path.
- Access to song content is gated behind having completed prerequisite basics, consistent with the story's own "who has the basics" framing.

## 11. Accessibility

### 11.1 As a screen-reader user, I want lesson instructions and feedback announced, so I'm not locked out of the non-piano parts of the app. `[MVP — see §24 for what's realistic on the piano itself]`

**Acceptance criteria:**
- Lesson instructions, exercise feedback, and score results are exposed via ARIA live regions and announced without the learner needing to manually locate them.
- All non-piano navigation, controls, and the tutor chat are fully keyboard-operable (Tab/Enter/Space) with visible focus indicators.
- The piano itself supports full keyboard-only play (see 9.1) — this benefits non-mouse users generally, but is **not** claimed as full accessible parity with sighted visual play. See `docs/UX/ACCESSIBILITY.md` for the honest caveat this story deliberately does not oversell.
- Acceptance is validated by both an automated axe-core check in CI and at least one manual screen-reader pass before launch (§26, §29) — automated checks alone do not close this story.

## 12. Content Ops

### 12.1 As Linford (admin), I want to edit lesson content without a code deploy, so content iteration doesn't block on engineering time. `[Future]`

**Acceptance criteria** (specified now for completeness; not built in MVP):
- An authenticated admin can edit `Lesson` fields (title, instructions, `expectedSequence`, etc.) through a CMS UI, with changes taking effect without redeploying the Next.js app.
- Writes still land in the same `lessons` Firestore collection, validated against the same schema `packages/shared-types` defines — no parallel, divergent content path.
- This story is explicitly built on top of, not instead of, the MVP mechanism of static seed JSON (§17) — MVP ships with no admin UI.

## 13. Cross-References

- `docs/PRODUCT/PRODUCT_REQUIREMENTS.md` — the stable-ID requirement list these stories collectively justify.
- `docs/PRODUCT/USER_FLOWS.md` — end-to-end flows that exercise multiple stories in sequence.
- `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — the exact shapes referenced throughout (`NoteEvent`, `ExerciseVerdict`, `AIMemorySummary`, `TutorRequest`, `UserProfile`, `LessonProgress`).

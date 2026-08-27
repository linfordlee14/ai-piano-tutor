# Screen Specifications

## 1. Purpose

Full per-screen specification for every MVP screen named in `docs/PRODUCT_SPEC.md` §10 (Screen-by-Screen Specification). §10 is a good starting point but is deliberately brief (a paragraph or two per screen) — this document expands each screen to implementation-ready detail: purpose, user goal, entry point, key elements/layout, primary interactions, loading/empty/success/error/degraded states, accessibility notes, and mobile-vs-desktop differences.

**Scope discipline:** only the MVP screens listed in §10 are specified here. No Phase 2+ screen (a Song Player, an Admin CMS view, a MIDI-device settings screen, etc.) is speculatively specified — see `docs/PRODUCT/MVP_SCOPE.md`. Where this document fills in a detail §10 doesn't state explicitly (e.g., a loading-spinner treatment), that is ordinary implementer-level UX elaboration consistent with stated principles, not a scope change. Where a gap is a genuine ambiguity about product behavior (not just visual polish), it is flagged inline as an open question rather than silently resolved.

## 2. Conventions Used Throughout

- **States** follow the pattern: Loading → Empty (if applicable) → Success → Error → Degraded. Not every screen has a meaningful Empty state; those are marked "N/A" with a one-line reason.
- **Component references** point to `docs/UX/COMPONENT_SPECIFICATIONS.md`.
- **Accessibility notes** here are screen-specific application of the rules in `docs/UX/ACCESSIBILITY.md` — that document owns the *rules*, this document owns *how they apply per screen*.
- **Type references** point to `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`.
- **Degraded-service banner pattern**: per §10, a persistent, non-alarming banner appears whenever Firestore or Gemini is unreachable, worded like "Playing offline — your progress will sync later." Every screen below states whether/how this pattern applies to it, rather than repeating the banner's full definition each time — that definition lives in §14 (Error / Degraded-Service States) of this document.

## 3. Landing (Pre-Auth)

### Purpose
The unauthenticated entry surface. Its entire job is to get a first-time visitor playing a note with the least possible friction — this screen exists to serve §2's north star, not to sell the product with marketing copy that delays interaction.

### User goal
"Is this worth my time, and can I try it right now without signing up for anything?"

### Entry point
First load of the app for anyone with no existing Firebase Auth session (anonymous or real).

### Key elements / layout
- Value proposition (short — this is not a marketing landing page with scroll depth, it's a gate a visitor should pass through in seconds).
- Primary CTA: **"Try the piano now"** — visually dominant, thumb-reachable on mobile.
- Secondary action: **"Sign in"** — present but deliberately subordinate to the primary CTA.
- No form fields on this screen at all.

### Primary interactions
- Tap/click "Try the piano now" → an anonymous Firebase Auth session is created invisibly → routes to Piano Free Play (§10: Free Play "is also the screen used for the try before signup flow").
- Tap/click "Sign in" → routes to Sign in / Sign up.

### Loading state
Brief — this screen has no data dependency beyond the app shell itself; there is no Firestore read gating its render. If Firebase SDK initialization takes a perceptible moment, show a minimal branded loading state, not a blank white screen.

### Empty state
N/A — this screen has no user-specific data to be empty of.

### Success state
Value prop and both CTAs rendered and interactive.

### Error state
If Firebase Auth fails to initialize at all (rare — client SDK/network issue before any user action), show a minimal inline message with a retry action. This is distinct from the degraded-service banner pattern (§14), since at this point no session exists yet for progress-sync concerns to apply to.

### Degraded/offline state
The degraded-service banner pattern does not apply here in its usual form — this screen precedes any Firestore/Gemini dependency. If the visitor has no network at all, "Try the piano now" should still succeed at creating a local session and reaching Piano Free Play (anonymous auth may queue; the piano itself has zero network dependency once loaded, per PR-PLATFORM-03) — but full offline app-shell loading is Phase 2 PWA scope (§23), so a first-ever visit with zero connectivity is not guaranteed to work in MVP. This is inherent to §23's stated scope, not a gap this document is introducing.

### Accessibility notes
Both CTAs are large touch targets, keyboard-reachable, with visible focus indicators (PR-A11Y-01). See `docs/UX/ACCESSIBILITY.md` §4.

### Mobile vs desktop
Mobile: single-column, CTA large and thumb-reachable in the lower half of the viewport. Desktop: value prop and CTAs can sit side-by-side or centered with more whitespace; no functional difference, layout only.

---

## 4. Sign In / Sign Up

### Purpose
Convert a visitor (anonymous or brand new) into an identified account, without losing any progress already made.

### User goal
"I want to save my progress / access it on another device, without redoing what I already did."

### Entry point
- From Landing's "Sign in" action.
- From an anonymous session choosing to upgrade (exact in-app trigger for this is not specified in §10 — see `docs/PRODUCT/USER_FLOWS.md` Flow 1, step 8 note).
- From Profile, for an anonymous learner who wants to create a real account.

### Key elements / layout
- Google Sign-In button (one-tap).
- Email/password fields with submit action.
- No third element competing visually with these two paths — §10 names exactly these two providers for MVP (`PR-AUTH-02`/`PR-AUTH-03`).

### Primary interactions
- Google button → OAuth flow → on success, `linkWithCredential` runs against the current anonymous session if one exists, so the just-completed anonymous progress carries over rather than starting a fresh uid (§10, §18).
- Email/password submit → same linking behavior on success.

### Loading state
Button/form shows an inline pending indicator during the auth round-trip; both inputs disable to prevent duplicate submission.

### Empty state
N/A — this is a form screen, not a data-listing screen.

### Success state
Redirects to Dashboard (Home), now reflecting the (possibly just-linked) account's data.

### Error state
- Invalid credentials / malformed email → inline field-level error, form remains editable.
- **Linking conflict** (explicit §10 state): the email being used to sign up is already registered to a different, pre-existing account. The screen must surface this as a distinct, resolvable state — e.g., offering "sign in to that existing account instead" — rather than silently discarding the anonymous session's progress or silently overwriting the other account. §10 names this as a state to handle explicitly; it does not specify the exact resolution UI, so implementers should treat "sign in to the existing account, then decide what to do with orphaned anonymous progress" as the safe default, and should not silently merge two distinct real accounts' data.

### Degraded/offline state
If Firebase Auth itself is unreachable, show a direct, non-generic error ("Can't sign in right now — check your connection and try again") rather than the ambient degraded-service banner, since sign-in is a blocking action the learner explicitly initiated and needs a direct answer about, not a background ambient notice.

### Accessibility notes
Standard accessible form patterns: labeled inputs, inline error messages associated with their field via ARIA, focus moves to the first error on failed submission. See `docs/UX/ACCESSIBILITY.md` §4.

### Mobile vs desktop
No functional difference. Mobile: full-width stacked fields. Desktop: fields may be narrower/centered rather than full-width.

---

## 5. Onboarding / Quick Skill Check

### Purpose
Route a learner to an appropriate starting point without gatekeeping anyone. Exists primarily to serve the secondary persona (the rusty returner) — see `docs/PRODUCT/USER_PERSONAS.md` §5.

### User goal
"I don't want to sit through beginner material I already know" (returning learner) — or simply, "let me get to the piano" (any learner who skips).

### Entry point
After first sign-up, or after a visitor's initial "Try the piano now" moment, before the first structured lesson begins (exact trigger for *when* this is shown relative to Free Play is not specified in §10 — see `docs/PRODUCT/USER_FLOWS.md` Flow 1, step 5 note).

### Key elements / layout
- 3–4 short questions, skippable individually or as a whole. §10 gives two example questions verbatim ("Have you played before?", "Do you know any notes already?"); the remaining 1–2 questions are not specified in `PRODUCT_SPEC.md` and are a content-authoring task, not a structural gap.
- A visible, unambiguous "Skip" action, at least as prominent as "Next"/"Continue" — this screen must never read as mandatory.

### Primary interactions
- Answer each question (single-select or similar lightweight input — the spec does not prescribe input type beyond "questions").
- Skip at any point → default: start at Level 1 (§10).
- Complete all questions → answers determine a starting lesson index.

### Loading state
Minimal — this screen has no external data dependency to load.

### Empty state
N/A.

### Success state
Routes to Dashboard or directly into the determined starting lesson.

### Error state
No meaningful failure mode for this screen beyond generic navigation failure — there's no server round-trip mid-flow (answers are applied once, on completion/skip).

### Degraded/offline state
Unaffected by Firestore/Gemini reachability in the moment — the resulting `currentLevel`/`currentLessonId` write can queue via Firestore's offline persistence like any other write (PR-PLATFORM-04) and does not block the learner from proceeding locally.

### Accessibility notes
Standard form/question-flow accessibility: each question keyboard-navigable, skip action reachable without completing prior questions. See `docs/UX/ACCESSIBILITY.md` §4.

### Mobile vs desktop
No functional difference; one question (or a short group) per screen works well on both, though desktop has room to show progress through the 3–4 questions more explicitly (e.g., a step indicator).

### Open question
**IMPORTANT, flagged not resolved:** `PRODUCT_SPEC.md` never specifies the mapping from answers to a starting lesson index — only that skipping defaults to Level 1. The actual answer→level logic needs to be authored (likely alongside lesson content itself, per §35's milestone 6 guidance that lesson sequencing is content design work). See `docs/PRODUCT/USER_PERSONAS.md` §5.6.

---

## 6. Dashboard (Home)

### Purpose
The default landing screen for a learner with an existing session — orients them and gets them back into practice in one tap.

### User goal
"What should I do today, and how am I doing overall?"

### Entry point
Default screen on app open for any learner (anonymous or signed-in) past onboarding; also reachable via the Home tab at any time.

### Key elements / layout
Per §10, exactly these elements: greeting, streak, XP, "Continue lesson" CTA, today's practice card, one weak-area callout, quick links to Piano and Tutor. Components: `StreakBadge`, `XPBar`, `DashboardCard` (used for the Continue-lesson CTA, today's practice card, and the weak-area callout — three instances of the same reusable shell, per `docs/UX/COMPONENT_SPECIFICATIONS.md`).

### Primary interactions
- Tap "Continue lesson" → Lesson Player, resuming at `UserProfile.currentLessonId`.
- Tap today's practice card → same destination as above, or the specific review content the deterministic daily-plan generator selected (next incomplete lesson + review of last 3 misses, PR-AI-04).
- Tap weak-area callout → a relevant drill in Practice Hub, or the Tutor pre-loaded with that weak area as context (either is a reasonable implementation; §10 doesn't specify the exact destination, only that the callout exists).
- Tap Piano/Tutor quick links → navigate directly to those tabs.

### Loading state
Skeleton/placeholder for streak, XP, and the practice card while `UserProfile` loads via its `onSnapshot` subscription (§21) — never render zeroed-out streak/XP as if they were real values before the read resolves.

### Empty state
Explicit in §10: **first-ever visit** shows "Start your first lesson" in place of streak/stats — not a zero-streak/zero-XP dashboard, a genuinely different empty-state layout, since a brand-new learner has nothing to show yet and showing "0-day streak" reads as discouraging rather than neutral.

### Success state
All elements populated from `UserProfile` (one read, per §17/§22's flattened-document optimization — the dashboard must not issue N per-lesson reads).

### Error state
If the `UserProfile` read fails outright (not just "offline," an actual error), show a retry affordance on the affected card rather than the whole screen — degrade element-by-element where possible.

### Degraded/offline state
The degraded-service banner (§14) renders when Firestore is unreachable; last-known-good streak/XP/progress values (from Firestore's local cache, since offline persistence is enabled per PR-PLATFORM-04) continue to display rather than disappearing, with the banner communicating that this is cached/pending-sync data.

### Accessibility notes
Streak/XP/weak-area information is conveyed with text, not just iconography or color (§24's non-color-alone rule applies to gamification elements too, not only correct/incorrect key feedback). See `docs/UX/ACCESSIBILITY.md` §4.

### Mobile vs desktop
Mobile: cards stack vertically, bottom tab bar for nav. Desktop: left rail nav; cards may arrange in a grid rather than a single column, given the extra width.

---

## 7. Learn (Lesson Path)

### Purpose
Shows the learner's structured curriculum as a navigable whole, so they can see where they are and what's ahead — not just the single next lesson the Dashboard CTA jumps to.

### User goal
"Where am I in the course, and what's coming next?"

### Entry point
Learn tab (bottom nav / left rail).

### Key elements / layout
Linear list/tree of levels/lessons with three visual states per item: locked, unlocked, completed (§10). Given §9's note that Practice lives "inside" Learn, this screen's layout should also carry the Practice Hub entry point — see `docs/UX/NAVIGATION.md` §4 for the flagged open question on exactly where.

### Primary interactions
- Tap an unlocked or completed lesson → Lesson Player, opening that specific lesson (completed lessons remain replayable — a learner may want to redo a passed lesson, though §10 doesn't explicitly confirm this; treating completed lessons as tappable/replayable is consistent with "never a dead end" and is the reasonable default).
- Locked lessons are visibly locked but not hidden — the learner can see what's ahead, just not enter it yet, which supports the sense of a structured path rather than an opaque one.

### Loading state
Skeleton list while the `lessons` collection (public-read, §17) and the learner's `ProgressSummary` resolve.

### Empty state
N/A in the sense of "no lessons exist" (the MVP catalog is always populated via seed script, PR-LESSON-07) — but a brand-new learner's view, where every lesson except the first is locked, is a valid and expected initial state, not an error.

### Success state
Full list renders with correct lock state per lesson, computed from `ProgressSummary[lessonId].status` and each `Lesson.prerequisiteLessonId`/`passThreshold`.

### Error state
If the lesson list or progress fails to load, show a retry affordance; do not render a list with incorrect/stale lock states as if authoritative.

### Degraded/offline state
Since `lessons` content is public-read static-ish data, it is highly cacheable client-side; a learner should still see the path (from cache) with the degraded-service banner shown if Firestore is currently unreachable for a fresh read.

### Accessibility notes
Lock state is conveyed with text/icon, not color alone (a locked lesson needs a non-color "locked" indicator, e.g., a lock icon + label, consistent with §24). Each list item is a proper focusable, activatable control, not a div with a click handler only. See `docs/UX/ACCESSIBILITY.md` §4.

### Mobile vs desktop
Mobile: vertical scroll list. Desktop: could render as a wider tree/map layout given more space, though a simple vertical list is also acceptable — §10 says "list/tree," leaving the exact visualization open; no functional requirement forces a graph-style tree view.

---

## 8. Lesson Player

**The core screen of the product** — per §10, this is "The core screen." It gets the most detailed spec here accordingly.

### Purpose
Deliver one structured lesson end-to-end through the full core interaction loop (Learn → Demonstrate → Practice → Feedback → Repeat → Progress, `docs/UX/UX_PRINCIPLES.md` §3).

### User goal
"Teach me this specific thing, tell me clearly if I got it, and don't make me guess what to do next."

### Entry point
- Dashboard's "Continue lesson" CTA.
- Learn (Lesson Path), tapping an unlocked/completed lesson.
- Dashboard's "today's practice" card, when it points at a lesson.

### Key elements / layout
Per §10: piano fixed at the bottom/center of the screen; instruction and prompt area above it; a progress-through-lesson indicator (`LessonProgressIndicator` component); a hint button (`HintButton` component); an "ask the tutor why" entry point that opens the AI Tutor pre-loaded with this exercise's context.

### Primary interactions
- Watch/replay the demonstration.
- Play the exercise on the fixed piano (`PianoKeyboard` component).
- Request a hint (`HintButton`) — escalates hint level rather than repeating the same hint.
- On feedback, optionally tap "ask the tutor why" → AI Tutor (see `docs/PRODUCT/USER_FLOWS.md` Flow 3).
- On correct/lesson-complete, advance to the next step/lesson.

### States (explicit in §10 — this section expands each)

| State | What renders | Notes |
|---|---|---|
| `demonstrating` | Animated + audio demo on the piano; instruction text states the lesson's goal | Learner can typically replay this; it is not a one-shot animation that vanishes |
| `awaitingInput` | Piano is live; keys highlight in response to press, live, before any verdict is known | Purely local — Audio Layer + UI Layer only, no network |
| `correct` | `ScoreCard` renders the deterministic verdict (what was right, what if anything to refine); progress indicator advances | Verdict is the server-recomputed one (§17), even though a client-side optimistic preview may have rendered instantly first |
| `incorrectWithHint` | `ScoreCard` renders the specific miss (named notes, not just "wrong"); an escalated hint is offered; retry is immediate | Never a hard stop — this is the Repeat stage of the core loop |
| `lessonComplete` | Full lesson summary; XP/streak update; "what's next" nudge | This is the Progress stage of the core loop |

### Loading state
Piano samples for this lesson's required octave(s) must finish loading before the screen is marked interactive — §22 explicitly requires a loading state here, "never a silently broken piano." Show a clear loading indicator, not a piano that renders but produces no sound.

### Empty state
N/A — a lesson always has content by construction (seeded, validated at build/seed time per `TYPES_AND_CONTRACTS.md` §4.14's `ScoringWeights` validation note).

### Success state
`lessonComplete`, as above.

### Error state
- If piano/audio assets fail to load, show a retry affordance distinct from "tap to enable sound" (which is a different, expected gesture-gating state, not an error).
- If the `getExerciseFeedback` call itself fails outright (not just Gemini's portion — see Degraded below), the learner's played notes should not be lost; retry the submission before declaring an error, since losing a just-completed attempt due to a transient network blip would be a serious "learner blocked" violation.

### Degraded/offline state
Two independent degradations apply here, and they must be handled separately (see `docs/PRODUCT/USER_FLOWS.md` Flow 4.2):
1. **Firestore unreachable at submission time:** the attempt queues via offline persistence (PR-PLATFORM-04); the learner should still see *a* verdict — client-side optimistic scoring can render provisionally with a clear "pending sync" indication, reconciled once the server responds. §10/§29 do not fully specify optimistic-verdict UI treatment; this is a reasonable elaboration consistent with "never blocked," not a new feature.
2. **Gemini unreachable for the feedback explanation:** the deterministic verdict (accuracy, missed/extra notes, composite score, pass/fail) still renders fully — only the natural-language prose explanation is missing, replaced by a deterministic canned line (§29: disabling Gemini entirely leaves every other screen fully functional).

### Accessibility notes
- Score/feedback text is announced via an ARIA live region the moment it's available (§24) — a screen-reader user must not have to hunt for the result.
- Progress-through-lesson indicator has an accessible text equivalent (e.g., "Step 2 of 5"), not just a visual bar.
- Per §24's explicit, honest caveat: full accessible parity on the piano-*playing* interaction itself is not claimed here — see `docs/UX/ACCESSIBILITY.md` §5 for what is and isn't achievable on this specific screen.

### Mobile vs desktop
Mobile: piano fixed at the bottom of the viewport, instruction/feedback area above scrolls if needed, following §9's "piano is the one fixed element" principle exactly. Desktop: more room allows instruction/feedback and the piano to coexist without either scrolling, and the piano can show a wider octave range without needing pan (see `docs/UX/RESPONSIVE_DESIGN.md`).

---

## 9. Practice Hub

### Purpose
A drill-only surface for a learner who wants to practice something already taught, without being taught something new. Per §10: "this is where a learner goes to drill, not to be taught something new."

### User goal
"Let me practice X specifically, right now, without lesson scaffolding around it."

### Entry point
From within Learn — see `docs/UX/NAVIGATION.md` §4 for the flagged open question on the exact entry-point placement.

### Key elements / layout
Entry points (cards or list items) to: Note Identification game, scale drill, chord drill, free play (§10).

### Primary interactions
Tap any entry point → routes to that specific drill mode (Note ID Game has its own full spec below; scale/chord drills reuse Lesson-Player-like mechanics but without lesson framing/instruction text, and without prerequisite gating — they are drawn from `Exercise` pool config, not `Lesson` content, per `TYPES_AND_CONTRACTS.md` §4.3).

### Loading state
Minimal — this screen is mostly static entry points into `Exercise` static config (§17: exercise definitions are bundled client config, not a Firestore read).

### Empty state
N/A — the drill catalog is always populated (static config).

### Success state
All four entry points render and are tappable.

### Error state
No meaningful screen-level failure mode beyond generic navigation.

### Degraded/offline state
Fully unaffected by Firestore/Gemini reachability for browsing/entering drills, since `Exercise` definitions are static client config, not a network read (§17). Only score-persistence at the end of a drill session depends on Firestore, same as any other exercise (PR-PLATFORM-04/06).

### Accessibility notes
Standard focusable/activatable list/card pattern; each entry point has a clear accessible name describing the drill type. See `docs/UX/ACCESSIBILITY.md` §4.

### Mobile vs desktop
Mobile: stacked cards. Desktop: grid of four cards given the extra width. No functional difference.

---

## 10. Note Identification Game

### Purpose
Build fast note-recall speed through repeated, quick-turnaround prompts, with visible adaptive difficulty. Per §5's user stories, this is a distinct drill from lesson-based note learning — pure recall speed, not conceptual introduction.

### User goal
"Test and improve how fast I can find a named note."

### Entry point
Practice Hub.

### Key elements / layout
Piano + prompt area ("Which key is C?"), score/streak counter, live difficulty indicator (§10). Components: `PianoKeyboard`, `ScoreCard` (or a lightweight running-score variant of it).

### Primary interactions
- Prompt shown → learner presses a key → immediate correct/incorrect feedback (locally, from the engine) → next prompt.
- Difficulty adjusts after a run of correct/incorrect answers, with a visible step-up/step-down indicator (§10, PR-DRILL-02) — the learner must be able to tell *why* it suddenly feels harder or easier.

### Loading state
Piano must be loaded/ready before the first prompt fires, same audio-preload requirement as Lesson Player (§22).

### Empty state
N/A — the game always has a pool of prompts (`Exercise.pool`, static config).

### Success state
Prompt/answer/score loop running normally, with visible score and streak counters updating each round.

### Error state
No meaningful mid-round failure mode beyond generic input/audio errors already covered under Piano Free Play/Lesson Player.

### Degraded/offline state
The round-by-round play loop is entirely local (engine + audio), so it is unaffected by Firestore/Gemini reachability. Only the end-of-session score write depends on connectivity, and that queues via offline persistence like any other session write (PR-PLATFORM-04).

### Accessibility notes
The prompt text ("Which key is C?") is itself the accessible content a screen-reader user needs — it should be announced via ARIA live region on each new round, not just visually updated. Difficulty step-up/step-down announcements should also be non-visual-only (§24's non-color-alone principle extends here: the difficulty *change* itself needs a non-visual signal, not just a color/animation cue). See `docs/UX/ACCESSIBILITY.md` §4.

### Mobile vs desktop
Same piano-fixed-at-bottom pattern as Lesson Player; no functional difference beyond the general responsive piano behavior (`docs/UX/RESPONSIVE_DESIGN.md`).

---

## 11. Piano Free Play

### Purpose
An unscaffolded, exploratory piano surface — both a genuine practice mode for confident learners and the try-before-signup surface for brand-new visitors.

### User goal
Two distinct goals depending on entry point: (a) new visitor — "let me just try this before committing to anything"; (b) existing learner — "let me just play, without a lesson telling me what to do."

### Entry point
- Landing's "Try the piano now" (anonymous session).
- Practice Hub's free-play entry point.
- Piano tab directly (bottom nav / left rail).

### Key elements / layout
Full piano, live note/chord recognition surfaced as a toast (e.g., "You played: C major"), zero scaffolding — no instruction text, no progress indicator, no lesson framing (§10).

### Primary interactions
Press any key/combination → sound plays locally → `ToastNotification` surfaces recognized note/chord name, ephemeral and non-blocking (does not require dismissal to keep playing).

### Loading state
Same audio-preload requirement as any piano screen (§22); since this may be the very first screen a brand-new anonymous visitor sees, its load-time performance is disproportionately important — this is the screen the §2 north star ("first correctly-identified note within two minutes") is most directly staked on.

### Empty state
N/A — the piano itself is always "populated."

### Success state
Piano fully interactive, recognition toasts firing correctly for both single notes and recognized chords.

### Error state
Audio/asset load failure handled the same as Lesson Player's error state (retry affordance, never a silently broken piano).

### Degraded/offline state
This screen is the purest expression of "learner never blocked" (§2, `docs/UX/UX_PRINCIPLES.md` §5) — it has no Firestore or Gemini dependency in its core interaction at all. It should be fully functional even when the degraded-service banner is showing for other reasons elsewhere in the app.

### Accessibility notes
Recognition toasts are announced via ARIA live region (polite, not assertive — they're informative, not urgent, and a rapid sequence of assertive announcements while a learner is experimenting would be actively unpleasant). See `docs/UX/ACCESSIBILITY.md` §4/§5 for the broader caveat this screen is the clearest example of: a screen-reader user can use every non-piano part of this screen fully, but cannot practically sight-read the live visual keyboard the way a sighted user exploring freely does.

### Mobile vs desktop
Mobile: 1–1.5 octaves visible with horizontal pan (unvalidated, see `docs/UX/RESPONSIVE_DESIGN.md`). Desktop: meaningfully more of the keyboard visible at once without panning, which changes the free-play experience qualitatively (easier to explore wider intervals/chords) — worth naming as a real mobile-vs-desktop experience gap, not just a layout reflow, since chord exploration specifically benefits from seeing more keys at once.

---

## 12. AI Tutor

### Purpose
Grounded, conversational help — the one place in the product where a learner interacts with Gemini directly (through the server proxy).

### User goal
"Answer my specific question about what I just did, in plain language."

### Entry point
- Tutor tab (bottom nav / left rail) — general entry, not pre-loaded with a specific exercise.
- Lesson Player's "ask the tutor why" — pre-loaded with the just-completed exercise's context.
- Dashboard's quick link to Tutor.

### Key elements / layout
Chat interface (`TutorChatBubble` components); quick-suggestion chips (e.g., "What should I practice today?"); a persistent, subtle indicator that the learner's recent-session context is loaded (§10) — this indicator is what keeps the tutor from feeling like "a blank generic chatbot" per §10's own framing.

### Primary interactions
- Type a free-form question, or tap a suggestion chip → submit → response streams/renders in chat.
- Tap a `TutorResponse.suggestedAction` deep-link hint (if present) → navigates to the relevant screen/drill.

### Loading state
A "thinking" state while the Cloud Function call is in flight — explicitly acceptable to take 1–3 seconds here since the call is user-initiated and gated (§22); the loading indicator should communicate "thinking," not look like a stalled/broken UI.

### Empty state
A first-ever visit to this screen (no chat history yet) should show the quick-suggestion chips prominently, since there's nothing else to show — this isn't specified explicitly in §10 but follows directly from "quick-suggestion chips" being named as a persistent element rather than something that only appears after the first message.

### Success state
Chat renders normally, context indicator visible, response grounded in `TutorRequest.recentExerciseContext`/`AIMemorySummary` as applicable.

### Error state
A validation failure on a single response (malformed/empty/too-long per §15's guardrails) shows a deterministic canned message for that turn — the chat itself continues to function; this is not the same as the full degraded state below.

### Degraded/offline state
**This is the one screen where Gemini's unavailability removes real functionality** (§34's MVP test explicitly names this as the sole exception to "everything still works if Gemini is unreachable"). Per §10: a clear "temporarily unavailable, here's what to practice instead" fallback card renders, built entirely from deterministic data — never a dead end, never an infinite "thinking" spinner. See `docs/PRODUCT/USER_FLOWS.md` Flow 4.1 for the exact sequence.

### Accessibility notes
Chat messages announced via ARIA live region as they arrive; the "thinking" state itself should be announced (e.g., "Tutor is responding") so a screen-reader user isn't left wondering if their message was received. `TutorResponse.tier` and `.cached` are internal/cost-monitoring fields and must never be exposed in the accessible name or visible text (`TYPES_AND_CONTRACTS.md` §4.10 explicitly marks these as not shown to the user). See `docs/UX/ACCESSIBILITY.md` §4.

### Mobile vs desktop
Standard chat-UI responsive behavior; no piano is present on this screen, so the "piano fixed" constraint doesn't apply here — this screen can use the full viewport height for chat on both mobile and desktop.

---

## 13. Progress / Stats

### Purpose
A deeper view of a learner's history than the Dashboard's single weak-area callout provides — "how am I actually doing, over time."

### User goal
"Show me my trend, not just today's snapshot."

### Entry point
Profile screen (§10: this screen "lives under Profile, not a top-level tab" — see `docs/UX/NAVIGATION.md` §3 for why).

### Key elements / layout
Accuracy over time, weak notes/chords, lessons completed, practice minutes (§10).

### Primary interactions
Primarily a read-only view; a weak note/chord shown here could reasonably link out to a relevant Practice Hub drill (consistent with the Dashboard's weak-area callout doing the same), though §10 doesn't mandate this specific interaction.

### Loading state
Skeleton while the `practiceSessions` query resolves — this is the one MVP screen whose data shape specifically needs the composite index named in §17 (`(userId, completedAt desc)` on `practiceSessions`), since "accuracy over time" implies an ordered range query, not a single-document read like the Dashboard.

### Empty state
A learner with zero completed `practiceSessions` sees an explicit "nothing to show yet — complete your first lesson" message, not an empty chart or a chart rendering against zero data as if it were meaningful.

### Success state
Charts/lists populated from aggregated `practiceSessions` data.

### Error state
If the query fails, show a retry affordance; do not render a partial/stale chart as current.

### Degraded/offline state
Degraded-service banner applies if Firestore is unreachable; previously-synced data (from local cache) can still display, clearly marked as potentially stale, consistent with the same pattern used on Dashboard.

### Accessibility notes
Any chart/graph used for "accuracy over time" needs a non-visual equivalent (e.g., a data table or textual summary) — a purely visual line chart with no text alternative would fail §24's accessibility bar for this screen. This is worth calling out explicitly since charts are a common accessibility blind spot. See `docs/UX/ACCESSIBILITY.md` §4.

### Mobile vs desktop
Mobile: stacked, simplified views (e.g., a trend summary rather than a dense chart). Desktop: room for a fuller chart and more simultaneous detail. No functional difference in underlying data.

---

## 14. Profile / Settings

### Purpose
Account management and app preferences — the one screen holding both identity-level actions (sign out, delete, export) and piano-specific preferences (keyboard mapping, audio defaults).

### User goal
"Manage my account, and set the app up the way I want it."

### Entry point
Profile tab (bottom nav / left rail).

### Key elements / layout
Account info; keyboard-mapping customization; audio/volume defaults; sign out; data export/delete (§10). Also the home for the Progress/Stats view (§13 above) and, per §12's legal requirement, the Salamander Grand Piano attribution credit (PR-PIANO-16) — §10 doesn't explicitly place the attribution credit on this screen by name, but §12 says "Settings/About screen," and Profile/Settings is the only settings-type screen in the MVP screen list, so this document places it here as the clear intended location.

### Primary interactions
- Edit keyboard mapping (writes to `UserProfile.settings.keyboardMapping`, client-writable per `TYPES_AND_CONTRACTS.md` §4.15's field-ownership table).
- Adjust volume/sustain defaults (same client-writable settings fields).
- Sign out.
- Request data export (`exportUserData` Cloud Function, PR-AUTH-07).
- Request account deletion (`deleteAccount` Cloud Function, PR-AUTH-08) — this is a destructive, irreversible action and should require explicit confirmation, consistent with standard practice for account deletion even though §10/§18 don't spell out a confirmation-dialog requirement in UI terms.

### Loading state
Skeleton for account info while `UserProfile` loads (likely already cached from Dashboard's subscription).

### Empty state
N/A — a signed-in learner always has a profile document by construction.

### Success state
All settings sections editable and reflecting current values.

### Error state
A failed settings write (e.g., keyboard-mapping save) should show an inline error and not silently discard the learner's change — since `settings.*` are the one part of `UserProfile` the client writes directly (`TYPES_AND_CONTRACTS.md` §4.15), a failed write here is a real, visible loss of a preference if not surfaced.

### Degraded/offline state
Settings edits queue via Firestore's offline persistence like any other client write; sign-out and local-only actions work regardless of connectivity. Data export/deletion, being Cloud Function calls (not direct Firestore writes), should show a clear "can't process this right now" message if the Function is unreachable, rather than silently failing — these are exactly the kind of action where a silent failure would be a serious trust problem (a learner believing their data was deleted/exported when it wasn't).

### Accessibility notes
Settings controls (toggles, selects) use proper semantic form elements with labels; the keyboard-remapping UI in particular needs careful keyboard-accessible design since it is, by definition, being edited by someone interacting via keyboard already. See `docs/UX/ACCESSIBILITY.md` §4.

### Mobile vs desktop
Standard settings-screen responsive behavior; no functional difference.

---

## 15. Error / Degraded-Service States

Not a standalone screen — a cross-cutting pattern applied on top of any screen, per §10. Specified here as its own entry because it needs one canonical definition every other screen in this document refers back to.

### Purpose
Communicate a Firestore or Gemini outage honestly and calmly, without ever blocking the piano or current lesson.

### Trigger
Firestore unreachable (detected via the Firestore SDK's connection state, or a failed read/write that offline persistence can't resolve locally) or Gemini unreachable (a Cloud Function call returning an error, or timing out).

### Presentation
A persistent, non-alarming banner — §10's own example wording: **"Playing offline — your progress will sync later."** Rendered via the `DegradedServiceBanner` component (`docs/UX/COMPONENT_SPECIFICATIONS.md`). "Non-alarming" is explicit in the source spec's own wording — this is not an error-red, iconography-heavy alert; it reads as a calm status notice.

### Scope of effect
- The piano and the current lesson **keep working locally regardless** (§10) — this banner is informational, never a modal, never something that blocks interaction underneath it.
- On the AI Tutor screen specifically, Gemini-unreachable additionally produces the dedicated fallback card described in §12 above — the ambient banner and the Tutor-specific fallback card are two different UI elements serving the same underlying principle at different levels of specificity (app-wide ambient notice vs. screen-specific actionable fallback).

### Clearing
The banner clears automatically once the underlying service is reachable again — no manual dismissal is described in §10, and given its non-blocking, ambient nature, auto-clearing on recovery (rather than requiring a manual dismiss that a learner might not think to do) is the safer default consistent with "never blocked."

### Minor flagged ambiguity
§10 groups "Firestore or Gemini is unreachable" under one banner description with one example wording clearly about progress-sync (a Firestore-shaped concern). It does not fully specify whether Gemini-only unavailability (with Firestore fine) should trigger this same ambient banner app-wide, or whether it should surface *only* as the AI-Tutor-screen-specific fallback card. This document's working assumption, used consistently above: Gemini-only unavailability surfaces primarily through the AI-Tutor-specific fallback card and through graceful degradation of the exercise-feedback prose (§29); the ambient app-wide banner is reserved primarily for Firestore/connectivity-shaped issues, since its example wording ("progress will sync later") doesn't fit a Gemini-only outage. Flagged as **NON-BLOCKING** — a content/copy decision, not a structural one, since the underlying component (`DegradedServiceBanner`) can carry either message with a `serviceType` prop regardless of which interpretation is chosen.

### Accessibility notes
The banner is announced via an ARIA live region (polite) when it first appears, so a screen-reader user is informed without the announcement being disruptive; it does not trap focus, since it is never a modal.

## 16. Cross-References

- `docs/UX/COMPONENT_SPECIFICATIONS.md` — every component named above.
- `docs/UX/ACCESSIBILITY.md` — the accessibility rules applied per-screen throughout this document.
- `docs/UX/NAVIGATION.md` — entry-point/nav-structure context, including the flagged Practice Hub placement question.
- `docs/UX/RESPONSIVE_DESIGN.md` — the mobile-vs-desktop specifics touched on throughout, especially for Piano Free Play and Lesson Player.
- `docs/PRODUCT/USER_FLOWS.md` — end-to-end sequences that traverse multiple screens from this document.
- `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — every data shape referenced above.

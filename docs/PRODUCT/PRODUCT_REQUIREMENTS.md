# Product Requirements

## 1. Purpose

The full functional requirements list for AI Piano Tutor, derived from `docs/PRODUCT_SPEC.md` §6 (Core Features) and §7 (MVP Scope), with every other section of the spec pulled in where it adds implementation-critical detail to a requirement already named in §6/§7. This is the **master requirement list** — `docs/REQUIREMENTS_TRACEABILITY.md` maps against the stable IDs defined here. Do not renumber or reuse an ID once assigned; deprecate instead.

## 2. Scope

Every requirement implied by §6/§7, tagged `[MVP]` or `[Future — Phase N]`. Phase 2/3/4 items are listed **so they can be traced and deliberately deferred**, not because they ship in MVP — see `docs/PRODUCT/MVP_SCOPE.md` for the authoritative in/out boundary and `docs/PRODUCT/FUTURE_ROADMAP.md` for phase detail.

## 3. ID Scheme and Organization

Format: `PR-<AREA>-<NN>`. Areas mirror §6's own five groupings (Piano, Learning Modes → split further below, AI Tutor, Progress & Gamification, Account & Data) plus three groups this document adds because §7 and the non-negotiable architecture rule call them out as distinct, individually-acceptance-tested concerns rather than sub-bullets of something else:

- **PR-ENGINE** — the deterministic music engine is its own §7 MVP bullet ("Deterministic music engine...") and the subject of the non-negotiable architecture rule. Burying it inside PR-PIANO or PR-LEARN would make it untraceable as the single most safety-critical requirement group in the product.
- **PR-A11Y** and **PR-PLATFORM** — §7's closing bullet ("Mobile-first responsive UI; baseline accessibility...") bundles two independently-testable concerns into one sentence; splitting them gives each its own acceptance trail.

This is an organizational choice for traceability, not a reinterpretation of scope — every requirement below still cites its source section(s).

| Area code | Feature area | Primary source |
|---|---|---|
| PR-AUTH | Account & Auth | §6 Account & Data, §18 |
| PR-PIANO | Virtual Piano & Input | §6 Virtual Piano, §11, §12 |
| PR-ENGINE | Deterministic Music Theory Engine | §7, §13, §14 |
| PR-LESSON | Structured Lessons / Learn Path | §6 Learning Modes, §7, §17 |
| PR-DRILL | Practice Hub, Note ID Game, Free Play | §6 Learning Modes, §10 |
| PR-AI | AI Tutor | §6 AI Tutor, §14, §15, §19 |
| PR-PROGRESS | Progress, Gamification, Dashboard | §6 Progress & Gamification, §10, §17 |
| PR-A11Y | Accessibility | §7, §24 |
| PR-PLATFORM | Mobile-first, Resilience, Performance | §7, §9, §22, §23 |
| PR-CONTENT | Content Operations | §5 Content Ops epic, §8 |

**Out of scope for this document:** the detailed Firestore security-rules and Cloud-Function trust-boundary specification (owned by `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` and `docs/ARCHITECTURE/AI_ARCHITECTURE.md`, referenced from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`); this list states *what* the product must do, cross-referencing security mechanics only where §6/§7 name them directly as a feature (e.g., data export/deletion, rate limiting).

## 4. Non-Negotiable Rule (applies across PR-ENGINE and PR-AI)

Per `docs/PRODUCT_SPEC.md` §14: **Gemini must never determine whether a note or chord is correct, calculate accuracy/timing/scores, determine lesson completion/XP/streaks, or validate user-submitted performance data.** That is `packages/music-engine`'s job — deterministic, server-verified. Gemini only explains a verdict it has already been given. This rule is restated as PR-ENGINE-15 and PR-AI-08 below and must not be weakened by any requirement elsewhere in this document.

## 5. Requirements

### 5.1 PR-AUTH — Account & Auth

| ID | Requirement | Tag | Source |
|---|---|---|---|
| PR-AUTH-01 | A visitor can start using the piano via an anonymous Firebase Auth session, with zero signup friction. | [MVP] | §6, §18, §5 (Onboarding & Auth epic) |
| PR-AUTH-02 | A learner can sign in/up with Google in one tap. | [MVP] | §6, §18, §5 |
| PR-AUTH-03 | A learner can sign in/up with email and password. | [MVP] | §6, §18 |
| PR-AUTH-04 | An anonymous session's progress is preserved when the learner signs up, via `linkWithCredential` (anonymous-to-linked upgrade) — never reset to zero. | [MVP] | §6, §18, §10 (Sign in/Sign up state) |
| PR-AUTH-05 | If the email used at sign-up is already registered to a different account, the linking flow surfaces a resolvable conflict state rather than silently failing or silently overwriting. | [MVP] | §10 (Sign in/Sign up: "State to handle: linking conflict") |
| PR-AUTH-06 | Every Cloud Function call is authenticated by verifying the caller's Firebase ID token server-side (Admin SDK); a client-supplied `uid` is never trusted alone. | [MVP] | §18, §19 |
| PR-AUTH-07 | A learner can export their full account data on request. | [MVP] | §6 ("required for any real launch, not optional polish"), §18 |
| PR-AUTH-08 | A learner can delete their account and its full data subtree on request. | [MVP] | §6, §18 |
| PR-AUTH-09 | Progress syncs via Firestore across devices for the same signed-in account. | [MVP] | §6, §7, §17 |
| PR-AUTH-10 | Additional auth providers (e.g., Apple Sign-In) can be added without a data-model change. | [Future — Phase unspecified] | §18. §18 states this is additive/low-effort but does not assign it to a numbered phase in §8's roadmap — flagged in project open questions, non-blocking. |

### 5.2 PR-PIANO — Virtual Piano & Input

| ID | Requirement | Tag | Source |
|---|---|---|---|
| PR-PIANO-01 | Playable virtual keyboard renders white and black keys correctly across the visible range. | [MVP] | §6 |
| PR-PIANO-02 | Note labels on keys are toggleable on/off. | [MVP] | §6 |
| PR-PIANO-03 | Keys animate on press and on hover (desktop). | [MVP] | §6 |
| PR-PIANO-04 | Mouse input is supported and produces identical engine/audio behavior to touch and computer-keyboard input for the same note. | [MVP] | §6, §11, §29 |
| PR-PIANO-05 | Touch input is supported and produces identical engine/audio behavior to mouse and computer-keyboard input for the same note. | [MVP] | §6, §11, §29 |
| PR-PIANO-06 | Computer-keyboard input is supported using the default mapping (A/W/S/E/D/F/T/G/Y/H/U/J/K for C–C, matching virtualpiano.net convention), stored as remappable data rather than hardcoded per-component. | [MVP] | §11, §5 ("learner on a laptop" story) |
| PR-PIANO-07 | A learner can remap computer-keyboard keys from Settings. | [MVP] | §11 ("the settings screen can let a learner remap it") |
| PR-PIANO-08 | Visible octave range is adjustable, with ~2–3 octaves visible and horizontal pan on typical viewports. | [MVP] | §6, §7 |
| PR-PIANO-09 | Sustain can be toggled on/off, with a settings-level default. | [MVP] | §6, §7, §17 (`settings.sustainDefault`) |
| PR-PIANO-10 | Volume is adjustable, with a settings-level default. | [MVP] | §6, §7, §17 (`settings.volume`) |
| PR-PIANO-11 | Touch targets are ≥44px per key on phone-sized viewports. | [MVP] | §9 |
| PR-PIANO-12 | Audio is produced via Tone.js `Tone.Sampler` loaded with Salamander Grand Piano samples; only the octave(s) the current lesson needs load before the screen is interactive, with the rest prefetched in the background. | [MVP] | §12, §22 |
| PR-PIANO-13 | Audio is compressed (OGG/MP3), not served as raw WAV. | [MVP] | §12 |
| PR-PIANO-14 | On browsers that block audio until a user gesture, a visible "tap to enable sound" affordance creates/resumes the `AudioContext` — the piano is never silently broken on first load. | [MVP] | §12, §29 |
| PR-PIANO-15 | Key-press-to-audible-sound latency is under ~50ms on a mid-range Android device and a current iPhone. | [MVP] | §12, §29 |
| PR-PIANO-16 | A visible attribution credit for Salamander Grand Piano (CC BY 3.0, Alexander Holm) appears on the Settings/About screen, and a `NOTICE` file exists in the repo. | [MVP] | §12 (legally required, not optional) |
| PR-PIANO-17 | MIDI keyboard input via the Web MIDI API is supported on browsers where `'requestMIDIAccess' in navigator`, normalized into the same `NoteEvent` shape as every other input adapter. | [Future — Phase 2] | §6, §8, §11 |
| PR-PIANO-18 | On browsers without Web MIDI support (verified: Safari macOS/iOS, permanently as of Aug 2026), the app shows a clear "MIDI isn't available in this browser" message — never a silent failure, never assumed as a baseline input method. | [Future — Phase 2, messaging ships alongside PR-PIANO-17] | §11, §30 |

### 5.3 PR-ENGINE — Deterministic Music Theory Engine

`packages/music-engine` requirements. See §4 above — this group carries the non-negotiable architecture rule.

| ID | Requirement | Tag | Source |
|---|---|---|---|
| PR-ENGINE-01 | MIDI note numbers (0–127) are the sole canonical representation of pitch; note name (scientific pitch notation) and frequency (`f = 440 × 2^((midi−69)/12)`) are derived display values, never stored or compared as source of truth. | [MVP] | §13, `TYPES_AND_CONTRACTS.md` §3 |
| PR-ENGINE-02 | Given an expected note sequence and a played `NoteEvent[]`, the engine identifies correct, missed, and extra notes. | [MVP] | §13, §29 (worked example) |
| PR-ENGINE-03 | All 12 major triads are identified correctly regardless of inversion or the order notes are played, via pitch-class set matching (`{0,4,7}` tested against all 12 roots). | [MVP] | §13, §29 |
| PR-ENGINE-04 | All 12 minor triads are identified correctly regardless of inversion or note order, as an engine capability. **Note:** §29's acceptance criteria and `ChordDefinition.quality` (`TYPES_AND_CONTRACTS.md` §4.12, "MVP ships major + minor only") both require minor-triad *detection* in MVP even though §6/§7 scope minor-triad *lesson content* to Phase 2 — see open question, flagged separately (not silently resolved). | [MVP — engine capability only; lesson content is Future, see PR-LESSON-15] | §29, `TYPES_AND_CONTRACTS.md` §4.12 |
| PR-ENGINE-05 | Chord inversion is identified as a secondary step after pitch-class match, by comparing the lowest sounding note to the matched set. | [MVP] | §13 |
| PR-ENGINE-06 | Major-scale attempts are validated using ordered comparison when sequence matters ("play it ascending") and unordered set comparison when it doesn't ("did you hit all seven notes"). | [MVP] | §13 |
| PR-ENGINE-07 | Note correctness = correct notes played ÷ required notes; wrong/extra notes are penalized, not ignored. | [MVP] | §13 |
| PR-ENGINE-08 | Timing is scored per note, bucketed: ≤100ms "on time," 100–250ms "slightly off," >250ms "late/early" — each bucket surfaced per note, not averaged away. | [MVP] | §13 |
| PR-ENGINE-09 | Completeness is scored independently of timing — were all required notes attempted at all. | [MVP] | §13 |
| PR-ENGINE-10 | Consistency is scored as variance in timing deviation, both within one attempt and across the learner's last N attempts of that exercise. | [MVP] | §13 |
| PR-ENGINE-11 | A composite score combines correctness/timing/consistency using weights sourced from `Lesson.scoringWeights` / `Exercise.scoringWeights` (never hardcoded in the engine), clamped 0–100. Weights must sum to 1.0 (±0.001). | [MVP] | §13, `TYPES_AND_CONTRACTS.md` §4.14 |
| PR-ENGINE-12 | The worked example — expected C-D-E-F-G, played C-D-E-G — returns exactly `{missedNotes: ['F'], accuracy: 0.8}`. | [MVP] | §13, §29 (exact acceptance criterion) |
| PR-ENGINE-13 | `packages/music-engine` has zero import of `firebase/*`, `react`, `next`, or any Gemini/AI SDK; it is a pure function of `(NoteEvent[], Lesson \| Exercise) → ExerciseVerdict`. | [MVP] | §11, §27, `TYPES_AND_CONTRACTS.md` §5.2 |
| PR-ENGINE-14 | A Cloud Function re-validates every submitted `ExerciseAttempt` server-side by re-running the engine against the independently-loaded `Lesson`/`Exercise` definition — a client-computed verdict is never trusted as final. | [MVP] | §17, §19, `TYPES_AND_CONTRACTS.md` §4.4/§4.5 |
| PR-ENGINE-15 | **Gemini never determines note/chord/scale correctness, computes accuracy/timing/completeness/consistency/composite scores, determines lesson completion/XP/streak, or validates submitted performance data.** This is engine-only and non-negotiable. | [MVP] | §14 (boundary table) |
| PR-ENGINE-16 | Acoustic pitch detection (listening to microphone/speaker output to infer what was played) is explicitly out of scope for this engine — it operates on symbolic `NoteEvent` input only. | [Future — Phase 3] | §13, §8 |
| PR-ENGINE-17 | Unit test coverage on chord/scale/scoring logic is ≥90%. | [MVP] | §26, §29 |

### 5.4 PR-LESSON — Structured Lessons / Learn Path

| ID | Requirement | Tag | Source |
|---|---|---|---|
| PR-LESSON-01 | ~12–15 structured lessons cover: keyboard orientation → note names → octaves → simple melodies → basic rhythm → intro to major triads, in a single linear sequence (no branching paths in MVP). | [MVP] | §7, §17 |
| PR-LESSON-02 | A guided "Learn the Keys" flow uses a glow-and-confirm interaction to teach key locations (e.g., find and confirm Middle C). | [MVP] | §6, §5 |
| PR-LESSON-03 | Each lesson opens with a demonstration: an animated + audio demo plays on the same piano instrument the learner will use, before the learner attempts it. | [MVP] | §9 (core loop: Demonstrate) |
| PR-LESSON-04 | A wrong key press is treated as a normal part of learning, not a failure state — no punitive framing, no hard stop. | [MVP] | §5, §9 |
| PR-LESSON-05 | On an incorrect attempt, the learner can retry with escalating hints; the lesson never hard-stops a learner from progressing toward a correct attempt. | [MVP] | §9 (core loop: Repeat), §10 (Lesson Player states) |
| PR-LESSON-06 | A lesson unlocks only after its prerequisite lesson's deterministic pass threshold (`Lesson.passThreshold`) is met. | [MVP] | §10, `TYPES_AND_CONTRACTS.md` §4.2 — **note:** no default `passThreshold` value has been ratified yet (inherited from `TYPES_AND_CONTRACTS.md` OQ-01); flagged as blocking for lesson-content authoring. |
| PR-LESSON-07 | Lesson content is authored as static seed JSON and loaded into Firestore by a seed script; it is public-read and not client-writable under any circumstance. | [MVP] | §17, §19 |
| PR-LESSON-08 | Guided scale lessons cover C major plus 3–4 more common keys. | [MVP] | §6, §7 |
| PR-LESSON-09 | A scale is demonstrated (visually and audibly) before the learner is asked to attempt it. | [MVP] | §5 |
| PR-LESSON-10 | On a scale attempt, the learner is told specifically which note(s) were missed, not just "wrong." | [MVP] | §5, §13 (worked example) |
| PR-LESSON-11 | Chord lessons cover major triads only, root position only. | [MVP] | §6, §7 |
| PR-LESSON-12 | Chord instruction builds gradually from single notes to simple triads, rather than introducing full triads immediately. | [MVP] | §5 |
| PR-LESSON-13 | The full set of major/minor scales beyond the MVP 4–5 keys is added. | [Future — Phase 2] | §6, §8 |
| PR-LESSON-14 | Minor, 7th, diminished, and augmented chord lesson *content*, plus inversions, are added as taught material. (Engine-level minor-triad *detection* is already MVP — see PR-ENGINE-04.) | [Future — Phase 2] | §6, §8 |
| PR-LESSON-15 | A song-learning module teaches a real, public-domain (or original) song section by section. | [Future — Phase 2] | §5, §6, §8, §30 (licensing risk gates this) |
| PR-LESSON-16 | Two-hand advanced-technique lessons are added. | [Future — Phase 4] | §7 (deferred table) |
| PR-LESSON-17 | Sight-reading / sheet-music instruction is added as a lesson type. | [Future — Phase 4] | §8 |

### 5.5 PR-DRILL — Practice Hub, Note ID Game, Free Play

| ID | Requirement | Tag | Source |
|---|---|---|---|
| PR-DRILL-01 | A Note Identification game prompts the learner to find a named key (e.g., "Which key is C?") and shows a running accuracy score. | [MVP] | §5, §6, §10 |
| PR-DRILL-02 | Note ID game difficulty increases as the learner improves, and every difficulty step-up/step-down is visibly indicated to the learner, never silent. | [MVP] | §5, §6, §10 |
| PR-DRILL-03 | A Practice Hub screen provides entry points to the Note ID game, a scale drill, a chord drill, and free play — with no lesson scaffolding (this is a drill surface, not a teaching surface). | [MVP] | §10 |
| PR-DRILL-04 | Piano Free Play exposes the full piano with live note/chord recognition surfaced as a toast (e.g., "You played: C major") and zero scaffolding. | [MVP] | §6, §10 |
| PR-DRILL-05 | Piano Free Play is the landing surface for the "try before signup" flow from the Landing screen. | [MVP] | §10 |

### 5.6 PR-AI — AI Tutor

See §4 above — this group also carries the non-negotiable architecture rule (PR-AI-08).

| ID | Requirement | Tag | Source |
|---|---|---|---|
| PR-AI-01 | The tutor answers free-form questions grounded in the learner's actual recent performance, assembled server-side (never client-supplied) into a `TutorRequest`. | [MVP] | §5, §6, §15, `TYPES_AND_CONTRACTS.md` §4.9 |
| PR-AI-02 | After a scored exercise, the tutor can produce structured feedback (e.g., "you missed F between E and G") generated from the `ExerciseVerdict` the engine already computed. | [MVP] | §6, §13, §15 |
| PR-AI-03 | For a returning learner, the tutor already knows their known weak areas (`AIMemorySummary`) and does not require them to re-explain. | [MVP] | §5, §17, `TYPES_AND_CONTRACTS.md` §4.8 |
| PR-AI-04 | A daily practice session is generated deterministically — next incomplete lesson + review of the last 3 misses — explicitly *not* Gemini-generated in MVP. | [MVP] | §5, §6, §7 (pushback) |
| PR-AI-05 | Lightweight practice-plan chat and motivational nudges are available; the top few common cases (e.g., "7-day streak") are served from a small pre-written pool rather than a live Gemini call. | [MVP] | §6, §15 |
| PR-AI-06 | Gemini is called only on explicit learner action (opening the tutor, tapping "ask why," requesting today's plan) or at the end of a scored exercise — never polled, never per keystroke, never speculative. | [MVP] | §14 |
| PR-AI-07 | The client never calls Gemini directly; every call routes through a server-side Cloud Function. | [MVP] | §14, §16, §19 |
| PR-AI-08 | **Gemini never determines correctness, computes any score, or determines completion/XP/streak; it explains a verdict the engine already produced.** Restates the rule in §4 for AI-domain traceability. | [MVP] | §14 |
| PR-AI-09 | Per-user rate limiting (a daily cap for free-tier users) is checked and atomically incremented server-side on every tutor call. | [MVP] | §19, §29 |
| PR-AI-10 | Learner free-text input to the tutor is length-capped and validated before being placed in a prompt; the Gemini call is scoped to plain-text output only, with no tool- or function-calling access to anything sensitive. | [MVP] | §19 |
| PR-AI-11 | Every Gemini response is validated against its template's output contract (length cap; JSON schema for structured templates); on validation failure, a deterministic canned message is shown — never a broken screen. | [MVP] | §15 |
| PR-AI-12 | A tutor response never states a note/chord/theory fact that contradicts (or is absent from) the structured data it was given. | [MVP] | §15, §29 |
| PR-AI-13 | Theory-explanation responses are cached per `(concept, skillLevel)` pair; a live Gemini call happens only on a cache miss. | [MVP] | §15, §31 |
| PR-AI-14 | If Gemini is unreachable, the AI Tutor screen shows a clear "temporarily unavailable, here's what to practice instead" fallback card built entirely from deterministic data — never a dead end. | [MVP] | §10, §29 |
| PR-AI-15 | Model tiering: the cheapest/fastest tier serves structured feedback and motivational copy; a mid tier is reserved for open-ended tutor conversation. | [MVP] | §14 |
| PR-AI-16 | Disabling the Gemini Cloud Function entirely leaves every other screen fully functional (tested explicitly, not assumed). | [MVP] | §29 |
| PR-AI-17 | Daily practice plans are generated by Gemini instead of the deterministic rule, once there is enough real usage data to personalize against. | [Future — Phase 2] | §6, §7 (pushback), §8, §15 |

### 5.7 PR-PROGRESS — Progress, Gamification, Dashboard

| ID | Requirement | Tag | Source |
|---|---|---|---|
| PR-PROGRESS-01 | XP is tracked per user and increases as exercises/lessons are completed. **Note:** no XP-earning formula (how much XP per action) is defined anywhere in `PRODUCT_SPEC.md` — flagged as an open question, not silently invented here. | [MVP] | §6, §5, §17 |
| PR-PROGRESS-02 | A streak (count + last-practice-date, calendar-day based) is tracked per user. | [MVP] | §6, §5, `TYPES_AND_CONTRACTS.md` §4.15 |
| PR-PROGRESS-03 | Lesson completion is tracked per lesson (`status`, `bestScore`, `attempts`, `lastAttemptAt`). | [MVP] | §6, `TYPES_AND_CONTRACTS.md` §4.7 |
| PR-PROGRESS-04 | Weak notes and weak chords are tracked and surfaced on the dashboard as a callout. | [MVP] | §6, §5 |
| PR-PROGRESS-05 | The Dashboard shows: greeting, streak, XP, a "Continue lesson" CTA, today's practice card, one weak-area callout, and quick links to Piano and Tutor. | [MVP] | §10 |
| PR-PROGRESS-06 | On a learner's first-ever visit, the Dashboard shows a "Start your first lesson" empty state instead of streak/stats. | [MVP] | §10 |
| PR-PROGRESS-07 | A Progress/Stats view (reached from Profile, not a top-level nav item) shows accuracy over time, weak notes/chords, lessons completed, and practice minutes. | [MVP] | §10 |
| PR-PROGRESS-08 | A learner resumes exactly where they left off between sessions, with no lost state. | [MVP] | §5, §6 |
| PR-PROGRESS-09 | Achievements/badges beyond streak and XP are added, kept deliberately restrained. | [Future — Phase 2] | §6, §8 |
| PR-PROGRESS-10 | Leaderboards / multiplayer / social challenges are added. | [Future — Phase 4, low priority, speculative — "no evidence yet that beginners want competitive piano"] | §6, §8 |
| PR-PROGRESS-11 | Shareable progress reports are added. | [Future — Phase 3] | §7 (deferred table), §8 |

### 5.8 PR-A11Y — Accessibility

Full detail and rationale in `docs/UX/ACCESSIBILITY.md`; this table exists for traceability IDs only.

| ID | Requirement | Tag | Source |
|---|---|---|---|
| PR-A11Y-01 | Every non-piano control (nav, lesson list, buttons, tutor chat) is fully operable via Tab/Enter/Space with visible focus indicators. | [MVP] | §24, §29 |
| PR-A11Y-02 | Lesson instructions, feedback, and score results are announced via ARIA live regions. | [MVP] | §24 |
| PR-A11Y-03 | Piano keys carry accessible names (e.g., "C4, white key"). | [MVP] | §24 |
| PR-A11Y-04 | Text meets WCAG AA contrast (4.5:1) minimum throughout. | [MVP] | §24 |
| PR-A11Y-05 | Correct/incorrect key feedback always pairs with a non-color signal (icon, shape, or animation) — never color alone. | [MVP] | §5, §24 |
| PR-A11Y-06 | Layout respects browser/OS text-size settings and remains usable at 200% browser zoom. | [MVP] | §24 |
| PR-A11Y-07 | Key-press animations and transitions respect `prefers-reduced-motion`. | [MVP] | §24 |
| PR-A11Y-08 | Full accessible parity on the piano-*playing* interaction itself (a screen-reader user sight-reading a rapidly changing visual keyboard) is an explicit non-goal — not achievable in MVP, not attempted. Only the surrounding UI is fully accessible, plus full keyboard-only *play* (already required by PR-PIANO-06). | [Explicit MVP limitation — not phase-scheduled, see §24's own caveat] | §24 |

### 5.9 PR-PLATFORM — Mobile-first, Resilience, Performance

| ID | Requirement | Tag | Source |
|---|---|---|---|
| PR-PLATFORM-01 | The UI is mobile-first and responsive; one-handed portrait use is the baseline, landscape/desktop are progressive enhancement, not the reverse. | [MVP] | §7, §9 |
| PR-PLATFORM-02 | The piano is the one fixed, always-visible element on relevant screens; everything else scrolls or collapses around it. | [MVP] | §9 |
| PR-PLATFORM-03 | Piano interaction (sound, input, the current lesson's already-loaded content) works with zero network dependency once assets are loaded, because the Engine Core has zero network dependency by design. | [MVP] | §2, §11, §22, §23 |
| PR-PLATFORM-04 | Firestore's built-in offline persistence is enabled; writes made while offline queue and flush automatically on reconnect. | [MVP] | §23 |
| PR-PLATFORM-05 | A persistent, non-alarming degraded-service banner appears whenever Firestore or Gemini is unreachable (e.g., "Playing offline — your progress will sync later"); the piano and current lesson keep working locally regardless. | [MVP] | §10, §23 |
| PR-PLATFORM-06 | Firestore writes are batched at exercise completion, never per note. | [MVP] | §11, §22 |
| PR-PLATFORM-07 | The dashboard reads one flattened progress document, not N per-lesson documents. | [MVP] | §17, §22 |
| PR-PLATFORM-08 | The virtual piano is memoized per-key/per-octave so pressing one key does not re-render the whole keyboard. | [MVP] | §22 |
| PR-PLATFORM-09 | The AI Tutor chat UI and any Phase-2-only modules are code-split from the core piano/lesson bundle. | [MVP] | §22 |
| PR-PLATFORM-10 | Full offline-first PWA behavior with service-worker asset caching for lesson content is added. | [Future — Phase 2] | §8, §23 |

### 5.10 PR-CONTENT — Content Operations

| ID | Requirement | Tag | Source |
|---|---|---|---|
| PR-CONTENT-01 | An admin can edit lesson content without a code deploy (Admin CMS). | [Future — Phase 3] | §5, §8 |

## 6. Summary Counts

| Area | MVP requirements | Future requirements |
|---|---|---|
| PR-AUTH | 9 | 1 |
| PR-PIANO | 16 | 2 |
| PR-ENGINE | 16 | 1 |
| PR-LESSON | 12 | 5 |
| PR-DRILL | 5 | 0 |
| PR-AI | 16 | 1 |
| PR-PROGRESS | 8 | 3 |
| PR-A11Y | 7 | 0 (+1 explicit non-goal) |
| PR-PLATFORM | 9 | 1 |
| PR-CONTENT | 0 | 1 |

## 7. Open Questions Inherited or Raised Here

These are raised, not silently resolved — full list with severity in the batch summary delivered alongside this document:

- No ratified default `passThreshold` value for `Lesson` (blocks concrete lesson-content authoring for PR-LESSON-06).
- No XP-earning formula defined anywhere in `PRODUCT_SPEC.md` (affects PR-PROGRESS-01).
- Engine-level minor-triad detection is required in MVP (§29, `TYPES_AND_CONTRACTS.md` §4.12) while minor-triad *lesson content* is Phase 2 (§6/§7) — resolved here by splitting PR-ENGINE-04 (capability, MVP) from PR-LESSON-14 (content, Future), but the source spec does not state this split explicitly.

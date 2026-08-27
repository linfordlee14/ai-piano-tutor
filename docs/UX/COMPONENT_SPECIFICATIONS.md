# Component Specifications

## 1. Purpose

Identifies the reusable UI components implied by `docs/UX/SCREEN_SPECIFICATIONS.md`, at interface level: purpose, key props/state, and which screens use each. This is **not** an implementation document — no component code, no JSX, no styling detail. Props are described at the contract level (name, type, source) so an implementer knows what data a component needs and where it comes from, referencing `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` types wherever a prop is (or wraps) a canonical shared type.

## 2. Scope

The eleven components implied across the MVP screens: `PianoKeyboard`, `KeyComponent`, `LessonProgressIndicator`, `StreakBadge`, `XPBar`, `ScoreCard`, `ToastNotification`, `DegradedServiceBanner`, `TutorChatBubble`, `HintButton`, `DashboardCard`. Out of scope: implementation code, exact styling/tokens, and components that are purely generic UI kit (basic buttons, inputs) with nothing piano-app-specific about their contract.

## 3. PianoKeyboard

### Purpose
Renders the full playable keyboard across the current visible octave range; owns panning/scrolling and aggregates individual `KeyComponent` instances into one coherent instrument. This is the one component present on nearly every screen with any piano-playing surface.

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `visibleOctaveRange` | `{ startMidi: number; endMidi: number }` | Drives which `KeyComponent`s render; adjustable per §6/PR-PIANO-08 |
| `keyboardMapping` | `KeyboardMapping` (`TYPES_AND_CONTRACTS.md` §4.11) | Computer-keyboard-to-MIDI-offset mapping; sourced from `UserProfile.settings.keyboardMapping` |
| `onNoteEvent` | `(event: NoteEvent) => void` | Fired on every note-on/note-off from any input adapter (mouse/touch/keyboard) this instance handles; consumers (Audio Layer, Engine Core via a parent hook, Zustand store) subscribe here — see `TYPES_AND_CONTRACTS.md` §4.1 |
| `highlightedKeys` | `{ midiNote: number; state: 'demo' \| 'correct' \| 'incorrect' }[]` | Drives per-key visual state during a demonstration or after a verdict; derived from Engine Core output, never computed by this component itself |
| `labelsVisible` | `boolean` | Toggleable note labels, PR-PIANO-02 |
| `sustainEnabled` | `boolean` | From `UserProfile.settings.sustainDefault` or a session-level override |
| `volume` | `number` (0–1) | From `UserProfile.settings.volume` |
| `touchTargetMinPx` | `number` | Enforces the ≥44px-on-phone rule (PR-PIANO-11); not something screens should be able to override below the accessibility floor |

### State it owns internally
Current pan/scroll offset; which samples are loaded vs. still loading (surfaces a loading state upward rather than rendering silently broken keys, per §22).

### Screens that use it
Piano Free Play, Lesson Player, Note Identification Game, Practice Hub (scale/chord drill views).

---

## 4. KeyComponent

### Purpose
A single playable key (white or black), the atomic rendering unit inside `PianoKeyboard`. Owns nothing about music theory — per §11, "UI Layer render[s] key state... from Engine Core output; owns nothing about music theory itself."

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `midiNote` | `number` (0–127) | Canonical pitch identity for this key — `TYPES_AND_CONTRACTS.md` §3 |
| `isBlackKey` | `boolean` | Derived from `midiNote mod 12`, but passed/cached rather than recomputed per render |
| `label` | `string \| null` | Display note name (e.g., "C4"), null when labels are toggled off |
| `visualState` | `'idle' \| 'hover' \| 'active' \| 'correct' \| 'incorrect'` | Exactly the five states named in §11 |
| `accessibleName` | `string` | e.g., "C4, white key" — required per §24, not optional polish |
| `onPress` / `onRelease` | `() => void` | Bubbles up to `PianoKeyboard`'s `onNoteEvent`, does not construct a `NoteEvent` itself (timestamp/velocity/source are adapter-level concerns, not per-key concerns) |

### Screens that use it
Never used standalone outside `PianoKeyboard` — always one of many instances inside it.

---

## 5. LessonProgressIndicator

### Purpose
Shows position within the *current lesson* (not overall course progress — that's the Learn/Lesson Path screen's job).

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `currentStep` | `number` | Position within `Lesson.expectedSequence` or the lesson's defined step breakdown |
| `totalSteps` | `number` | Length of the same |
| `lessonType` | `Lesson['type']` (`TYPES_AND_CONTRACTS.md` §4.2) | May affect display (e.g., a scale lesson's progress reads differently from a melody lesson's) |
| `accessibleLabel` | `string` | e.g., "Step 2 of 5" as real text, not implied only by a visual bar's fill percentage (§10/`SCREEN_SPECIFICATIONS.md` §8) |

### Screens that use it
Lesson Player.

---

## 6. StreakBadge

### Purpose
Displays the learner's current streak with a non-color-only signal (icon + number, not color alone — §24).

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `streakCount` | `number` | `UserProfile.streak.count` (`TYPES_AND_CONTRACTS.md` §4.15) |
| `lastPracticeDate` | `string` (`YYYY-MM-DD`) | `UserProfile.streak.lastPracticeDate`; used to determine "streak at risk today" styling if desired, not required |
| `size` | `'compact' \| 'full'` | Dashboard may want a compact badge; Progress/Stats may want a fuller one with more context |

### Screens that use it
Dashboard (Home), Progress/Stats.

---

## 7. XPBar

### Purpose
Displays accumulated XP. **Flagged, not silently assumed:** `PRODUCT_SPEC.md` defines `UserProfile.xp` as a running integer total but never defines an XP-earning formula (how much XP a given action awards) or a leveling curve/threshold — `UserProfile.currentLevel` is explicitly tied to furthest-unlocked `Lesson.level`, not to an XP threshold (`TYPES_AND_CONTRACTS.md` §4.15). This component is therefore specified as a **running-total display**, not a leveled progress bar with defined thresholds, until that formula is defined elsewhere. See the open-questions summary for this batch.

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `xp` | `number` | `UserProfile.xp` |
| `recentDelta` | `number \| null` | Optional — if the screen wants to animate "+X XP" immediately after a scored exercise, sourced from the just-received verdict's session, not recomputed client-side |

### Screens that use it
Dashboard (Home); also reasonable on Lesson Player's `lessonComplete` state to show the XP just earned (an elaboration consistent with the core loop's Progress stage, not a new requirement).

---

## 8. ScoreCard

### Purpose
Renders a completed exercise's verdict — the primary place `ExerciseVerdict` becomes visible to the learner. This is the component most directly responsible for making the non-negotiable engine/AI boundary *visible*: it renders engine output only; any Gemini-authored prose is a clearly separate element layered on top or opened via a distinct action, never blended into the score fields themselves.

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `verdict` | `ExerciseVerdict` (`TYPES_AND_CONTRACTS.md` §4.5) | Full verdict object — `correctNotes`, `missedNotes`, `extraNotes`, `accuracy`, `timingScore`, `completeness`, `consistency`, `perNoteTiming`, `compositeScore` (`Score`), `passed` |
| `isProvisional` | `boolean` | True for a client-side optimistic preview rendered before the server's authoritative verdict returns (`TYPES_AND_CONTRACTS.md` §4.5's client/server boundary note) — must visually communicate "pending confirmation" when true |
| `onAskTutor` | `() => void` | Wires the "ask the tutor why" entry point (Lesson Player) |
| `onRetry` | `() => void` | Wires the Repeat stage of the core loop |
| `attemptNumber` | `number` | For display context (e.g., "Attempt 3") — sourced from `ExerciseAttempt.attemptNumber`-equivalent server-confirmed count, not the client's own guess |

### Screens that use it
Lesson Player (`correct`/`incorrectWithHint`/`lessonComplete` states), Note Identification Game (per-round or session summary), Practice Hub drills (scale/chord drill results).

---

## 9. ToastNotification (Note/Chord Recognition)

### Purpose
An ephemeral, non-blocking surface for live recognition results during unscaffolded play — "You played: C major."

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `message` | `string` | Rendered display text, e.g., "You played: C major" |
| `recognizedName` | `string \| null` | The note or chord name the engine recognized (derived display value per `TYPES_AND_CONTRACTS.md` §3 — never the raw MIDI number shown to the learner directly) |
| `autoDismissMs` | `number` | Toast disappears on its own; does not require learner action, since requiring dismissal would interrupt the exploratory flow this component exists to support |
| `politeness` | `'polite'` (fixed) | Not learner-configurable — informative, not urgent; see `docs/UX/SCREEN_SPECIFICATIONS.md` §11's accessibility note on why this must not be assertive |

### Screens that use it
Piano Free Play (primary use case), Practice Hub's free-play entry.

---

## 10. DegradedServiceBanner

### Purpose
The persistent, non-alarming, app-wide notice for Firestore/Gemini unreachability — one canonical implementation of the pattern defined in `docs/UX/SCREEN_SPECIFICATIONS.md` §15.

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `serviceType` | `'firestore' \| 'gemini'` | Drives message copy; see the flagged, non-blocking ambiguity in `docs/UX/SCREEN_SPECIFICATIONS.md` §15 about exactly when each fires |
| `message` | `string` | e.g., "Playing offline — your progress will sync later" |
| `visible` | `boolean` | Driven by connectivity/Function-health state, not learner-dismissible per §10's "persistent" framing — this component intentionally has no `onDismiss` prop |

### Screens that use it
Rendered at the app-shell level, potentially visible above any screen — not scoped to one specific screen's component tree.

---

## 11. TutorChatBubble

### Purpose
A single message in the AI Tutor chat, for either the learner or the tutor.

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `role` | `'learner' \| 'tutor'` | Determines alignment/styling |
| `text` | `string` | `TutorResponse.text` for tutor messages (`TYPES_AND_CONTRACTS.md` §4.10), or the learner's own submitted question |
| `suggestedAction` | `string \| null` | `TutorResponse.suggestedAction` — rendered as a deep-link affordance (e.g., a button), not raw text, since the contract describes it as "a deep-link hint, not free text" |
| `isFallback` | `boolean` | True when this bubble is a deterministic canned message substituted for a failed/invalid Gemini response (§15's guardrail) — should read differently from a normal tutor answer only if that distinction is useful to the learner; not required to look alarming |

**Explicitly excluded from props:** `TutorResponse.tier` and `TutorResponse.cached` are never passed into anything this component renders as visible text or accessible name — both are marked "not shown to the user" / "for cost monitoring" in `TYPES_AND_CONTRACTS.md` §4.10, and this component's contract should make that impossible to violate by simply not accepting those fields at all.

### Screens that use it
AI Tutor.

---

## 12. HintButton

### Purpose
Triggers the next escalating hint level during a Lesson Player (or drill) attempt — the concrete UI trigger for the core loop's Repeat stage.

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `currentHintLevel` | `number` | How many hints already used for this attempt |
| `maxHintLevel` | `number \| null` | If a lesson defines a cap; `null` if hints can escalate indefinitely without ever hard-stopping the learner (consistent with §5/§9's "never a hard stop" principle — this component must never render a state that blocks further attempts entirely) |
| `onRequestHint` | `() => void` | Requests the next hint level |
| `disabled` | `boolean` | Only true while a hint is actively being fetched/computed, never true as a permanent "no more help" dead end |

### Screens that use it
Lesson Player; potentially Practice Hub drills if scale/chord drills also offer escalating hints (reasonable extension, not explicitly ruled in or out by §10).

---

## 13. DashboardCard

### Purpose
A generic reusable card shell used for the Dashboard's three distinct content blocks: "Continue lesson" CTA, today's practice card, and the weak-area callout. One component, three instances with different content — not three bespoke components, since all three share the same "title + body + primary action" shape.

### Key props / state

| Prop | Type | Notes |
|---|---|---|
| `title` | `string` | e.g., "Continue lesson," "Today's practice," "Work on this" |
| `body` | `string \| ReactNode` | Supporting detail (lesson name, plan summary, weak-note/chord name) |
| `ctaLabel` | `string` | e.g., "Continue," "Start," "Practice now" |
| `onCtaPress` | `() => void` | Destination varies per instance — Lesson Player, a specific drill, or Tutor with context, per `docs/UX/SCREEN_SPECIFICATIONS.md` §6 |
| `variant` | `'continue' \| 'practicePlan' \| 'weakArea'` | Purely presentational distinction; underlying shape is identical |

### Screens that use it
Dashboard (Home) — three instances.

## 14. Components Implied But Not Detailed Here

Mentioned for completeness; not elaborated since they weren't named in the task's component list and are lower-stakes/more generic:

- **QuickSuggestionChip** — the tappable suggestion chips on the AI Tutor screen (e.g., "What should I practice today?"). Simple label + `onPress`.
- **AttributionCredit** — the Salamander Grand Piano CC BY 3.0 credit line on Profile/Settings (PR-PIANO-16). Static text + link, no meaningful state.
- **LockedLessonIndicator** — the non-color "locked" signal on Learn (Lesson Path) list items (icon + label). Could be folded into a lesson-list-item component rather than standing alone.

## 15. Cross-References

- `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — every referenced type (`NoteEvent`, `ExerciseVerdict`, `Score`, `KeyboardMapping`, `TutorResponse`, `UserProfile`, etc.).
- `docs/UX/SCREEN_SPECIFICATIONS.md` — where each component is used, in full screen context.
- `docs/UX/ACCESSIBILITY.md` — accessibility requirements that shape several props above (`accessibleName`, `accessibleLabel`, ARIA-politeness choices).

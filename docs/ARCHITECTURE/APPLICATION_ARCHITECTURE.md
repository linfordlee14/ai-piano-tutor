# Application Architecture

**Status:** Draft v1.0 — architecture documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §9, §11, §21, §22, §27
**Companion documents:** `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md`, `docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md`, `docs/ENGINEERING/PROJECT_STRUCTURE.md` (folder-level detail, written in parallel — this document describes *why* the structure is shaped this way; `PROJECT_STRUCTURE.md` is the authority on exact paths), `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`

## 1. Purpose

Describe the client-side application's structure: how the five-layer module breakdown from `docs/PRODUCT_SPEC.md` §11 maps onto a real Next.js application, how Next.js App Router, Zustand, and Firestore's `onSnapshot` divide up state ownership, and where the boundaries between them are drawn and enforced.

**The non-negotiable rule this structure exists to protect:** Gemini must never determine whether a note or chord is correct, calculate accuracy, calculate timing, calculate scores, determine lesson completion, determine XP, determine streaks, or validate user-submitted performance data. On the client specifically, this means no component, hook, or store in `apps/web` may treat a Gemini/`TutorResponse` value as the source of truth for correctness, scoring, or progress — those always come from the Engine Core (client-side, provisional) or a Cloud Function response (server-side, authoritative), never from AI Tutor UI output. The AI Tutor is a consumer of already-computed results, not a producer of them. Full boundary and reasoning: `AI_ARCHITECTURE.md` §3.

## 2. Scope

In scope: `apps/web` — routing, component organization, state management, and how the client consumes the Engine Core, Audio Layer, and Sync Layer described in `MUSIC_ENGINE_ARCHITECTURE.md`.

Out of scope: the internals of `packages/music-engine` itself (module boundaries and dependency rules → `MUSIC_ENGINE_ARCHITECTURE.md`; algorithms → `docs/MUSIC/MUSIC_ENGINE_SPEC.md`), server-side (`functions/`) internals (→ `AI_ARCHITECTURE.md`, `SECURITY_ARCHITECTURE.md`), exact folder paths and file naming (→ `docs/ENGINEERING/PROJECT_STRUCTURE.md`).

## 3. Architecture

### 3.1 The five layers, at application scope

`docs/PRODUCT_SPEC.md` §11 defines five module boundaries. This document places each one inside the running Next.js application; `MUSIC_ENGINE_ARCHITECTURE.md` is the authority on the dependency rules *between* them.

| Layer | Lives in | What it owns in the running app |
|---|---|---|
| **Input Layer** | `apps/web` (adapters) | Mouse/touch/computer-keyboard event listeners attached to the piano UI component; normalizes raw DOM/keyboard events into `NoteEvent` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1). MIDI adapter is designed-for-but-not-built (Phase 2) — see `MUSIC_ENGINE_ARCHITECTURE.md` §6. |
| **Engine Core** | `packages/music-engine`, imported into `apps/web` | A plain module, called imperatively from the Input Layer / lesson-player logic — never a React component, never Zustand state itself. Its *output* (a client-side, provisional `ExerciseVerdict`) flows into Zustand. See §4 below and `MUSIC_ENGINE_ARCHITECTURE.md`. |
| **Audio Layer** | `apps/web` (hooks/services wrapping Tone.js) | Triggered directly by the Input Layer, in parallel with the Engine Core — not downstream of it. See `AUDIO_ARCHITECTURE.md`. |
| **UI Layer** | `apps/web/components` | React components rendering key state (idle/hover/active/correct/incorrect) from Engine Core output and Zustand; owns no music theory logic itself. |
| **Sync Layer** | `apps/web` (a hook/service, not a component) | Buffers session results and calls `getExerciseFeedback` on exercise completion — not per note. |

### 3.2 Next.js App Router structure

Routes mirror the five-tab navigation from `docs/PRODUCT_SPEC.md` §9 (deliberately flatter than the original brief's structure — Practice and Progress live inside Home/Learn rather than as separate top-level tabs):

- `(marketing)/` — pre-auth landing page (`docs/PRODUCT_SPEC.md` §10: value prop + "Try the piano now," zero-friction anonymous session start)
- `(app)/dashboard` — Home tab
- `(app)/learn` — Learn tab (lesson path + Lesson Player)
- `(app)/piano` — Piano tab (free play; also the screen used for the pre-auth try-first flow)
- `(app)/tutor` — Tutor tab (AI Tutor chat)
- `(app)/profile` — Profile tab (settings, and Progress/Stats, which lives here rather than as its own top-level tab)

Exact route file layout, layout nesting, and naming conventions: `docs/ENGINEERING/PROJECT_STRUCTURE.md`.

**Screen states owned per route** (from `docs/PRODUCT_SPEC.md` §10, restated here because they drive component structure, not content): the Lesson Player is a state machine — demonstrating / awaiting input / correct / incorrect-with-hint / lesson-complete — not five separate routes; the Dashboard has an explicit empty state (first-ever visit) distinct from its steady state; every screen carries the degraded-service banner as a cross-cutting shell element, not a per-screen concern.

### 3.3 Core interaction loop, mapped to layers

`docs/PRODUCT_SPEC.md` §9's loop — **Learn → Demonstrate → Practice → Feedback → Repeat → Progress** — exercises the five layers in this order per cycle:

1. **Learn / Demonstrate:** UI Layer renders lesson instructions; Audio Layer plays a pre-scripted demonstration sequence (no Input Layer or Engine Core involvement — this is playback, not scoring).
2. **Practice:** learner input → Input Layer emits `NoteEvent` → fan-out to Audio Layer (sound) and Engine Core (local, provisional validation) → UI Layer updates key visuals from Engine Core's output.
3. **Feedback:** on exercise end, Sync Layer sends the buffered `NoteEvent[]` (as an `ExerciseAttempt`) to `getExerciseFeedback`; the response (an authoritative `ExerciseVerdict`, plus short Gemini feedback prose) replaces the provisional client-side verdict in the UI. Full trace: `DATA_FLOW.md`.
4. **Repeat:** escalating hints on a failed attempt, never a hard stop — a UI Layer / lesson-content concern, not an engine one.
5. **Progress:** dashboard reflects the server-written `ProgressSummary`/`UserProfile.xp`/`UserProfile.streak`, read via `onSnapshot` (§4.2 below) — not computed client-side.

## 4. State management

Per `docs/PRODUCT_SPEC.md` §21, state is split by *ownership*, not by convenience — three distinct mechanisms, none layered redundantly on top of another:

### 4.1 Client-only UI/engine state — Zustand

A small Zustand store holds: current lesson step, the live note buffer (recent `NoteEvent`s for the active exercise), audio-loading state, and tutor-chat draft text. Zustand is chosen specifically because it's readable and writable from *outside* React's render cycle — the Audio Layer needs to read and update state from Tone.js event callbacks, which fire independently of React's scheduler. This keeps the audio-latency budget (`AUDIO_ARCHITECTURE.md`) safe from render-cycle jitter.

**Explicit rule, restated from `docs/PRODUCT_SPEC.md` §21 because it is easy to violate by accident:** the Engine Core does **not** live in React state and is **not** a Zustand slice that "is" the engine. It is a plain module with its own internal state, called imperatively (e.g., `engine.evaluate(noteEvents, activeLesson)`). Only its *output* — a verdict, a set of key-highlight states — flows into Zustand for rendering. If a future contributor is tempted to model the engine itself as a Zustand store, that's a violation of this rule and of the dependency boundary in `MUSIC_ENGINE_ARCHITECTURE.md` §4 (the engine must stay framework-independent; a Zustand-native engine is no longer framework-independent).

### 4.2 Server state — Firestore `onSnapshot`, no extra caching layer

User profile, progress, and lesson content are read via lightweight custom hooks wrapping Firestore's `onSnapshot`, relying on the Firestore SDK's own cache and realtime sync. `docs/PRODUCT_SPEC.md` §21 explicitly rejects layering React Query or SWR on top — that would duplicate caching Firestore already provides, for no benefit. Hooks like `useUserProfile()` or `useLessonPath()` (exact names: `docs/ENGINEERING/PROJECT_STRUCTURE.md`) subscribe once and re-render on server-side changes, including the writes `getExerciseFeedback` makes after scoring.

### 4.3 Ephemeral / non-persisted state

`TutorResponse` objects are rendered into the chat UI but not persisted as a growing client-side log beyond the current session's component state — per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.10, nothing about tutor conversation content is retained beyond the `AIMemorySummary` aggregate, which is server-written, not client-cached.

### 4.4 Why this three-way split, not one state library

A single global store (Redux-style) would force either the Engine Core to become framework-aware (violating its independence) or server data to be manually kept in sync with Firestore's own cache (duplicating work Firestore already does). The three-way split — Zustand for client/engine state, `onSnapshot` hooks for server state, and the Engine Core's own internal state kept entirely outside both — is the direct consequence of `docs/PRODUCT_SPEC.md` §21's instruction to "choose [a state library] only if needed," applied literally rather than defaulted away.

## 5. Data structures

The client consumes and produces the following shared types (full definitions: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`):

- **Produces:** `NoteEvent` (Input Layer), `ExerciseAttempt` (Sync Layer, wrapping buffered `NoteEvent[]`), `{ learnerQuestion }` (Tutor UI).
- **Consumes (read via `onSnapshot`):** `UserProfile` (including embedded `ProgressSummary` and `settings.keyboardMapping: KeyboardMapping`), `Lesson[]` (public-read collection).
- **Consumes (as Cloud Function responses):** `ExerciseVerdict`, `TutorResponse`.
- **Computes locally, provisionally:** a client-side `ExerciseVerdict` via the Engine Core, for instant feedback — explicitly overwritten by the server's response (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5; full sequencing in `DATA_FLOW.md`).

The client never constructs a `PracticeSession`, `ProgressSummary` write, or `AIMemorySummary` — those are Cloud-Function-only writes (`SECURITY_ARCHITECTURE.md`).

## 6. Interfaces / contracts

- **Hooks layer** (`apps/web/hooks`, per `docs/PRODUCT_SPEC.md` §27): Firestore subscription hooks (server state, §4.2) and audio-engine hooks (bridging Tone.js callbacks into Zustand, §4.1).
- **Stores layer** (`apps/web/stores`): Zustand stores, one per concern (engine/UI state) rather than one monolithic store — keeps the "what triggers a re-render" surface small, which matters for the per-key memoization strategy in §7.
- **Callable-function client wrappers:** thin typed wrappers around Firebase's callable-function SDK, one per function listed in `SYSTEM_ARCHITECTURE.md` §6, each returning the `{ ok, data | error }` shape untouched so every call site handles errors the same way.

## 7. Dependencies

- `packages/music-engine` — imported directly, never re-implemented in `apps/web`. See `MUSIC_ENGINE_ARCHITECTURE.md` for why this must remain a workspace package rather than a folder inside `apps/web`.
- `packages/shared-types` — the types from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`, imported by both client and Functions so the two never drift.
- Next.js App Router, Zustand, Firebase client SDK (Auth, Firestore, Functions callable client), Tone.js (via the Audio Layer). Tailwind CSS for styling (`docs/PRODUCT_SPEC.md` §32).
- Exact versions and additional tooling: `docs/ENGINEERING/PROJECT_STRUCTURE.md`.

## 8. Performance considerations

From `docs/PRODUCT_SPEC.md` §22, at the application-structure level:

- **Rendering:** the virtual piano is memoized per-key/per-octave so pressing one key doesn't re-render the whole keyboard — a common, avoidable perf bug in naive piano UIs. This is why engine/audio state changes are scoped narrowly in Zustand (§4.1) rather than causing broad store updates.
- **Bundle size:** the AI Tutor chat UI and any Phase-2-only modules (a future MIDI adapter, a future Daily Practice Plan UI) are code-split so the core piano/lesson bundle stays lean on mobile networks. This is a routing/bundling concern enforced at the App Router level (route-based code splitting is Next.js's default; Phase-2 modules should not be imported from a route that ships in MVP).
- **Firestore reads:** the dashboard reads one flattened `UserProfile` document (with embedded `ProgressSummary`) via a single `onSnapshot`, not N per-lesson documents — a direct consequence of the data-model decision in `docs/PRODUCT_SPEC.md` §17, surfaced here because it's *this* layer (the hooks in §6) that would otherwise be tempted to fan out into N subscriptions.

## 9. Error handling

- The degraded-service banner (`docs/PRODUCT_SPEC.md` §10) is a shell-level concern — rendered once in the app's root layout, driven by connectivity/Firestore-listener state, not duplicated per screen.
- Every screen that depends on a Cloud Function response (Lesson Player's Feedback state, the AI Tutor screen) has an explicit fallback UI for the `{ ok: false, error }` shape (`SYSTEM_ARCHITECTURE.md` §6) — a canned deterministic message, never a raw error dump or a blank screen.
- Audio-loading failure (sample fetch fails) and the autoplay-gesture requirement are Audio Layer concerns with UI Layer surfaces — detailed in `AUDIO_ARCHITECTURE.md` §Error handling, not duplicated here.

## 10. Testing requirements

- **Playwright, UI/interaction:** key press → sound triggers → visual state updates, the computer-keyboard mapping, and the full anonymous-try-the-piano flow end to end (`docs/PRODUCT_SPEC.md` §26). This is where the Input → Audio/Engine → UI fan-out (§3.3) gets exercised as a whole, not layer-by-layer.
- **Accessibility:** automated axe-core checks in CI on all core screens, plus one manual screen-reader pass before launch (`docs/PRODUCT_SPEC.md` §24, §26).
- **Component-level:** standard React component tests for UI Layer components, asserting they render correctly from a given Engine Core output / Zustand state — not re-testing engine logic itself (that's `MUSIC_ENGINE_ARCHITECTURE.md`'s test surface).

## 11. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29, application-layer-relevant items:
- [ ] Mouse, touch, and computer-keyboard input all produce identical engine and audio behavior for the same note.
- [ ] Full keyboard-only navigation of every non-piano screen.
- [ ] Automated axe-core check passes on all core screens.
- [ ] An anonymous try-first session's progress survives linking to a real account (client-side: `linkWithCredential` flow, `docs/PRODUCT_SPEC.md` §18).

## 12. Out of scope

- Exact file/folder paths, naming conventions, workspace configuration → `docs/ENGINEERING/PROJECT_STRUCTURE.md`
- Engine Core dependency rules and the Input Adapter contract (including the Phase-2 MIDI adapter design) → `MUSIC_ENGINE_ARCHITECTURE.md`
- Tone.js implementation, sample loading strategy → `AUDIO_ARCHITECTURE.md`
- Cloud Function internals, Gemini call assembly → `AI_ARCHITECTURE.md`
- Firestore security rules and write-ownership enforcement → `SECURITY_ARCHITECTURE.md`
- Precise request/response sequencing for exercise completion and tutor Q&A → `DATA_FLOW.md`

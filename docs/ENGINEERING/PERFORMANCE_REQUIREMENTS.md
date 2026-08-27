# Performance Requirements

## 1. Purpose

The numeric, testable performance targets for AI Piano Tutor's MVP, extracted from `docs/PRODUCT_SPEC.md` §22 (Performance) and cross-checked against every section that establishes a specific number or rule (§12 audio, §14 AI call gating, §17/§21 data model, §26 testing strategy, §29 acceptance criteria).

**Division of labor:** this document owns *what the targets are and why they exist*. `docs/TESTING/PERFORMANCE_TESTING.md` owns *how each one gets measured* — the Lighthouse CI configuration, the manual device-testing protocol, and any RUM/telemetry setup. Where a target below has a specific verification method already named in PRODUCT_SPEC.md, it's stated here so the two documents stay aligned instead of each guessing at the other's content.

## 2. Scope

Covers: key-press-to-sound latency, Firestore write/read patterns, bundle code-splitting, Gemini call latency, and rendering/memoization — the five areas §22 names explicitly, plus the closely-related autoplay/sample-loading readiness requirement §12 treats as part of the same "does the piano feel responsive" concern.

Out of scope: acoustic pitch detection performance (§13's explicit non-goal, Phase 3), MIDI input latency (Phase 2, §8), full offline-first PWA asset-caching performance (Phase 2, §23), microphone-based input (Phase 3, §8). None of these are MVP-scoped, so none get a performance target here.

## 3. Requirements

Each requirement is numbered, stated with its target, and marked with how firm the number is — **Spec** (an explicit number from PRODUCT_SPEC.md, not up for reinterpretation) or **Recommended** (this document proposing a number where PRODUCT_SPEC.md gives a qualitative requirement but no figure, marked as such rather than presented as extracted fact).

### PERF-1 — Key-press-to-sound latency (Spec)

**Target: under ~50ms from key-press event to audible sound onset**, measured device-side — this is 100% local per the architecture (§22: "Piano interaction never waits on the network... sound and key-highlight feedback are 100% local"), so no network round-trip is ever part of this number.

- §12: "under ~50ms from key-press to audible sound is the general threshold for 'feels responsive.'"
- §29 (acceptance criterion, exact wording): "Key press (mouse, touch, or mapped computer key) produces sound in under ~50ms **on a mid-range Android device and a current iPhone**." The device pairing is deliberate, not incidental — §12 flags that "Android Chrome has historically had higher baseline audio latency than iOS Safari," so this target is not satisfied by testing on a flagship phone or a dev laptop alone. A real mid-range Android device is the binding case.
- §29 (adjacent, directly related): "Mouse, touch, and computer-keyboard input all produce identical engine and audio behavior for the same note" — the 50ms target holds uniformly across all three MVP input methods, not just whichever is fastest to implement.

**Verification:** manual key-press-to-sound latency check on a real mid-range Android device, specifically because — per §26 — "that specific number doesn't show up in Lighthouse." Full protocol: `docs/TESTING/PERFORMANCE_TESTING.md`.

### PERF-2 — Time-to-first-sound readiness (Spec, qualitative + Recommended, numeric)

The piano must never be silently broken. Two related requirements from §12/§22:

- **Spec:** samples for the current lesson's needed octave(s) are preloaded, and the lesson screen shows an explicit loading state, before the screen becomes interactive — "never a silently broken piano" (§22). Adjacent octaves prefetch during idle time, not before the current lesson is playable.
- **Spec:** browsers that block audio until a user gesture (autoplay policy) show a visible "tap to enable sound" affordance — never a silent piano on first load. This is an explicit, tested acceptance criterion (§29): "Browsers that block audio until a gesture show a 'tap to enable sound' prompt — never a silently broken piano."

⚠ **Open question (NON-BLOCKING):** PRODUCT_SPEC.md does not give a numeric time budget for "how long is initial sample loading allowed to take before it counts as a problem." **Recommended, not extracted from the spec:** set this from real Lighthouse CI / device data once M1 hardening begins (`docs/PRODUCT_SPEC.md` §28), rather than guessing a number here with no measurement behind it. Until then, the qualitative requirement (a visible loading state, never a silent gap) is the binding one.

### PERF-3 — Firestore writes: batched at exercise completion, never per-note (Spec)

**Target: exactly one write operation (a `PracticeSession` create + the corresponding `ProgressSummary` update on `UserProfile`) per completed exercise/lesson attempt — zero writes during the attempt itself.**

- §11 (Sync Layer): "buffers session results and writes them to Firestore on exercise completion (not per note)."
- §22: "Firestore writes: batched at exercise completion, not per note."
- Mechanism: this happens as a side effect of one `getExerciseFeedback` invocation (`docs/PRODUCT_SPEC.md` §20) — the client sends the buffered `NoteEvent[]` once, at the end. **Recommendation, not explicitly spec'd:** implement the `PracticeSession` write and the `ProgressSummary` map update as a single Firestore transaction or batched write inside that one Function invocation, so the two either both succeed or both fail atomically — the alternative (two separate writes that could partially fail) would leave a session logged with no matching progress update, or vice versa.

**Verification:** a test counting Firestore write calls triggered by one complete exercise attempt, asserting the count is 1 (or 1 batch), not N.

### PERF-4 — Dashboard reads: one flattened document, not N (Spec)

**Target: the Dashboard's initial data load is exactly one Firestore read** (`users/{uid}`, which embeds `ProgressSummary` directly per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15: "embedded on this same document, not a separate read") — never one read per lesson.

- §17: "Flattened deliberately — at ~12–15 MVP lessons, one map on the user doc costs 1 read for the whole dashboard instead of N; revisit only if lesson count grows enough to matter."
- §22: "the dashboard reads one flattened progress doc, not N per-lesson docs."
- This is a data-model decision (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7, §4.15), and this requirement exists specifically to prevent the client's hooks layer from being "tempted to fan out into N subscriptions" despite the data model already supporting a single read (a risk `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §8 names directly).

**Verification:** a test or manual network-panel check asserting the Dashboard route triggers exactly one `onSnapshot` subscription for progress data.

### PERF-5 — Bundle code-splitting (Spec, qualitative + Recommended, numeric)

**Spec:** the AI Tutor chat UI and any Phase-2-only modules (a future MIDI adapter, a future Daily Practice Plan UI) are code-split, separate from the initial bundle for the core piano/lesson experience (§22). Per `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §8, this is enforced at the Next.js App Router level — route-based splitting is the default, and the rule is simply that a Phase-2-only module is never imported from a route that ships in MVP, which would silently pull it back into the shared bundle.

⚠ **Open question (NON-BLOCKING):** no specific KB/size budget for the core bundle is given in PRODUCT_SPEC.md. As with PERF-2, recommend setting a concrete budget from real Lighthouse CI measurement during hardening (`docs/TESTING/PERFORMANCE_TESTING.md`) rather than asserting an unmeasured number here.

**Verification:** Lighthouse CI budget on the Lesson Player route specifically — §26: "it's the whole product — if that page is slow, nothing else matters." Bundle-analyzer output showing the Tutor/Phase-2 chunks are separate from the Lesson Player's initial chunk is a reasonable build-time check alongside the runtime Lighthouse budget.

### PERF-6 — Gemini call latency: 1–3 seconds acceptable, because of how it's gated (Spec)

**Target: 1–3 seconds is an acceptable response time for a Gemini call, conditional on two things both being true — the call is gated to an explicit learner action, and a visible "thinking" state is shown for the duration.** This is a compound requirement, not a standalone latency number to optimize:

- §14: Gemini is called "only on explicit learner action (opening the tutor chat, tapping 'ask why,' requesting today's plan) or at the end of a scored exercise. Never polled, never called per keystroke, never called 'just in case.'"
- §22: "since calls are already gated to explicit user actions, 1–3 seconds is acceptable there as long as the UI shows a clear 'thinking' state — **this is not a place to over-invest engineering effort chasing speed.**"

The explicit gating is what makes 1–3 seconds acceptable — it is not evidence that Gemini latency doesn't matter at all, and it is not a target to sprint-optimize below 1 second. Do not spend engineering effort shaving Gemini response time; do spend it making sure the "thinking" state renders immediately and never leaves the learner staring at a static screen.

**Verification:** a UI test asserting a loading/"thinking" state renders within the same frame the request is sent, for both `getTutorResponse` and the feedback-prose portion of `getExerciseFeedback`.

### PERF-7 — Rendering: per-key memoization (Spec)

**Target: pressing one piano key must not trigger a re-render of any other key.** This is a binary, decidable requirement even without a numeric budget:

- §22: "the virtual piano is memoized per-key/per-octave so pressing one key doesn't re-render the whole keyboard — a common, avoidable perf bug in naive piano UIs."
- `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §8 ties this directly to the state-management design: engine/audio state changes are scoped narrowly in Zustand (one store slice per concern, not one monolithic store) specifically so a key-press's state update doesn't fan out into a broad store change that would force sibling keys to re-render.

**Verification:** a component test or React DevTools profiler check asserting only the pressed key's component re-renders on a `NoteEvent`, not its siblings.

## 4. Requirements Summary Table

| ID | Requirement | Target | Firmness |
|---|---|---|---|
| PERF-1 | Key-press-to-sound latency | <50ms, mid-range Android + current iPhone, all input methods | Spec |
| PERF-2 | Sample preload before lesson screen interactive | No silent gap; loading state always shown | Spec (qualitative); numeric budget Recommended |
| PERF-3 | Firestore writes per exercise attempt | 1 (batched at completion) | Spec |
| PERF-4 | Firestore reads for Dashboard load | 1 (flattened doc) | Spec |
| PERF-5 | Bundle code-splitting (Tutor/Phase-2) | Separate chunks from core bundle | Spec (qualitative); numeric budget Recommended |
| PERF-6 | Gemini call latency | 1–3s acceptable, gated + "thinking" state required | Spec |
| PERF-7 | Per-key render isolation | Sibling keys never re-render | Spec |

## 5. Testing Requirements

Methodology for every verification method named above — Lighthouse CI configuration, the manual Android-device test protocol, any RUM/telemetry setup — is owned by `docs/TESTING/PERFORMANCE_TESTING.md`, referenced by name here rather than duplicated. This document is the source of truth for *the numbers*; that one is the source of truth for *how they get checked, and how often*.

Two testing-strategy facts worth restating here because they explain *why* two different test types are both needed for PERF-1 specifically (§26): "a Lighthouse CI budget on the Lesson Player route... plus a manual key-press-to-sound latency check on a real mid-range Android device, since that specific number doesn't show up in Lighthouse." Automated CI catches regressions in bundle size and general page performance; it does not, and cannot, measure real device audio latency — that gap is exactly why a manual check stays in the process rather than being fully automated away.

## 6. Acceptance Criteria

Directly from `docs/PRODUCT_SPEC.md` §29, restated here as this document's own bar:

- [ ] Key press (mouse, touch, or mapped computer key) produces sound in under ~50ms on a mid-range Android device and a current iPhone (PERF-1).
- [ ] Browsers that block audio until a gesture show a "tap to enable sound" prompt — never a silently broken piano (PERF-2).
- [ ] Mouse, touch, and computer-keyboard input all produce identical engine and audio behavior for the same note (PERF-1).

Plus acceptance items specific to this document's own numbered requirements:

- [ ] One exercise attempt completion produces exactly one Firestore write operation (PERF-3).
- [ ] The Dashboard's initial load triggers exactly one Firestore read/subscription for progress data (PERF-4).
- [ ] The Lesson Player route's initial bundle does not include the AI Tutor chat UI or any Phase-2-only module (PERF-5).
- [ ] A "thinking" state renders immediately (same frame) on every Gemini-calling action, before the response arrives (PERF-6).
- [ ] Pressing one piano key does not re-render any other key's component (PERF-7).

## 7. Out of Scope

- How each requirement gets measured (Lighthouse CI config, device-test protocol, RUM) — `docs/TESTING/PERFORMANCE_TESTING.md`.
- Acoustic pitch detection, MIDI input latency, offline-first PWA asset caching, microphone-based input — all Phase 2/3 (`docs/PRODUCT_SPEC.md` §8, §13), not MVP-scoped, so no target is set here.
- Gemini cost mechanics (as distinct from latency) — `docs/AI/AI_COST_CONTROL.md`, `docs/PRODUCT_SPEC.md` §31.
- Firestore Security Rules performance/cost implications — `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.

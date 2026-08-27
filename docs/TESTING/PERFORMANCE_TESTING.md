# Performance Testing

## 1. Purpose

Defines performance testing for AI Piano Tutor, split into the two categories `docs/PRODUCT_SPEC.md` §26 explicitly names as distinct: an automated Lighthouse CI budget on the Lesson Player route, and a manual key-press-to-sound latency check on real Android hardware — a number that does not show up in Lighthouse at all and cannot be replaced by it.

**Source of truth:** `docs/PRODUCT_SPEC.md` §12 (Audio Architecture — the ~50ms latency target and its two named implementation gotchas), §22 (Performance), §26, §29, §30 (Risks — audio latency named as the single biggest "doesn't feel like a real piano" risk). The authoritative, detailed latency budget and its measurement methodology is owned by `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md` (parallel batch) — this document tests against that spec's numbers; where this document states a target inline, treat `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md` as authoritative if the two ever differ, since it is the dedicated owner of that number.

## 2. Scope

Covers: Lighthouse CI budget definition and enforcement on the Lesson Player route specifically, the manual real-device audio-latency test procedure, rendering-performance checks (piano re-render scoping), and Firestore write-batching verification. Does not cover: Gemini response latency (explicitly *not* latency-sensitive per §22 — see §6 below), server-side Cloud Function cold-start optimization beyond what's needed to keep the reference E2E scenario responsive, or load/concurrency testing (`docs/TESTING/TESTING_STRATEGY.md` §9).

## 3. Why the Lesson Player Route, Specifically

Per `docs/PRODUCT_SPEC.md` §26: *"it's the whole product — if that page is slow, nothing else matters."* The Lesson Player is where the core interaction loop (`docs/PRODUCT_SPEC.md` §9: Learn → Demonstrate → Practice → Feedback → Repeat → Progress) actually happens, and it's the screen carrying the heaviest real-time burden: audio sample loading, the piano UI, live key-state rendering, and the network round-trip to `getExerciseFeedback`. A Lighthouse budget on the Landing or Dashboard screen would be easier to keep green and would tell you almost nothing about whether the product's core promise holds. Every milestone from M2 onward that touches the Lesson Player must keep this budget green — it is not a one-time M5 check bolted on at the end (see `docs/ROADMAP/M5_HARDENING.md` §6 and `docs/TESTING/TESTING_STRATEGY.md` §8).

## 4. Lighthouse CI Budget

- **Runs in CI on every PR** touching `apps/web` routes that affect the Lesson Player (the route itself, shared layout/nav, the piano/audio components it embeds).
- **Budget categories to set thresholds on** (exact numeric budgets owned by `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md` — this document specifies *what* must be budgeted, not the final numbers): Largest Contentful Paint, Total Blocking Time, Cumulative Layout Shift, and JavaScript bundle size delivered to this route specifically.
- **Mobile-first measurement:** Lighthouse CI runs its mobile/throttled-network profile as the primary gate, matching the product's mobile-first principle (`docs/PRODUCT_SPEC.md` §9) — a desktop-only green result does not satisfy this budget.
- **Bundle-size regression check:** per §22's code-splitting requirement (the AI Tutor chat UI and Phase-2-only modules are code-split so the core piano/lesson bundle stays lean), assert the Lesson Player route's JS payload does not regress past its budget when unrelated features are added elsewhere in the app — a Tutor-screen feature accidentally pulled into the Lesson Player's bundle (a code-splitting boundary violation) should fail this check even if the Tutor screen itself is fast.
- **A failing budget blocks merge**, same as any other CI gate in this repository (`docs/TESTING/TESTING_STRATEGY.md` §8) — it is not an advisory report to review later.

## 5. Manual Key-Press-to-Sound Latency Check — Required, Cannot Be Automated Away

Per `docs/PRODUCT_SPEC.md` §12's explicit warning: *"Android Chrome has historically had higher baseline audio latency than iOS Safari — spot-test on a real mid-range Android device, not just a dev laptop, before considering the piano 'done.'"* And §26: *"a manual key-press-to-sound latency check on a real mid-range Android device, since that specific number doesn't show up in Lighthouse."*

**Target:** under ~50ms from key-press to audible sound (`docs/PRODUCT_SPEC.md` §12, §29), cross-referenced against the authoritative figure in `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md`.

**Procedure:**
1. Test on a genuine **mid-range Android device** running Chrome — not a flagship phone, not an emulator/simulator, not a dev laptop. "Mid-range" specifically because that's the hardware profile most likely to expose the real-world gap §12 warns about; a flagship device or a laptop's audio stack will not surface the same latency characteristics.
2. Also test on a **current iPhone** (Safari) as the comparison baseline named alongside the Android target in §29's acceptance criterion.
3. Measure key-press-to-audible-sound latency using a method sensitive enough to catch sub-100ms differences — e.g. a high-frame-rate video capture of the tap and the device's audio-visualization/waveform output, timestamped frame-by-frame, or an external audio-latency measurement rig if available. A human pressing a key and subjectively judging "that felt fast enough" is not a substitute for a timestamped measurement — the whole reason this check exists is that subjective impressions on a dev laptop have historically hidden this exact class of problem.
4. Test with samples already preloaded (the steady-state case the ~50ms target describes) and separately note (not necessarily gated on the same target) the cold-load-in-progress case, since §22 requires a visible loading state precisely because the piano must never be silently broken while samples are still loading.
5. Record results with device model, OS/browser version, and date — this is a spot-check meant to be **repeated**, not a one-time box to check. Re-run whenever the Audio Layer, sample loading strategy, or Tone.js version changes materially, and at minimum once before `docs/ROADMAP/M5_HARDENING.md` sign-off.

**When this check fails:** per `docs/PRODUCT_SPEC.md` §12's stated fallback plan, a SoundFont fallback for low-bandwidth/high-latency conditions is "a legitimate Phase 2 optimization if telemetry ever shows it's needed — don't build it speculatively into the MVP." A failing latency check is therefore a signal to investigate the current Tone.js/sample-loading configuration first (sample format, preload scope, `AudioContext` handling), not an automatic trigger to scope in the SoundFont fallback ahead of schedule.

## 6. Explicitly Not a Performance Target: Gemini Latency

Per `docs/PRODUCT_SPEC.md` §22: since Gemini calls are already gated to explicit user actions (§14 — never polled, never per-keystroke), *"1–3 seconds is acceptable there as long as the UI shows a clear 'thinking' state — this is not a place to over-invest engineering effort chasing speed."* This document deliberately sets **no** Lighthouse or latency budget on `getTutorResponse`/`getExerciseFeedback`'s AI-explanation latency. What *is* tested for the Tutor screen: that a "thinking"/loading state renders promptly while a Gemini call is in flight (an E2E/UI concern, `docs/TESTING/E2E_TESTING.md`), not that the call itself completes within any tight budget.

## 7. Rendering & Sync-Layer Performance Checks

- **Piano re-render scoping (§22):** verify (via React DevTools profiler or an equivalent render-count assertion in a component test) that pressing one key does not re-render the entire keyboard — only the affected key/octave should update. This is named in `docs/PRODUCT_SPEC.md` §22 as "a common, avoidable perf bug in naive piano UIs," worth a dedicated regression check rather than trusting it to stay correct by convention.
- **Firestore writes batched at exercise completion, not per note** (§11's Sync Layer, §22): assert (at the integration level, cross-referencing `docs/TESTING/INTEGRATION_TESTING.md`) that a full exercise attempt produces exactly one `practiceSessions` write, not one per `NoteEvent`.
- **Dashboard reads one flattened progress doc, not N per-lesson docs** (§17, §22): assert the Dashboard's data-loading path issues a single read against `users/{uid}` (which embeds `progressSummary`) rather than N reads against per-lesson documents.

## 8. Error Handling

- A Lighthouse CI budget failure reports which specific metric regressed and by how much, not just pass/fail — this is standard Lighthouse CI behavior and should not be configured away in favor of a bare pass/fail gate.
- A failed manual latency check is recorded with enough detail (device, measurement method, actual measured value) to be actionable, not just "felt slow."

## 9. Acceptance Criteria (Restated From `docs/PRODUCT_SPEC.md` §29)

- [ ] Key press (mouse, touch, or mapped computer key) produces sound in under ~50ms on a mid-range Android device and a current iPhone.
- [ ] Browsers that block audio until a user gesture show a "tap to enable sound" prompt — never a silently broken piano (verify this is tested, even though it's a correctness/UX check more than a raw-speed one — it belongs here because it's the direct companion to the autoplay-policy gotcha named alongside the latency target in §12).
- [ ] Lighthouse CI budget on the Lesson Player route is green and enforced as a CI gate, not an advisory report.
- [ ] Piano re-rendering is scoped to the affected key/octave, verified by a render-count check.
- [ ] Firestore writes are batched at exercise completion, not per note.

## 10. Out of Scope

- Gemini call latency as a budgeted/gated metric — explicitly not a target (§6).
- Load/concurrent-user testing — not a pre-launch concern (`docs/TESTING/TESTING_STRATEGY.md` §9).
- MIDI input latency — Phase 2 (`docs/PRODUCT_SPEC.md` §8, §11); no MIDI-specific performance target exists in any MVP milestone.
- Full offline/PWA performance characteristics — Phase 2 PWA scope (`docs/PRODUCT_SPEC.md` §23); MVP performance testing covers Firestore's built-in offline persistence behaving reasonably, not a full offline-first performance profile.

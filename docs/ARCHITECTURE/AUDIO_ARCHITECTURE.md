# Audio Architecture

**Status:** Draft v1.0 — architecture documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §11, §12, §22, §29, §30
**Companion documents:** `docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` (the boundary rule that Audio is triggered directly by the Input Layer, not routed through the Engine Core, is owned there and restated here), `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md`, `docs/ARCHITECTURE/FIREBASE_ARCHITECTURE.md` (why samples are not in Firebase Storage)

## 1. Purpose

Define how AI Piano Tutor produces sound: the technology choice, the comparison that justifies it, the loading strategy, and the two implementation gotchas (autoplay policy, latency target) that are common, real bugs in browser instrument apps rather than hypothetical edge cases.

## 2. Scope

In scope: the audio stack (Tone.js + sample source), sample loading/prefetch strategy, licensing/attribution obligations, autoplay handling, and the latency target.

Out of scope: music-theory scoring (`MUSIC_ENGINE_ARCHITECTURE.md`), acoustic pitch detection / microphone input (explicitly a non-goal — Phase 3, `docs/PRODUCT_SPEC.md` §13, §8), MIDI velocity-sensitive playback nuance beyond what's noted in §7 (Phase 2, `docs/PRODUCT_SPEC.md` §11).

## 3. Requirements

- **R1 — Sounds like a piano, not a synthesizer.** The core product promise is a *piano* app; a synthesized tone undercuts it (`docs/PRODUCT_SPEC.md` §12).
- **R2 — Under ~50ms key-press-to-audible-sound**, the general threshold for "feels responsive" (`docs/PRODUCT_SPEC.md` §12, §29), verified on real mid-range Android hardware and a current iPhone, not just a dev laptop.
- **R3 — Never silently broken.** A browser that blocks audio until a user gesture must show a clear "tap to enable sound" affordance; a silent piano on first load is a support-ticket-generating bug, not a hypothetical (`docs/PRODUCT_SPEC.md` §12).
- **R4 — No network dependency.** Once loaded, audio playback has zero Firebase/Gemini involvement, consistent with the closed local loop in `SYSTEM_ARCHITECTURE.md` §4.3.

## 4. Architecture

### 4.1 Technology comparison

Reproduced from `docs/PRODUCT_SPEC.md` §12 — the brief asked for a comparison, not a default:

| Approach | Realism | Payload/perf | Dev effort | Verdict |
|---|---|---|---|---|
| Raw Web Audio oscillators | Low — sounds synthesized, not like a piano | Minimal | High (hand-roll envelopes/scheduling) | Rejected as the primary voice — undercuts R1 |
| Tone.js (wraps Web Audio) | N/A — it's a framework, not a sound source | Small library overhead | Low | **Adopted as the framework layer** regardless of sample source — its Transport/scheduling is needed for demo playback, metronome, and timing scoring either way |
| SoundFont / WebAudioFont | Medium — synthesized from samples, noticeably "MIDI-ish" | Small (~1–3MB) | Low | Good low-bandwidth fallback, not primary |
| Real piano samples via `Tone.Sampler` | High — actual recorded piano, velocity layers | Larger, but manageable with lazy loading | Low–Medium | **Recommended primary** |

### 4.2 Recommendation

`Tone.js` as the framework layer (scheduling, transport, envelope handling), with `Tone.Sampler` loaded from a trimmed **Salamander Grand Piano** sample set as the sound source.

> ⚠ **Licensing fact, flagged per this document's own standard of evidence:** Salamander Grand Piano is stated in `docs/PRODUCT_SPEC.md` §12 to be **CC BY 3.0** — free for commercial use, with attribution to Alexander Holm legally required. This fact is stated in `docs/PRODUCT_SPEC.md`, verified via web search at time of writing (`docs/PRODUCT_SPEC.md`'s own Sources Consulted list cites the [Salamander Grand Piano GitHub repository](https://github.com/sfzinstruments/SalamanderGrandPiano)) — **reverify the license terms directly against that source before shipping**, since license terms and hosting locations for freely-distributed sample sets can change, and this document cannot independently re-verify it at doc-writing time. If verified, a visible credit (Settings/About screen) and a `NOTICE` file in the repo are required — a five-minute fix if done now, a compliance gap if forgotten at launch.

**Format:** compress to OGG/MP3, not raw WAV, to keep payload manageable.

**SoundFont fallback:** a legitimate Phase 2 optimization for very low-bandwidth conditions **if telemetry ever shows it's needed** — explicitly not built speculatively into MVP (`docs/PRODUCT_SPEC.md` §12). Do not implement a fallback path in MVP code; note the seam (an alternate sample-source implementation behind the same Audio Layer interface, §5) so it can be added later without restructuring.

### 4.3 Loading strategy

1. **Before the lesson/piano screen becomes interactive**, preload only the 1–2 octaves the current lesson actually needs — show a visible loading state, never a silently broken piano (`docs/PRODUCT_SPEC.md` §12, §22).
2. **During idle time**, prefetch adjacent octaves in the background so panning to a different range doesn't introduce a new load stall.
3. **Bundle boundary:** sample files are static assets served from the CDN alongside the rest of the app (`FIREBASE_ARCHITECTURE.md` §Why not Storage), not bundled into the JS bundle itself — this keeps the core piano/lesson JS bundle lean (`docs/PRODUCT_SPEC.md` §22) independent of how many octaves' worth of samples eventually get cached.

### 4.4 Boundary rule, restated from `MUSIC_ENGINE_ARCHITECTURE.md`

The Audio Layer is triggered **directly by the Input Layer**, receiving `NoteEvent`s in parallel with the Engine Core — not downstream of it (`docs/PRODUCT_SPEC.md` §11; full diagram: `MUSIC_ENGINE_ARCHITECTURE.md` §4.1). Sound plays the instant a key event fires; it does not wait for the Engine Core to judge correctness. This is the architectural reason R2 (§3) is achievable at all — if sound had to wait for a scoring verdict (even a fast, local one), the latency budget would include scoring logic, key-highlight computation, and re-render time, not just the audio pipeline itself.

## 5. Data structures

The Audio Layer's only input is `NoteEvent` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1) — it reads `midiNote` (which sample/pitch to trigger), `velocity` (defaults to 100 for non-velocity-sensitive sources; used to shape dynamics once a velocity-sensitive source like MIDI exists in Phase 2), and `type` (`noteOn` triggers attack, `noteOff` triggers release/sustain-pedal-aware note-off). It does not read or produce an `ExerciseVerdict` — correctness has no bearing on whether or how a note sounds.

No new shared types are introduced by the Audio Layer; sample-map/instrument-configuration structures are internal implementation detail, not a cross-module contract, and therefore not part of `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`'s scope.

## 6. Interfaces / contracts

The Audio Layer exposes, conceptually, one operation to its only caller (the Input Layer): given a `NoteEvent`, produce the corresponding sound. The reverse is never true — the Audio Layer never calls into the Input Layer, Engine Core, or Sync Layer. Sustain-pedal state and volume (both learner-configurable, `docs/PRODUCT_SPEC.md` §6, stored in `UserProfile.settings`) are read by the Audio Layer as ambient configuration, not passed per-`NoteEvent`.

## 7. Dependencies

- **Tone.js** — scheduling/transport/sampler framework.
- **Salamander Grand Piano sample set** (trimmed, OGG/MP3) — see licensing note, §4.2.
- **CDN / static asset hosting** — the same hosting layer serving the rest of the app (`FIREBASE_ARCHITECTURE.md`), explicitly not Firebase Storage.
- *(Phase 2, not built)* **MIDI velocity data**, once the `MIDIAdapter` exists (`MUSIC_ENGINE_ARCHITECTURE.md` §4.3) — the Audio Layer's `NoteEvent.velocity` handling already accepts a 0–127 range today, so no interface change is anticipated when MIDI ships, only richer input data.

## 8. Performance considerations

- **Latency target: ~50ms**, key-press to audible sound (`docs/PRODUCT_SPEC.md` §12, §29). Tone.js with preloaded samples comfortably hits this on modern devices.
- **Android Chrome vs. iOS Safari variance:** Android Chrome has historically had higher baseline audio latency than iOS Safari. This is not a target to design around abstractly — `docs/PRODUCT_SPEC.md` §12 and §30 both call for a **spot-test on a real mid-range Android device**, not just a dev laptop or a flagship phone, before the piano is considered "done." This is listed explicitly as the single biggest "this doesn't feel like a real piano" risk in `docs/PRODUCT_SPEC.md` §30.
- **Rendering coupling:** the virtual piano is memoized per-key/per-octave (`APPLICATION_ARCHITECTURE.md` §8) so that a key press's audio trigger isn't entangled with a full-keyboard re-render — a naive implementation that re-renders every key on every press would reintroduce latency the audio pipeline itself doesn't have.
- **Bundle size:** the sample assets are lazy-loaded (§4.3), not part of the initial JS bundle, keeping the Lesson Player route's Lighthouse budget (`docs/PRODUCT_SPEC.md` §26) achievable.

## 9. Error handling

- **Autoplay policy:** browsers block audio until a user gesture. The `AudioContext` must be created/resumed on the first tap, with a visible "tap to enable sound" affordance — restated as R3 (§3) because `docs/PRODUCT_SPEC.md` §12 explicitly flags this as "a common real bug in browser instrument apps, not a hypothetical." A silent piano on first load must never happen silently — if the context is suspended, the UI must say so and offer the tap affordance, not fail quietly.
- **Sample load failure:** consistent with the "never a dead end" principle applied elsewhere in this app (`docs/PRODUCT_SPEC.md` §10), a failed sample fetch should surface a visible retry state on the loading indicator from §4.3, not a piano that looks ready but produces no sound when pressed.
- **Degraded network during prefetch:** the idle-time prefetch of adjacent octaves (§4.3) is best-effort — if it fails, the current lesson's already-loaded octave(s) remain playable; only panning to an unloaded range should trigger a fresh loading state, not a full-piano failure.

## 10. Testing requirements

- **Playwright, piano interaction:** key press → sound triggers → visual state updates (`docs/PRODUCT_SPEC.md` §26) — shared with `APPLICATION_ARCHITECTURE.md`'s UI/interaction test surface, since this is the same end-to-end press-to-sound path.
- **Manual latency check:** a key-press-to-sound latency measurement on a real mid-range Android device, since this specific number does not show up in Lighthouse or any automated CI check (`docs/PRODUCT_SPEC.md` §26).
- **Lighthouse CI budget** on the Lesson Player route — it's the whole product; if that page is slow, nothing else matters (`docs/PRODUCT_SPEC.md` §26).

## 11. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29:
- [ ] Key press (mouse, touch, or mapped computer key) produces sound in under ~50ms on a mid-range Android device and a current iPhone.
- [ ] Browsers that block audio until a gesture show a "tap to enable sound" prompt — never a silently broken piano.
- [ ] Mouse, touch, and computer-keyboard input all produce identical audio behavior for the same note.

## 12. Out of scope

- Note/chord/scale correctness logic → `MUSIC_ENGINE_ARCHITECTURE.md`, `docs/MUSIC/MUSIC_ENGINE_SPEC.md`
- Acoustic pitch detection (microphone-based input) → explicitly a Phase 3, unbuilt, "fundamentally different problem" (`docs/PRODUCT_SPEC.md` §13, §8) — not designed for here, not even at interface level, unlike the MIDI adapter
- Why samples are hosted on a CDN rather than Firebase Storage → `FIREBASE_ARCHITECTURE.md`
- MIDI adapter's event normalization → `MUSIC_ENGINE_ARCHITECTURE.md` §4.3

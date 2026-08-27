# Input System

**Status:** Draft v1.0 — music engine documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §5, §6, §11, §30; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1 (`NoteEvent`)
**Companion documents:** `docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` §4.3 (the architecture-level summary of the adapter contract this document details), `docs/MUSIC/KEYBOARD_MAPPING.md` (the default mapping table the computer-keyboard adapter reads), `docs/MUSIC/MUSIC_ENGINE_SPEC.md`, `docs/ARCHITECTURE/AUDIO_ARCHITECTURE.md` (consumes the same `NoteEvent`s in parallel)

---

## 1. Purpose

Specify, precisely enough to implement without clarification, how each input method — mouse/touch, computer keyboard, and (Phase 2, designed-for-not-built) MIDI — normalizes its own raw browser events into the one common `NoteEvent` shape every downstream layer consumes. `docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` §4.3 states the adapter contract at the architecture level ("produce a stream of `NoteEvent`s, tagged with the correct `source`, and nothing else"); this document is that architecture-level contract's detailed, per-adapter algorithm specification — the same relationship `docs/MUSIC/MUSIC_ENGINE_SPEC.md` itself has to `MUSIC_ENGINE_ARCHITECTURE.md`.

**A scope note that matters for reading this document correctly:** the Input Layer lives in `apps/web`, **not** `packages/music-engine` (`docs/ENGINEERING/PROJECT_STRUCTURE.md` §3, §6: `apps/web` depends on `packages/music-engine`, never the reverse). This document is grouped under `docs/MUSIC/` because it shares this cluster's MIDI-note vocabulary and because an implementer building the engine needs to understand exactly what shape of `NoteEvent` will arrive — but the code this document describes is upstream of, and architecturally separate from, the engine itself. `docs/MUSIC/KEYBOARD_MAPPING.md` is the same kind of adjacent-but-separate concern.

## 2. Scope

In scope: the exact field-derivation logic for each of the three adapters, the MIDI-message-byte-level detail of the Phase-2 MIDI adapter, the Safari/Web-MIDI feature-detection requirement, and input validation at the adapter boundary.

Out of scope: chord/scale/note detection (`docs/MUSIC/CHORD_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/NOTE_SYSTEM.md` — this document's output feeds those, it does not implement them), the default keyboard-to-note mapping table itself (`docs/MUSIC/KEYBOARD_MAPPING.md`), Tone.js/audio triggering (`docs/ARCHITECTURE/AUDIO_ARCHITECTURE.md` — the Audio Layer receives the same `NoteEvent`s this document produces, in parallel with the Engine Core, not downstream of it).

## 3. Requirements

- **Every adapter normalizes to the exact same `NoteEvent` shape** (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1) — nothing downstream ever branches on which adapter produced an event except for the diagnostic-only `source` field (`docs/PRODUCT_SPEC.md` §11: "Nothing downstream ever needs to know whether a note came from a click or a MIDI cable").
- **Mouse, touch, and computer-keyboard input must produce identical engine and audio behavior for the same note** — a named MVP acceptance criterion (`docs/PRODUCT_SPEC.md` §29).
- **MIDI must be feature-detected, never assumed.** As of August 2026 (verified, `docs/PRODUCT_SPEC.md` §11, §30), Safari on both macOS and iOS does not support the Web MIDI API and Apple has not committed to adding it; Chrome, Edge, and Opera support it at roughly 80% global coverage. Since this app is mobile-first and a large share of mobile learners are on iOS Safari, the MIDI adapter must be gated behind `'requestMIDIAccess' in navigator` and degrade to a clear "MIDI isn't available in this browser" message — never a silent failure, never assumed as a baseline input method.
- **`midiNote` and `velocity` are validated at the adapter boundary**: integers in `[0, 127]`; out-of-range values are rejected — logged, not thrown — including from misbehaving MIDI hardware (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1).

## 4. Architecture

### 4.1 Mouse/touch adapter

**Raw input:** DOM pointer events (`pointerdown`/`pointerup`, or the `mousedown`/`mouseup`/`touchstart`/`touchend` families if `PointerEvent` support is not assumed) on each piano key's rendered element.

**Field derivation:**

| `NoteEvent` field | Derivation |
|---|---|
| `midiNote` | Read directly from the pressed key element's own data attribute (e.g. `data-midi-note`), set by the UI Layer when it renders that key — the adapter never computes a pitch itself, it only reads what the UI already knows about the element the user touched. |
| `type` | `'noteOn'` on `pointerdown`/`touchstart`; `'noteOff'` on `pointerup`/`touchend`. |
| `velocity` | Fixed at `100` — mouse and touch are non-velocity-sensitive input sources (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1: "Non-velocity-sensitive sources... default to 100"). |
| `timestamp` | High-resolution, relative to session/exercise start (`performance.now()`-based, per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §3's convention — not wall-clock `Date.now()`). |
| `source` | `'mouse'` or `'touch'`, determined from the event type/`PointerEvent.pointerType` that triggered it. |

**A defensive edge case worth stating explicitly:** a pointer/touch sequence can be interrupted by the OS or browser before a matching `pointerup`/`touchend` fires (e.g., an incoming call, a browser gesture takeover, a finger sliding off-screen). The adapter must also listen for `pointercancel`/`touchcancel` and treat either as equivalent to a release (`type: 'noteOff'`) — otherwise a single interrupted gesture leaves a key logically "stuck on" with no way for the learner to release it, a real, avoidable bug class in touch-driven instrument UIs, not a hypothetical.

### 4.2 Computer-keyboard adapter

**Raw input:** `keydown`/`keyup` DOM events, listened for at the document/window level (not per-key-element, since the physical keyboard has no visual key element of its own the way a rendered piano key does).

**Field derivation:**

| `NoteEvent` field | Derivation |
|---|---|
| `midiNote` | The pressed physical key is resolved through the learner's `KeyboardMapping` (`docs/MUSIC/KEYBOARD_MAPPING.md`) to a semitone offset, then added to the mapping's current root note to produce a `midiNote`. |
| `type` | `'noteOn'` on `keydown`; `'noteOff'` on `keyup`. |
| `velocity` | Fixed at `100` (same non-velocity-sensitive-source rule as §4.1). |
| `timestamp` | Same convention as §4.1. |
| `source` | `'keyboard'`. |

**PROPOSED — physical-layout key identification, not extracted from `docs/PRODUCT_SPEC.md`:** the adapter reads `KeyboardEvent.code` (the physical key position, e.g. `"KeyA"`, `"KeyW"`), not `KeyboardEvent.key` (the character the OS keyboard layout would actually type, e.g. `"a"`, `"q"`, or a locale-specific character). `docs/MUSIC/KEYBOARD_MAPPING.md`'s default table (`A, W, S, E, D, F, T, G, Y, H, U, J, K`) is a **physical position** mapping in the virtualpiano.net tradition — the point is "the key where your finger already knows to reach on a QWERTY-shaped keyboard," not "whatever character that key currently produces." Using `event.code` keeps the mapping correct on non-QWERTY OS layouts (e.g., a learner with an AZERTY system layout still gets the same physical-key piano mapping a QWERTY user does, rather than a shifted, confusing one); using `event.key` would silently break the mapping for exactly the international users this app is also targeting (`docs/PRODUCT_SPEC.md` §18 names POPIA/South Africa and a "realistic chance of EU/UK users" explicitly). `docs/MUSIC/KEYBOARD_MAPPING.md`'s stored string values are therefore interpreted as the letter suffix of a `KeyboardEvent.code` value (`"A"` → `"KeyA"`), not a literal character to match against `event.key`.

⚠ **Open question (NON-BLOCKING):** this `event.code`-vs-`event.key` choice is not stated anywhere in `docs/PRODUCT_SPEC.md` — it is a necessary implementation decision this document makes and states explicitly (marked PROPOSED) rather than leaving silent, since the two approaches produce genuinely different, non-interchangeable behavior for any learner not on a US QWERTY physical keyboard.

**The mapping's "root note"** — which absolute octave the mapping is anchored to at any given moment — defaults to Middle C (`60`), matching the emphasis `docs/PRODUCT_SPEC.md` §5 places on Middle C as the learner's first anchor point. `docs/MUSIC/KEYBOARD_MAPPING.md` §Architecture covers this in full, including its relationship to the piano's separately adjustable visible-octave pan (`docs/PRODUCT_SPEC.md` §6).

### 4.3 MIDI adapter (Phase 2 — designed-for, not built)

Per `docs/PRODUCT_SPEC.md` §11: a `MIDIAdapter` listens via `navigator.requestMIDIAccess()` and normalizes `MIDIMessageEvent` note-on/note-off (with velocity and timestamp) into the same `NoteEvent` shape as every other adapter. This section specifies the byte-level detail needed to implement that normalization correctly, since MIDI's wire format is unlike the other two adapters' DOM-event sources and has real, well-known gotchas.

**Access and permission:**

```
navigator.requestMIDIAccess()
  .then(midiAccess => { /* attach 'midimessage' listeners to each input port */ })
  .catch(() => { /* permission denied or hardware error — show the clear "MIDI isn't available" message, §Error handling */ })
```

Feature-detected first, per §3's requirement: `if ('requestMIDIAccess' in navigator)` gates whether this code path runs at all — on Safari (macOS/iOS), the method does not exist, and the check must fail cleanly into the "MIDI isn't available in this browser" state, not attempt the call and catch a runtime error.

**Message parsing:** each `MIDIMessageEvent.data` is a `Uint8Array` of `[status, data1, data2]`.

| Byte | Meaning |
|---|---|
| `status` | Upper nibble = message type (`0x9` = note-on, `0x8` = note-off; other values — control-change, pitch-bend, etc. — are ignored by this adapter, since only note on/off is relevant to `NoteEvent`); lower nibble = MIDI channel (`0`–`15`, not surfaced in `NoteEvent` — this app does not distinguish MIDI channels). |
| `data1` | MIDI note number, `0`–`127` — maps directly to `NoteEvent.midiNote`. |
| `data2` | Velocity, `0`–`127` — maps directly to `NoteEvent.velocity`, **except** the case below. |

**The well-known MIDI gotcha this adapter must handle correctly:** a note-on message (`status` upper nibble `0x9`) with `data2 === 0` (velocity zero) is, by long-standing MIDI convention, actually a **note-off** — many MIDI devices and DAWs use "running status" note-on-with-zero-velocity instead of a literal note-off byte as a wire-efficiency optimization. An adapter that naively maps "upper nibble `0x9`" straight to `type: 'noteOn'` without checking `data2` will misinterpret real note-off messages from real hardware as note-on events, producing stuck notes. The adapter must therefore compute `type` as:

```
type = (statusUpperNibble === 0x8 || (statusUpperNibble === 0x9 && data2 === 0)) ? 'noteOff' : 'noteOn'
```

**Timestamp:** `MIDIMessageEvent.timeStamp` is a `DOMHighResTimeStamp`, which in current browser implementations shares the same time origin as `performance.now()` used elsewhere in this app's timing model (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §3) — this should be treated as an assumption to verify at implementation time (browser timestamp-origin behavior has historically had inconsistencies across engines), not hard fact locked into this spec; a defensive normalization (capturing a reference offset once at adapter-init time and computing all subsequent event timestamps relative to it) is a reasonable safeguard if a discrepancy is ever observed.

**Source:** `'midi'`.

**Why this adapter's design belongs in this document even though it isn't built:** because every adapter's only contractual job is "emit correctly-shaped `NoteEvent`s," adding this adapter in Phase 2 requires zero changes anywhere downstream — not the Engine Core, not the Audio Layer, not the UI Layer (`docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` §4.3). Specifying it precisely now, rather than leaving it as a one-line "Phase 2 TODO," means Phase 2 implementation work starts from an already-reviewed design instead of a blank page — and the byte-level detail above (especially the velocity-zero-means-note-off case) can be captured as unit tests against synthetic, mocked `MIDIMessageEvent`-shaped fixtures well before any real MIDI hardware is available to test against, per §9 below.

## 5. Data structures

Every adapter in this document produces `NoteEvent` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1), reproduced for reference:

```typescript
interface NoteEvent {
  midiNote: number;
  type: 'noteOn' | 'noteOff';
  velocity: number;
  timestamp: number;
  source: 'mouse' | 'touch' | 'keyboard' | 'midi';
}
```

No new shared types are introduced by this document.

## 6. Interfaces / contracts

Each adapter's only externally visible effect is emitting `NoteEvent`s — via a callback, event emitter, or direct Zustand-store action (the exact mechanism is an `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` concern, not this document's). No adapter calls into the Engine Core, the Audio Layer, or Firestore directly, and no adapter consumes its own output (`docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` §4.3).

## 7. Dependencies

- **Mouse/touch, computer-keyboard:** browser DOM APIs only (`PointerEvent`/`TouchEvent`, `KeyboardEvent`), plus `docs/MUSIC/KEYBOARD_MAPPING.md` for the keyboard adapter's mapping lookup.
- **MIDI (Phase 2):** the browser-native Web MIDI API (`navigator.requestMIDIAccess`) — no npm dependency, per `docs/ENGINEERING/TECH_STACK.md` §2's note that MIDI support "doesn't change [the tech stack] document when it ships."

## 8. Error handling

- **`midiNote`/`velocity` out of `[0, 127]`, from any adapter** (including a misbehaving piece of MIDI hardware sending an out-of-spec value): rejected — logged, not thrown, per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1. The offending event is dropped before it becomes a `NoteEvent` at all; the rest of the input stream continues unaffected.
- **MIDI permission denied, or `requestMIDIAccess()`'s returned promise rejects for any other reason:** the UI shows the same clear "MIDI isn't available" affordance used for the feature-detection failure case (§3) — from the learner's perspective, "the browser doesn't support this" and "the browser refused permission" should look the same: a calm, non-blocking message, never a broken screen, and the rest of the app (mouse/touch/keyboard input) is completely unaffected either way.
- **Interrupted pointer/touch gesture:** handled via `pointercancel`/`touchcancel` per §4.1, preventing a stuck-key state.

## 9. Testing requirements

- **Playwright, mouse/touch/keyboard:** key press → `NoteEvent` emitted with correct `midiNote`/`type`/`source` → sound triggers → visual state updates, exercised for all three MVP input methods against the same target notes, asserting identical downstream behavior (`docs/PRODUCT_SPEC.md` §26, §29).
- **The computer-keyboard mapping**, specifically: pressing each of the 13 default-mapped physical keys (§4.2) produces the correct `midiNote` relative to the current root.
- **MIDI normalization (Phase 2, but testable now against synthetic fixtures):** unit tests constructing mock `MIDIMessageEvent`-shaped objects (a plain `{ data: Uint8Array([...]), timeStamp: number }`) do not require real hardware — this is a case where writing the test now, ahead of the adapter's implementation, is genuinely low-cost and de-risks Phase 2. Required cases: a standard note-on (`0x90, note, velocity>0`) → `type: 'noteOn'`; a standard note-off (`0x80, note, velocity`) → `type: 'noteOff'`; a note-on with zero velocity (`0x90, note, 0`) → `type: 'noteOff'` (the gotcha in §4.3); a non-note message (e.g. control-change, `0xB0, ...`) → ignored, no `NoteEvent` emitted.

## 10. Acceptance criteria

From `docs/PRODUCT_SPEC.md` §29:
- [ ] Mouse, touch, and computer-keyboard input all produce identical engine and audio behavior for the same note.
- [ ] Browsers that block audio until a user gesture are handled by the Audio Layer, not this document — but this document's adapters must still fire the underlying `NoteEvent` correctly regardless of `AudioContext` state (`docs/ARCHITECTURE/AUDIO_ARCHITECTURE.md` owns the gesture-gating itself).
- [ ] `'requestMIDIAccess' in navigator` gates every code path that would otherwise call the Web MIDI API — verified by a test running under a Safari-like environment (no `requestMIDIAccess`) confirming the app never throws and shows the "MIDI isn't available" message instead.

## 11. Out of scope

- Chord/scale/note detection consuming these `NoteEvent`s → `docs/MUSIC/CHORD_SYSTEM.md`, `docs/MUSIC/SCALE_SYSTEM.md`, `docs/MUSIC/NOTE_SYSTEM.md`
- The default keyboard mapping table itself and its customization UX → `docs/MUSIC/KEYBOARD_MAPPING.md`
- Tone.js sample triggering from a `NoteEvent` → `docs/ARCHITECTURE/AUDIO_ARCHITECTURE.md`
- Mouse-drag-across-keys ("glissando") behavior — not specified anywhere in `docs/PRODUCT_SPEC.md`; not designed here, left as an implementation detail for whoever builds the mouse/touch adapter to decide (or explicitly not support) rather than invented in this document

# Keyboard Mapping

**Status:** Draft v1.0 — music engine documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §5, §6, §11; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.11 (`KeyboardMapping`), §4.15 (`UserProfile.settings`)
**Companion documents:** `docs/MUSIC/INPUT_SYSTEM.md` §4.2 (the computer-keyboard adapter that reads this mapping), `docs/MUSIC/NOTE_SYSTEM.md` (MIDI note derivations this mapping resolves to)

---

## 1. Purpose

Define the default computer-keyboard-to-piano mapping and the `KeyboardMapping` data shape that makes it remappable, per `docs/PRODUCT_SPEC.md` §11.

## 2. Scope

In scope: the default mapping table, its `KeyboardMapping` representation, its relationship to the mapping's "root note," and validation rules for a learner-customized remap.

Out of scope: how the computer-keyboard adapter reads and applies this mapping to produce a `NoteEvent` (`docs/MUSIC/INPUT_SYSTEM.md` §4.2), the Settings screen UI for remapping (owned by whichever document specifies the Profile/Settings screen's UI in detail — not yet written in this batch).

## 3. Requirements

- The default mapping mirrors the layout used by sites like virtualpiano.net, so returning users already have muscle memory for it (`docs/PRODUCT_SPEC.md` §11) — this is a deliberate choice to reuse an existing convention, not invent a new one.
- The mapping is stored as **data**, not hardcoded per-component, so the Settings screen can let a learner remap it (§11).
- The mapping is user-customizable (`docs/PRODUCT_SPEC.md` §6: "Adjustable... keyboard-mapping customization" under Profile/Settings).

## 4. Architecture

### 4.1 The default mapping table

Reproduced from `docs/PRODUCT_SPEC.md` §11:

| Piano key | C | C# | D | D# | E | F | F# | G | G# | A | A# | B | C (next 8ve) |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Semitone offset | 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 |
| Keyboard key | A | W | S | E | D | F | T | G | Y | H | U | J | K |

Note the visual pattern this table encodes, worth stating explicitly since it's the actual mnemonic a learner (or an implementer double-checking the table) can lean on: the **white keys** (C, D, E, F, G, A, B, C) map to the bottom QWERTY row **A, S, D, F, G, H, J, K** in order, and the **black keys** (C#, D#, F#, G#, A#) map to the row above — **W, E, T, Y, U** — positioned directly above the white key immediately to their left, exactly mirroring the physical layout of black keys sitting between and above their neighboring white keys on a real piano. This is the same visual logic virtualpiano.net's own layout uses, which is the entire reason `docs/PRODUCT_SPEC.md` §11 calls it out as a convention worth reusing rather than reinventing.

### 4.2 `KeyboardMapping` representation

Shape from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.11, reproduced for reference:

```typescript
type KeyboardMapping = Record<number, string>; // keyed by semitone offset from the mapping's root (0-12)
```

**Default value**, as an explicit literal (this document is its canonical source, per §4.11: "Default value: defined in `docs/MUSIC/KEYBOARD_MAPPING.md`"):

```typescript
const DEFAULT_KEYBOARD_MAPPING: KeyboardMapping = {
  0:  "A",  // C
  1:  "W",  // C#
  2:  "S",  // D
  3:  "E",  // D#
  4:  "D",  // E
  5:  "F",  // F
  6:  "T",  // F#
  7:  "G",  // G
  8:  "Y",  // G#
  9:  "H",  // A
  10: "U",  // A#
  11: "J",  // B
  12: "K",  // C (next octave)
};
```

Per `docs/MUSIC/INPUT_SYSTEM.md` §4.2, each string value is interpreted as the letter suffix of a `KeyboardEvent.code` (`"A"` → `"KeyA"`), not a literal character to compare against `event.key` — this keeps the mapping correct regardless of the learner's OS keyboard layout.

### 4.3 The mapping's root note

A semitone offset alone (`0`–`12`) is not an absolute pitch — it needs a root `midiNote` to resolve against (`resolvedMidiNote = root + offset`). This document's default: **the root is Middle C (`60`)**, matching the emphasis `docs/PRODUCT_SPEC.md` §5 places on Middle C as the learner's first anchor point on the keyboard ("As a beginner, I want the app to show me where Middle C is") and keeping the computer-keyboard input consistent and predictable across the early lessons that establish that anchor.

This is deliberately **not** tied to the piano's separately adjustable visible-octave pan (`docs/PRODUCT_SPEC.md` §6: "Adjustable visible octave range... with pan") — panning the on-screen keyboard to look at a different octave (primarily a mouse/touch, small-screen affordance, `docs/PRODUCT_SPEC.md` §9) does not, by this document's design, shift what the computer-keyboard's `A`–`K` keys play. A learner using a full computer keyboard is very likely on a laptop/desktop with a larger screen already showing more of the piano at once, and a fixed root avoids surprising the learner mid-lesson if they happen to scroll/pan the visual keyboard incidentally.

⚠ **Open question (NON-BLOCKING):** `docs/PRODUCT_SPEC.md` does not explicitly state whether the keyboard mapping's root should ever move — e.g., for a later lesson demonstrating a scale or melody anchored somewhere other than Middle C's octave. This document's fixed-root-at-C4 default is a reasonable, spec-consistent inference (grounded in §5's Middle-C-as-anchor framing), not an explicit requirement — flagged so a future lesson requiring computer-keyboard input outside the C4–C5(+1) range prompts a deliberate decision (e.g., a per-lesson root override) rather than silently breaking.

## 5. Data structures

`KeyboardMapping` (§4.2) is stored at `users/{uid}.settings.keyboardMapping`, a field of `UserProfile.settings` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15) — **client-writable**, the one deliberate exception to "everything under `users/{uid}` beyond `settings` is Function-written only" (§4.11: "the one exception to 'everything under users/{uid} beyond settings is Function-written only'"), because a keyboard remap is a preference, not a scored fact.

## 6. Interfaces / contracts

No callable Cloud Function is needed to change this value — it is written directly by the client via the Firestore SDK, scoped to the owning `uid` by Firestore Security Rules (`docs/SECURITY/FIRESTORE_SECURITY_RULES.md` §`users/{uid}/settings`). The computer-keyboard adapter (`docs/MUSIC/INPUT_SYSTEM.md` §4.2) reads the current value from the same client-side state the Settings screen writes to (via the `onSnapshot`-backed `UserProfile` hook, `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §4.2) — there is no separate "apply the mapping" step; a remap takes effect on the next key press once the updated document round-trips through Firestore's realtime listener.

### 6.1 Validation for a custom (remapped) `KeyboardMapping`

⚠ **PROPOSED, not specified by `docs/PRODUCT_SPEC.md`:** since a learner can write directly to this field, the Settings UI should apply light validation before saving, to avoid an unusable mapping — though see the note on stakes below, this is a UX nicety, not a security concern:

- All 13 offsets (`0`–`12`) should be present with a value (an incomplete mapping means some notes become unplayable from the keyboard entirely).
- No two offsets should map to the same physical key (an ambiguous mapping — pressing that key would be undefined which note it means). The UI should warn on a conflicting remap attempt rather than silently accepting it.
- Each value should be a recognizable single-key `code` suffix (letters, digits, or a small set of punctuation keys) — not, for instance, a modifier key (`Shift`, `Ctrl`) or a multi-key combination, which the single-`keydown`-event model in `docs/MUSIC/INPUT_SYSTEM.md` §4.2 isn't designed to interpret as a chord of intent.

**Why this is low-stakes, unlike every scored field on `UserProfile`:** a malformed or incomplete custom mapping can only degrade the *owning learner's own* input experience — it has no bearing on scoring integrity, since `packages/music-engine` validates against raw `midiNote` values in a `NoteEvent`, regardless of which physical key produced them (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §3). There is no cheating vector here the way there is for `xp`/`streak`/`progressSummary` (`docs/SECURITY/SECURITY_REQUIREMENTS.md` SEC-08), so this validation is client-side UX guidance only — it does not need, and does not have, server-side re-validation.

## 7. Dependencies

`docs/MUSIC/NOTE_SYSTEM.md` (the root/offset arithmetic resolves to a `midiNote`, whose meaning that document owns); `packages/shared-types` for `KeyboardMapping`. Firestore, for persistence of a learner's custom mapping — the only document in the `docs/MUSIC/` cluster with a genuine Firestore dependency, since it is a stored user preference, not engine logic.

## 8. Error handling

- A missing or malformed `KeyboardMapping` document (e.g., a new anonymous user whose `settings` hasn't been initialized yet) falls back to `DEFAULT_KEYBOARD_MAPPING` (§4.2) client-side — the piano must never be unplayable via computer keyboard while waiting on a settings document to load.
- An invalid custom mapping that somehow reaches the adapter (validation in §6.1 was bypassed or not yet implemented) should not crash key handling — an offset with no mapped key simply produces no `NoteEvent` for that physical key, rather than throwing.

## 9. Testing requirements

- Each of the 13 default mappings produces the correct `midiNote` (root `60` + offset) when its physical key is pressed — a direct, table-driven test against §4.1's table.
- The Settings screen's remap validation (§6.1): a duplicate-key remap attempt is flagged, not silently saved; an incomplete 13-offset mapping is flagged.
- Fallback to `DEFAULT_KEYBOARD_MAPPING` when `settings.keyboardMapping` is absent.

## 10. Acceptance criteria

- [ ] The default mapping matches §4.1's table exactly, verified against `docs/PRODUCT_SPEC.md` §11.
- [ ] A learner can remap computer-keyboard keys from Settings (`docs/PRODUCT_SPEC.md` §6).
- [ ] Mouse, touch, and computer-keyboard input all produce identical engine/audio behavior for the same note (`docs/PRODUCT_SPEC.md` §29) — for the keyboard specifically, this means the default mapping's resolved `midiNote`s must exactly match the corresponding on-screen keys' own `data-midi-note` values.

## 11. Out of scope

- How the adapter reads this mapping to produce a `NoteEvent` → `docs/MUSIC/INPUT_SYSTEM.md`
- Settings screen UI/interaction design for remapping → not yet specified in this documentation batch
- MIDI input mapping (MIDI notes arrive with their own absolute `midiNote` already, no keyboard-mapping lookup involved) → `docs/MUSIC/INPUT_SYSTEM.md` §4.3

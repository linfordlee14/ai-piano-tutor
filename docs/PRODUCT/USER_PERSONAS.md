# User Personas

## 1. Purpose

Expands `docs/PRODUCT_SPEC.md` §4 (Target Users) into full personas an implementer or content author can design against. Every claim here traces back to §4, or to another spec section explicitly cross-referenced from §4 (§2 north star, §3 problem statement, §9 UX architecture, §17/`TYPES_AND_CONTRACTS.md` for the `skillLevel` field these personas back). No demographic detail (age, income, geography, etc.) is invented beyond what the source spec states — where the spec is silent, this document says so rather than fabricating market-research color.

## 2. Epistemic Status — Read This First

Per `PRODUCT_SPEC.md` §3, the personas below (and the assumption that this product's approach serves them) are **not yet validated with real users**. The spec's own recommendation stands: run 5–10 informal sessions watching actual beginners attempt the first-note flow before or in parallel with the MVP build. Treat every "goal," "frustration," and "context of use" below as the spec's working hypothesis about these users, not confirmed research. This document should be revisited once that validation happens.

## 3. Type System Mapping

`AIMemorySummary.skillLevel` (`TYPES_AND_CONTRACTS.md` §4.8) is a two-value enum: `'beginner' | 'returning'`. It maps directly to the two personas below:

| Persona | `skillLevel` value |
|---|---|
| Primary — the complete beginner | `'beginner'` |
| Secondary — the rusty returner | `'returning'` |

Every prompt template and dashboard element that reads `skillLevel` is, in effect, reading "which of these two personas is this learner."

## 4. Primary Persona — The Complete Beginner

This is the persona **every MVP decision optimizes for** (§4). If a feature decision has to choose between serving this persona and serving anyone else, this persona wins in MVP.

### 4.1 Snapshot

| Attribute | Detail | Source |
|---|---|---|
| Piano access | None | §4 |
| Music-theory background | None | §4 |
| Motivation | Curiosity, or one specific goal (e.g., "I want to play one song for someone") | §4 |
| Devices | Smartphone and/or laptop | §4 |
| Practice pattern | Short, irregular sessions, 5–20 minutes | §4 |
| `skillLevel` | `'beginner'` | `TYPES_AND_CONTRACTS.md` §4.8 |

### 4.2 Goals

- Find out, within minutes, whether learning piano through this app is something they actually want to keep doing — before committing to an account. This is the entire reason the anonymous try-first flow exists (§2, §18).
- Play a first, recognizably correct note or short melody without needing to already know anything (§2's north star: a first-time visitor plays a recognizably correct 5-note melody within their first session, unassisted by a human).
- Build an anchor point on the keyboard (e.g., "where is Middle C") rather than being expected to already orient themselves (§5).
- Get a reason to return tomorrow, without having to plan their own practice (§5: streak/stats visible; one daily practice session generated for them).
- Ask plain-language questions ("why does my chord sound wrong") and get an answer grounded in what they actually just played, not a generic canned response (§5).

### 4.3 Context of Use

- Primarily mobile, primarily one-handed portrait use as the baseline interaction mode (§9) — this persona is assumed to be picking the app up in short bursts, not sitting at a desk with two hands free and a stand.
- Sessions are short and irregular (5–20 minutes, §4) — onboarding friction and load time matter disproportionately, because there's no guarantee of a second attempt if the first one is slow or confusing.
- No existing mental model of the keyboard, notation, or terms like "triad" or "octave" — content and tutor language must not assume prior vocabulary.
- Likely no MIDI hardware, and if on iOS Safari, cannot use MIDI at all even later, since Safari has no Web MIDI support and Apple has not committed to adding it (§11) — this persona's primary and often *only* input method is touch/mouse/computer-keyboard, not a future hardware upgrade path.

### 4.4 Frustrations This Persona Is Assumed To Have (with existing alternatives)

Per §3's problem statement:
- A physical instrument, a human teacher, or a static method book all raise the barrier to *starting* — this persona wants to start today, on the device already in their hand.
- Existing interactive competitors (Simply Piano, Skoove, Flowkey, Yousician) have proven the category, but the spec's working (unvalidated) hypothesis is that this persona specifically wants natural-language Q&A ("why does my chord sound wrong") rather than just more scripted exercises (§3) — this is explicitly flagged as plausible but unverified, not a confirmed pain point.
- A wrong key press treated as a "failure" is assumed to be discouraging enough to matter — hence §5's explicit story that a wrong key press must be treated as a normal part of learning, not a failure state.

### 4.5 What MVP Does Solve For This Persona

- Zero-friction trial: anonymous try-first session, piano usable before any signup screen (PR-AUTH-01, PR-DRILL-05).
- An anchor point onboarding flow ("Learn the Keys" glow-and-confirm for Middle C) rather than an unstructured keyboard (PR-LESSON-02).
- Non-punitive error handling throughout lessons — wrong notes are learning signal, not failure (PR-LESSON-04).
- A structured, linear path from "which key is C" through simple melodies and an introduction to major triads (§7, PR-LESSON-01) — no need to plan their own curriculum.
- A tutor that explains *this specific* mistake, grounded in the exercise they just attempted (PR-AI-01, PR-AI-02) — not generic theory content.
- A deterministic daily practice session so they never face a blank "what do I do now" screen (PR-AI-04).
- A streak and XP as a lightweight reason to return (PR-PROGRESS-01, PR-PROGRESS-02).
- The app keeps working (piano, current lesson) even if Gemini or Firestore is unreachable — this persona is never blocked mid-practice by an infrastructure failure they have no way to diagnose (PR-PLATFORM-03, PR-PLATFORM-05).

### 4.6 What MVP Does Not Solve For This Persona

- Full accessible parity on the *piano-playing* interaction itself, if this persona is also a screen-reader user — an honest, explicit MVP limitation (§24, PR-A11Y-08).
- Validated confidence that mobile-first is actually the right primary form factor for this persona to practice an instrument on — this is an open, unvalidated assumption (§3, §9), not a proven decision. See `docs/UX/RESPONSIVE_DESIGN.md` for the flagged breakpoint/octave-count question this creates.
- A confirmed answer to whether this persona actually wants conversational AI Q&A over more exercises — unverified per §3.
- Anything requiring MIDI, real songs, achievements beyond streak/XP, or offline-first PWA behavior — all explicitly Phase 2+ (§7's deferred table), regardless of how much this persona might want them.
- Acoustic instrument input (playing on a real piano and having the app listen) — a fundamentally different, harder problem, explicitly Phase 3 (§13, §8).

## 5. Secondary Persona — The Rusty Returner

### 5.1 Snapshot

| Attribute | Detail | Source |
|---|---|---|
| Prior experience | Had lessons as a child, or plays another instrument | §4 |
| Desired tone | A refresher, not being talked down to | §4 |
| `skillLevel` | `'returning'` | `TYPES_AND_CONTRACTS.md` §4.8 |

### 5.2 Goals

- Skip material they already know, rather than re-sitting Level 1 content designed for someone who has never touched a keyboard (§4). This is described in the spec as *the main reason a skip-ahead assessment (onboarding skill-check) exists at all* — this persona is the entire justification for that screen existing.
- Be treated as someone with existing musical instinct, even if their piano-specific recall is rusty (e.g., knows what a chord *is* conceptually but may have forgotten which keys form C major).
- Same secondary goals as the primary persona once past onboarding — resume-where-left-off, streak/XP, grounded tutor answers — nothing about the ongoing product experience is persona-specific beyond the starting point and the tutor's tone calibration.

### 5.3 Context of Use

Same device and session-pattern assumptions as the primary persona (§4 does not differentiate device or session length for this persona) — the distinction the spec draws is entirely about **starting point and content pacing**, not platform or usage pattern.

### 5.4 Frustrations This Persona Is Assumed To Have

- Being "talked down to" — content or tutor tone that explains concepts they already understand, or gatekeeps them behind beginner-paced material, is the named frustration (§4).
- By implication (not stated explicitly in §4, flagged here as a reasonable inference rather than a sourced fact): resitting content that provides no new information is a bigger risk of churn for this persona specifically, since they have a baseline to compare the app's pacing against.

### 5.5 What MVP Does Solve For This Persona

- The onboarding / Quick Skill Check screen: 3–4 skippable questions ("Have you played before?", "Do you know any notes already?") used to set a starting lesson index — never gatekeeping (§10). Answering (rather than skipping) this is how this persona avoids Level 1.
- Tutor responses are calibrated to skill level via the `TutorRequest.skillLevel` field, so a `'returning'` learner's tutor answers can differ in register from a `'beginner'` learner's, even from the same prompt template (§15, `TYPES_AND_CONTRACTS.md` §4.9).
- `AIMemorySummary.skillLevel` persists this distinction across sessions, so the app doesn't need to re-ask (`TYPES_AND_CONTRACTS.md` §4.8).

### 5.6 What MVP Does Not Solve For This Persona

- **Open question, flagged rather than resolved:** `PRODUCT_SPEC.md` never specifies the mapping from onboarding answers to a starting lesson index (e.g., which answer combinations skip to which `level`). §10 says the skill-check "sets a starting lesson index," but the actual mapping logic is undefined. Until this is authored, a `'returning'` learner who answers honestly may still land somewhere that doesn't actually match their skill — this is a real gap, not a design decision already made. See the open-questions summary for this batch.
- A skill check deep enough to place this persona *mid-scale-lesson* or *mid-chord-lesson* rather than at a lesson boundary — the spec describes setting a "starting lesson index," implying placement is lesson-granular, not finer-grained.
- Any content this persona might specifically want that a true beginner wouldn't (e.g., faster-paced review, terminology-forward explanations) — the spec does not describe any `'returning'`-specific *content*, only tutor-tone calibration and a different starting point. If distinct returning-learner content is expected, that is not currently specified.

## 6. Explicit Non-Goals / Anti-Personas

Per §4: **"Explicitly not the v1 target (revisit in Phase 3+)."** These are named so that a feature request framed around one of them can be correctly identified as out of MVP scope, not silently accepted as a new implicit persona.

| Anti-persona | Why excluded from v1 | Revisit |
|---|---|---|
| Intermediate/advanced pianists wanting sight-reading or classical repertoire | The product's core loop, content, and tutor calibration are built for someone with zero theory background — sight-reading and repertoire are a different product surface, not an extension of the beginner path | Phase 3+ (§4); sight-reading is explicitly named its own future item, "a different product surface; scope as its own spec when it's time" (§8) |
| Users wanting two-hand advanced technique | MVP lesson content tops out at single-hand melody + intro to major triads (§7); two-hand technique is a materially different skill progression | Phase 4 (§7's deferred table) |
| Users with no internet access at all | Offline is explicitly *resilience* against blips, not the primary mode — the app assumes a network connection exists most of the time, with graceful local-first behavior when it briefly doesn't (§7, §23) | Not on the current roadmap as a stated target; full offline-first PWA (Phase 2, §8) still assumes intermittent, not absent, connectivity |

## 7. Cross-References

- `docs/PRODUCT/USER_STORIES.md` — the epics these personas' goals are drawn from, expanded with acceptance criteria.
- `docs/PRODUCT/USER_FLOWS.md` — flow 1 (first-time anonymous visitor) is written primarily for the primary persona; the onboarding step in flow 1/2 is where the secondary persona's skip-ahead need is served.
- `docs/UX/SCREEN_SPECIFICATIONS.md` — Onboarding / Quick Skill Check screen spec.
- `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8/§4.9 — `AIMemorySummary`, `TutorRequest`.

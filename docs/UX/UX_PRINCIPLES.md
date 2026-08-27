# UX Principles

## 1. Purpose

The interaction philosophy every screen, component, and flow in this product must follow, drawn from `docs/PRODUCT_SPEC.md` §9 (User Experience Architecture), grounded by §2 (Product Vision) and §33 (Architecture Diagram commentary). Three things live here: the core interaction loop, mobile-first principles, and the "learner is never blocked" principle. A screen or component spec that conflicts with this document should be treated as the thing that's wrong, not this document.

## 2. A Note on Sourcing "Learner Never Blocked"

The task that produced this document referred to it as a "section 43-style" principle "referenced in `PRODUCT_SPEC.md` section 2." Worth stating precisely, since it affects how this principle should be cited elsewhere: `PRODUCT_SPEC.md` is a 35-section document, and several of its own sections (§10, §20, §23) refer to "the brief's section 43" — that's a reference to the *original 44-section brief* this spec was written in response to, a document outside this repository, not to a section within `PRODUCT_SPEC.md` itself. There is no section 43 inside `PRODUCT_SPEC.md`. This document therefore cites the principle from where it actually and fully appears inside `PRODUCT_SPEC.md`: §2 (Product Vision, which states it as a named design principle) and §33 (which describes its architectural realization). Flagged as a non-blocking documentation-hygiene note, not a contradiction to resolve.

## 3. The Core Interaction Loop

Per §9, mapped directly to UI:

```
Learn → Demonstrate → Practice → Feedback → Repeat → Progress
```

| Stage | What happens | UI mapping | Grounding |
|---|---|---|---|
| **Learn** | The lesson intro states the goal before anything is expected of the learner. | Lesson Player, intro/instruction area above the piano | §9, §10 (Lesson Player) |
| **Demonstrate** | An animated + audio demo plays on the piano itself — the same instrument the learner will use, not a separate video or diagram. | Lesson Player, `demonstrating` state | §9 |
| **Practice** | The learner plays; keys highlight live as they go. | Lesson Player, `awaitingInput` / active play state; `NoteEvent`s trigger sound and key-visual state (idle/hover/active/correct/incorrect) with zero network dependency | §9, §11 |
| **Feedback** | The deterministic engine's verdict is shown first. An optional "ask Gemini why" is available second — never the other way around. | Lesson Player, `correct` / `incorrectWithHint` state; `ExerciseVerdict` renders before any Gemini call is even offered | §9, §14 |
| **Repeat** | On an incomplete/incorrect attempt, the learner retries with escalating hints. Never a hard stop. | Lesson Player, `incorrectWithHint` → retry | §9 |
| **Progress** | XP/streak update; one "what's next" nudge. | Lesson Player transition → Dashboard, or an inline update within Lesson Player before advancing | §9 |

**Why Feedback is ordered "engine verdict first, Gemini second," structurally, not just as a display convention:** the engine verdict is available the instant the server responds (§22: sound/visuals are local; only the *persisted* result depends on the network, and that round-trip is a single batched write, not per-note). A Gemini call, when the learner asks for one, is a separate, slower, explicitly-user-initiated action layered on top (§14: 1–3 seconds is acceptable there specifically *because* it's optional and gated). If an implementation ever makes the deterministic verdict wait on a Gemini response, that inverts this principle and should be treated as a bug, not a stylistic choice.

## 4. Mobile-First Principles

Reproduced from §9, with the boundary this document adds: these are described in the source spec as *principles*, and the specific numbers attached to them (breakpoints, exact octave counts) are explicitly **not** locked — see `docs/UX/RESPONSIVE_DESIGN.md` for the open question this creates.

1. **The piano is the one fixed, always-visible element.** Everything else — instructions, navigation chrome, feedback text — scrolls or collapses around it. The piano is never what gets pushed off-screen to make room for something else.
2. **Touch targets need ≥44px per key on phones.** This constraint is what drives the "1–1.5 octaves visible at a time with horizontal pan" guidance, rather than squeezing more octaves onto a small screen at the cost of unplayable key sizes. §9 is explicit that this needs an actual prototype/usability check before the layout is locked — treat the specific range as a working hypothesis, not a final spec.
3. **One-handed portrait use is the baseline, not the exception.** Every screen is designed assuming a learner holding a phone in one hand, thumb-reachable. Landscape and desktop are progressive enhancement layered on top of that baseline — not the primary design target that mobile gets squeezed into afterward.

## 5. "The Learner Is Never Blocked"

Stated in §2 as a design principle carried through *every* section of the spec, not a nice-to-have bullet:

> If Gemini is down, the piano still works. If Firestore is down, the piano still works. If MIDI isn't available, the piano still works.

This is why the music engine and the AI layer are architecturally separate (§14) — it is an architectural consequence, not just a UX aspiration layered on top afterward.

### 5.1 What This Looks Like, Concretely

Per §33's architecture-diagram commentary, two properties make this real rather than aspirational:

1. **Input → Engine → Audio/UI is a closed local loop.** No Firebase or Gemini call sits anywhere in the critical path between a key press and the learner hearing/seeing the result. This is what makes "the piano still works" literally true, not just a marketing claim.
2. **Gemini is reachable only through Cloud Functions, and only on explicit user action.** There is no code path where a Gemini outage can block scoring, progress-saving, or lesson advancement — because none of those things ever depended on Gemini succeeding in the first place (§14, §29: "disabling the Gemini Function entirely leaves every other screen fully functional").

### 5.2 UX Consequences This Principle Forces

- **Every degraded/unreachable state needs a designed fallback, not a spinner or a blank screen.** The AI Tutor's "temporarily unavailable, here's what to practice instead" card (§10) and the app-wide degraded-service banner ("Playing offline — your progress will sync later," §10) are the two concrete implementations of this rule already specified. See `docs/UX/SCREEN_SPECIFICATIONS.md` for the full per-screen degraded-state treatment.
- **A wrong key press is never a dead end either.** This is the same principle applied one level down, inside a single exercise: §5's story that a wrong key press is treated as a normal part of learning, not a failure state, and the Repeat stage of the core loop (escalating hints, never a hard stop) are both this same "never blocked" principle, just applied to *pedagogical* blocking rather than *infrastructure* blocking.
- **Optional AI features must degrade without taking anything else down with them.** If a screen's primary value depends on Gemini succeeding, that screen violates this principle. The AI Tutor screen is the one screen where a Gemini failure removes real functionality (the conversational answer itself) — and even there, the fallback still gives the learner something actionable (§34's MVP test explicitly carves this out as the *one* exception: "...except the tutor question itself").

## 6. Cross-References

- `docs/UX/NAVIGATION.md` — how the 5-item nav structure supports one-handed, mobile-first use.
- `docs/UX/SCREEN_SPECIFICATIONS.md` — per-screen loading/error/degraded states that implement §5 of this document concretely.
- `docs/UX/RESPONSIVE_DESIGN.md` — the specific, currently-unvalidated numbers behind §4 of this document.
- `docs/PRODUCT/USER_FLOWS.md` — Flow 4 walks through the degraded-Gemini case step by step.

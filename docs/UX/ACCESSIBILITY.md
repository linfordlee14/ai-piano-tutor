# Accessibility

## 1. Purpose

Expands `docs/PRODUCT_SPEC.md` §24 (Accessibility) in full. This document preserves the source spec's own honest caveat rather than softening or overselling it: **full accessible parity on the piano-playing interaction itself is not achievable in MVP.** Only the surrounding UI is fully accessible. Any future edit to this document that removes or waters down that caveat should be treated as a regression in honesty, not an improvement in ambition — the point of stating it plainly is so no one downstream (a learner, a reviewer, an accessibility auditor) is misled about what's actually been achieved.

## 2. Scope

Every accessibility commitment named in §24, expanded to implementer-ready detail, plus the testing regime from §26 that verifies it. This document is the cross-reference target named throughout `docs/UX/SCREEN_SPECIFICATIONS.md` and `docs/UX/COMPONENT_SPECIFICATIONS.md` — those documents apply these rules per screen/component; this document owns the rules themselves.

## 3. The Honest Caveat, Stated First and Precisely

Reproduced from §24, because getting this exactly right matters more than any other paragraph in this document:

> A screen-reader user cannot practically sight-read a rapidly changing visual keyboard the way a sighted user does — claiming full accessible parity on the *piano-playing interaction itself* would be overselling a genuinely hard, separate research problem.

**What this means concretely:** the moment-to-moment experience of pressing keys and seeing/hearing live visual feedback on the `PianoKeyboard` component (key glow during a demonstration, live highlight during play, the instant of a note sounding) is not, and is not claimed to be, an equivalent experience for a screen-reader user versus a sighted user. This is a real, unsolved research problem — not a corner this team cut. Real-time visual state changing many times per second across dozens of keys does not translate into a screen-reader-consumable form using known, mature techniques; solving it well would be its own significant product effort, not a checklist item.

**What MVP actually commits to instead**, per §24: everything **around** the piano — instructions, navigation, feedback, dashboard, tutor chat — is fully accessible, and the piano supports **full keyboard-only play** (already required independently by the computer-keyboard input story, `docs/PRODUCT/USER_STORIES.md` §9.1). Full keyboard-only play helps non-mouse users generally (including many but not all screen-reader users, and users with motor disabilities who don't use a screen reader at all) — but it does not, by itself, solve the sight-reading problem above. Conflating "keyboard-operable" with "screen-reader-equivalent" would be exactly the overselling §24 warns against.

**Do not claim, in any marketing copy, App Store listing, or support content derived from this spec, that the piano-playing experience itself is screen-reader accessible.** That claim would be false, and this document exists partly to make sure that falseness doesn't happen by accident through copy drift.

## 4. Concrete Commitments (What MVP Does Achieve)

### 4.1 Keyboard Navigation

- Every non-piano control — navigation (5-item nav, `docs/UX/NAVIGATION.md`), the lesson list, buttons, form fields, the tutor chat input and message list — is fully operable via Tab (move focus), Enter/Space (activate), with **visible focus indicators**.
- Focus indicators are never suppressed by a CSS reset (e.g., no blanket `outline: none` without a replacement focus style) — a common, easy-to-introduce regression worth naming explicitly so it doesn't slip in silently during later styling work.
- Focus order follows visual/logical reading order on every screen; a modal or transient element (e.g., a validation error, a toast) does not strand keyboard focus somewhere the learner can't recover from.
- The piano itself supports full keyboard-only play via the computer-keyboard input mapping (`docs/PRODUCT/USER_STORIES.md` §9.1, `KeyboardMapping` type) — a learner can play any note without a mouse or touch input, using the default or a remapped key layout.

### 4.2 Screen Reader Support

- Lesson instructions, exercise feedback, and score results are announced via **ARIA live regions**. Politeness level matters and should not be uniform:
  - Score/feedback results (a meaningful, learner-initiated outcome) — assertive or polite depending on how disruptive an interruption is acceptable at that moment; feedback that follows a deliberate learner action (submitting an attempt) can reasonably be more assertive than passive/ambient updates.
  - Free Play's recognition toasts (an ambient, continuous byproduct of exploration, not a discrete requested result) — **polite**, explicitly, per `docs/UX/SCREEN_SPECIFICATIONS.md` §11 and `docs/UX/COMPONENT_SPECIFICATIONS.md` §9 — an assertive announcement firing on every single key press during free exploration would be actively unpleasant, not helpful.
  - The degraded-service banner — polite, since it's an ambient status notice, not an urgent interrupt (`docs/UX/SCREEN_SPECIFICATIONS.md` §15).
- Piano keys carry accessible names — the explicit example given in §24 is **"C4, white key"** — meaning both the note identity (in a form more useful than a bare MIDI number) and the key's visual/tactile category (white/black) are conveyed, not just one or the other.
- Every component with a `label`/`accessibleLabel`/`accessibleName` prop in `docs/UX/COMPONENT_SPECIFICATIONS.md` (`KeyComponent`, `LessonProgressIndicator`) exists specifically to carry this requirement — those props are not optional decoration.

### 4.3 Color Contrast

- **WCAG AA minimum (4.5:1 for text)** throughout the app — this is a floor, not a target to sometimes miss for a "close enough" brand color.
- Correct/incorrect key feedback, and any other state that currently or in the future might rely on color coding (e.g., the difficulty step-up/step-down indicator on the Note ID game, the locked/unlocked/completed states on Learn), always pairs with a **non-color signal** — icon, shape, or animation — never color alone. §24 states this "exactly as required," meaning this is treated as a hard requirement, not a nice-to-have.
- This rule extends to gamification elements (streak, XP) per the elaboration in `docs/UX/SCREEN_SPECIFICATIONS.md` §6 — a streak badge cannot rely on "green means good" without an accompanying icon/number.

### 4.4 Adjustable Text Size

- Layout respects browser/OS text-size settings — no fixed-pixel text sizing that ignores browser zoom or OS-level text-size preferences.
- Tested explicitly **at 200% browser zoom**: content must remain readable and usable (no clipped text, no overlapping elements, no horizontally-scrolling body text) at that zoom level, on every MVP screen named in `docs/UX/SCREEN_SPECIFICATIONS.md`.

### 4.5 Reduced Motion

- Key-press animations and transitions (the "press/hover animation" on `KeyComponent`, lesson-complete celebration animations, difficulty step-change indicators, etc.) respect `prefers-reduced-motion` — when set, these reduce to instant or minimal-motion equivalents rather than full animated transitions.

## 5. Per-Screen / Per-Component Application

This document owns the rules; `docs/UX/SCREEN_SPECIFICATIONS.md` applies them concretely per screen (see each screen's "Accessibility notes" subsection) and `docs/UX/COMPONENT_SPECIFICATIONS.md` encodes them into component contracts (`accessibleName`, `accessibleLabel`, ARIA-politeness props). The screens where the §3 caveat is most directly load-bearing, worth naming explicitly here:

- **Lesson Player** — the core screen; keyboard-only play is fully supported, but live key-highlight-during-demonstration is not screen-reader-equivalent to watching it.
- **Piano Free Play** — the clearest example: every non-piano element (recognition toasts, nav) is fully accessible; the moment-to-moment "explore the keyboard visually" experience is not claimed as equivalent.
- **Note Identification Game** — the prompt text itself ("Which key is C?") is the accessible content that matters most here and must be reliably announced; the visual keyboard remains subject to the same caveat as elsewhere.

## 6. Testing Regime

Per §26 (Testing Strategy), accessibility is verified two ways, and **neither alone is sufficient**:

1. **Automated axe-core checks in CI**, covering contrast and ARIA basics on all core screens. Run on every relevant change, not just before launch.
2. **At least one manual screen-reader pass before launch**, because — explicitly per §26 — "automated tools catch roughly half of real accessibility issues, and section 24's honest caveat specifically needs a human check." A human pass is the only way to actually evaluate whether the caveat in §3 above is being represented honestly in the shipped experience, not just whether ARIA attributes are technically present.

`docs/PRODUCT/USER_STORIES.md` §11.1's acceptance criteria for the screen-reader story make this two-part testing requirement an explicit, checkable condition of that story's completion — not an aspiration.

## 7. What Is Explicitly Out of Scope for MVP

Stated plainly, consistent with §3's caveat and the broader MVP-scope-discipline instruction governing this whole documentation batch:

- Solving screen-reader-equivalent access to the live, moment-to-moment piano-playing visual experience. This is named in the source spec as "a genuinely hard, separate research problem" — not scheduled into any phase of `docs/PRODUCT/FUTURE_ROADMAP.md`, because the source spec does not commit to a future phase for it either. If this becomes a real product priority later, it needs its own research and design investment, not a checkbox added to an existing phase.
- Any accessibility work tied to features that are themselves Phase 2+ (MIDI input accessibility, a future song player's accessibility, etc.) — out of scope by the same logic that the features themselves are out of scope; see `docs/PRODUCT/MVP_SCOPE.md`.

## 8. Cross-References

- `docs/PRODUCT_SPEC.md` §24, §26 — source of every commitment above.
- `docs/UX/SCREEN_SPECIFICATIONS.md` — per-screen application, including the "Accessibility notes" subsection on every screen.
- `docs/UX/COMPONENT_SPECIFICATIONS.md` — component-level props (`accessibleName`, `accessibleLabel`, ARIA politeness) that implement these rules.
- `docs/PRODUCT/USER_STORIES.md` §11.1 — the screen-reader user story and its acceptance criteria, which this document's §6 directly supports.
- `docs/PRODUCT/PRODUCT_REQUIREMENTS.md` §5.8 (PR-A11Y) — the stable requirement IDs for every commitment in this document, including PR-A11Y-08 for the caveat itself.

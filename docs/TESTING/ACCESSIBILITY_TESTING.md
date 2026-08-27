# Accessibility Testing

## 1. Purpose

Defines automated and manual accessibility testing for AI Piano Tutor, built around one honest, load-bearing distinction the whole product commits to: everything **around** the piano is fully accessible; the piano-playing interaction **itself** does not claim full accessible parity for screen-reader users. This document exists to make sure testing verifies exactly that claim — no more, no less — rather than either overclaiming coverage the product doesn't have or underclaiming and failing to verify what it does commit to.

**Source of truth:** `docs/PRODUCT_SPEC.md` §24 (Accessibility — including the explicit "Honest caveat"), §26 (testing strategy), §29 (acceptance criteria). The full accessibility commitment and its caveat's exact wording is owned by `docs/UX/ACCESSIBILITY.md` (parallel batch) — this document tests against that spec's commitments and should be re-checked against it once it exists; where the two differ, `docs/UX/ACCESSIBILITY.md` is authoritative on *what* is accessible, and this document is authoritative on *how it's verified*.

## 2. Scope

Covers: automated axe-core checks in CI on all core screens, the manual screen-reader pass required before launch, keyboard-only navigation testing of every non-piano control, color-contrast and non-color-correctness-signal verification, reduced-motion and text-zoom behavior. Does not cover: making the piano-playing interaction itself screen-reader-navigable in real time — that is an explicit, named non-goal for MVP (§3 below), not an oversight in this test plan.

## 3. The Honest Caveat, and What It Means for Testing

`docs/PRODUCT_SPEC.md` §24 states: *"a screen-reader user cannot practically sight-read a rapidly changing visual keyboard the way a sighted user does — claiming full accessible parity on the piano-playing interaction itself would be overselling a genuinely hard, separate research problem."* The realistic MVP commitment, and therefore the realistic testing target, is two-layered:

1. **Everything around the piano** — instructions, navigation, feedback, dashboard, tutor chat, lesson lists, buttons, forms — is fully accessible. Test this thoroughly and hold it to the same bar as any other web app (§5–§7 below).
2. **The piano itself supports full keyboard-only play** (already required independently for the computer-keyboard input story, `docs/PRODUCT_SPEC.md` §5) and keys carry accessible names ("C4, white key") for a screen reader to announce **when focused**, one at a time. This is real, testable accessibility support — it is not the same claim as "a screen-reader user can fluently sight-read and play the keyboard the way a sighted user does," and this document's tests must not conflate the two. Test that a screen-reader user *can* Tab to a key, hear its name, and activate it with Enter/Space, producing correct sound and engine behavior. Do not write a test (or accept a bug report) that treats the *absence* of real-time audio-visual sight-reading parity as a defect — that gap is a disclosed, deliberate MVP scope boundary, not a bug.

**A test suite that silently drops this distinction** — either by skipping piano keyboard-navigability entirely (under-testing what's actually committed to) or by asserting the piano must somehow be "fully screen-reader accessible" in the fluent-playing sense (over-testing against a claim the product doesn't make) — is itself a defect in this document. Every accessibility test involving the piano should be traceable to one of the two numbered commitments above.

## 4. Automated Testing — axe-core in CI

- **Runs on every core screen:** Landing, Sign in/Sign up, Onboarding/Skill Check, Dashboard, Learn (Lesson Path), Lesson Player, Practice Hub, Note ID Game, Piano Free Play, AI Tutor, Progress/Stats, Profile/Settings (full list per `docs/PRODUCT_SPEC.md` §10).
- **Runs in CI on every PR** touching `apps/web`, blocking merge on any new violation (contrast, missing ARIA labels/roles, missing form labels, heading-order issues, and the other categories axe-core's default rule set covers).
- **Known and accepted limit, stated explicitly so it's never mistaken for full coverage:** automated tools catch roughly half of real accessibility issues (`docs/PRODUCT_SPEC.md` §26). Axe-core passing on every screen is a necessary floor, not proof of accessibility — §5's manual pass exists specifically to catch the other half, and a green CI badge here must never be treated as "accessibility is done."
- **Piano screen specifically:** axe-core runs against the Lesson Player and Piano Free Play screens too (checking the *surrounding* chrome — instruction text, buttons, the piano container's own labeling), but a clean axe-core pass on these screens is not evidence for or against the deeper real-time-playability question in §3 — axe-core cannot detect that kind of interaction-level gap by design, it only catches static/structural violations.

## 5. Manual Screen-Reader Pass

**At least one manual pass, completed and its issues triaged before launch** (`docs/PRODUCT_SPEC.md` §26, §29) — this is a required, named gate, not optional polish, precisely because axe-core's ~50% real-world coverage means a manual pass is the only way to catch the other half.

- **Screen readers:** at minimum one mobile (VoiceOver on iOS, given the mobile-first §9 principle and iOS Safari's large expected user share per §11) and one desktop (NVDA or VoiceOver on macOS) pass.
- **Walk the full core loop** (`docs/PRODUCT_SPEC.md` §9): Learn → Demonstrate → Practice → Feedback → Repeat → Progress, narrating what the screen reader announces at each step, specifically checking:
  - Lesson instructions and feedback are announced via ARIA live regions as they change (§24) — not silently updated with no announcement.
  - Score/verdict results are announced in a way that communicates the *specific* miss (e.g., "you missed F"), not just a generic pass/fail tone, matching the product's own stated value ("As a learner, I want to know specifically which note I missed... not just 'wrong'" — `docs/PRODUCT_SPEC.md` §5).
  - Every non-piano interactive element (nav, lesson list items, buttons, tutor chat input and send control, quick-suggestion chips) is reachable and operable via the screen reader's normal navigation, with sensible announced names — not generic "button" or "clickable" with no label.
  - The piano itself: confirm individual keys announce a sensible accessible name when focused ("C4, white key") and that Enter/Space activates them correctly (per §3's second commitment) — and separately, explicitly note in the pass's findings that fluent real-time play via screen reader is not being evaluated as a pass/fail criterion, consistent with §3.
- **Issue triage, not just a report:** every finding from the manual pass gets explicitly triaged into "blocks launch" vs. "known limitation, tracked for later" before `docs/ROADMAP/M5_HARDENING.md` is signed off — an untriaged list of findings does not satisfy the acceptance criterion.

## 6. Keyboard-Only Navigation (Non-Piano)

Per `docs/PRODUCT_SPEC.md` §24: every non-piano control is fully operable via Tab/Enter/Space with visible focus indicators, never suppressed by a CSS reset.

- Tab through every core screen (§4's list) using only the keyboard; assert every interactive element receives visible focus in a sensible order, and every action reachable by mouse/touch is also reachable this way.
- Explicitly test that no global CSS reset or component library default has stripped `:focus` outlines without a deliberate, equally-visible replacement.
- This overlaps with, but is distinct from, the piano's own keyboard-playability (§3, §5) — this section covers *navigating to and operating* controls like the "ask the tutor why" button or the hint button; §3/§5 covers *playing notes on* the piano itself via the physical-key-to-note mapping (`docs/PRODUCT_SPEC.md` §11).

## 7. Color Contrast & Non-Color Signals

- **WCAG AA minimum (4.5:1 text contrast)** verified automatically by axe-core (§4) across all core screens; spot-check manually on any custom-designed component axe-core's static analysis might miss (e.g., a gradient background behind text).
- **Correct/incorrect key feedback always pairs with a non-color signal** (icon, shape, animation) — never color alone (§24). This is not fully catchable by axe-core (which checks contrast ratios, not "is meaning conveyed by color alone" in a bespoke game UI) — test this manually: with a grayscale/color-blindness simulation filter applied, confirm correct vs. incorrect key state remains distinguishable.
- **Adjustable text size:** test the app at 200% browser zoom (§24's explicit test point) across at least the Dashboard, Lesson Player, and Tutor screens, confirming layout doesn't break or clip content — this is a manual/exploratory check, not purely automatable.
- **Reduced motion:** with `prefers-reduced-motion` set, confirm key-press animations and screen transitions respect it (reduced or removed, not just slowed) — a manual/exploratory check per affected component.

## 8. Error Handling

- An accessibility violation caught in CI (axe-core) fails the build with a specific rule ID and element reference, not a generic "accessibility check failed" — actionable failures get fixed; vague ones get ignored under deadline pressure.
- A screen-reader pass finding that cannot be immediately fixed is still recorded and triaged (§5), never silently dropped because it wasn't blocking.

## 9. Acceptance Criteria (Restated From `docs/PRODUCT_SPEC.md` §29)

- [ ] Full keyboard-only navigation of every non-piano screen.
- [ ] Automated axe-core check passes on all core screens.
- [ ] One manual screen-reader pass completed and issues triaged before launch.
- [ ] The piano supports full keyboard-only *play* (note-by-note, per its computer-keyboard mapping) — distinct from, and not a substitute for, the two criteria above.

## 10. Out of Scope

- Real-time, fluent screen-reader sight-reading of the visual piano keyboard — an explicit, disclosed non-goal (§3), not a deferred feature with its own milestone. If this is ever pursued, it needs its own research-backed spec, not a testing-document assumption.
- Automated color-blindness simulation as a CI gate — treated as a manual/exploratory check (§7) given the bespoke nature of the piano UI's correctness signaling.
- Accessibility testing of Phase 2+ features (MIDI setup flows, song learning module, admin CMS) — out of scope for every M0–M5 milestone (`docs/PRODUCT_SPEC.md` §7) and therefore out of scope here too.

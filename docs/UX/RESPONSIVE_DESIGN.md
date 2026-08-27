# Responsive Design

## 1. Purpose

Expands the mobile-first principles in `docs/PRODUCT_SPEC.md` §9 into concrete breakpoint guidance. **This document's central caveat, stated up front rather than buried:** the exact breakpoints and the "how many octaves fit on a phone" question are **UNVALIDATED** and need a usability check before being locked, per §9's own language ("this needs an actual prototype/usability check before locking the layout") and §3's recommendation to run 5–10 informal beginner sessions before over-investing in the mobile layout. Everything numeric in this document is a **starting proposal for implementation to begin from**, not a ratified spec. Do not treat any pixel value below as load-bearing without that usability check having happened.

## 2. Source Citation Note

The task that produced this document pointed to "`PRODUCT_SPEC.md` section 23" for the touch-target/octave-range bullet. That content is actually in **§9** (User Experience Architecture, "Mobile-first principles") — §23 is Offline Strategy and does not contain this material. This document cites §9 throughout, and this discrepancy is flagged as a non-blocking documentation note in the batch summary rather than propagated silently into a wrong citation here.

## 3. What `PRODUCT_SPEC.md` Actually Commits To

From §9, verbatim in substance:
- The piano is the one fixed, always-visible element; everything else scrolls or collapses around it.
- Touch targets need **≥44px per key on phones**.
- This "likely means showing **1–1.5 octaves at a time** with horizontal pan rather than squeezing more octaves onto a small screen" — the spec's own wording is "likely," not "must," precisely because it hasn't been validated.
- Every screen assumes **one-handed portrait use as the baseline**, landscape/desktop as progressive enhancement, not the reverse.

Everything else below is this document's proposed operationalization of those principles into actual breakpoints — informed by the stated constraints, not contradicting them, but going further than the source spec commits to.

## 4. Proposed Breakpoints

Using standard, widely-recognized breakpoint conventions (consistent with Tailwind CSS's default scale, since `docs/PRODUCT_SPEC.md` §32 names Tailwind as the styling choice) as a reasonable, low-risk starting point:

| Tier | Viewport width | Primary input assumption | Nav pattern |
|---|---|---|---|
| Mobile | < 640px | Touch, one-handed portrait | Bottom tab bar (§9) |
| Tablet | 640px – 1023px | Touch or mouse, two-handed likely | Bottom tab bar or left rail — either is defensible; not specified in `PRODUCT_SPEC.md` |
| Desktop | ≥ 1024px | Mouse/trackpad + keyboard | Left rail (§9) |

**Status: proposed, not locked.** These three tiers and their pixel boundaries are not present anywhere in `PRODUCT_SPEC.md` — they are this document's attempt to turn "mobile-first... landscape/desktop as progressive enhancement" into something a CSS implementation can target immediately, while remaining easy to revise once real usability data exists.

## 5. Proposed Piano Visible-Range Behavior Per Tier

| Tier | Visible octave range | Pan behavior | Rationale |
|---|---|---|---|
| Mobile | 1–1.5 octaves | Horizontal pan/scroll to reach the rest | Directly from §9's own "likely" guidance; keeps every key ≥44px |
| Tablet | ~2 octaves | Pan still likely needed for full range, less often | Interpolated — not stated in `PRODUCT_SPEC.md`; a reasonable middle point between mobile's constrained view and desktop's fuller view |
| Desktop | Full MVP-relevant range (~2–3 octaves per §6/§7's "adjustable visible octave range... ~2–3 octave visible range") visible without pan for typical lesson content | Pan only needed if content ever exceeds ~3 octaves | §7 states "~2–3 octave visible range with pan" as the general MVP spec without tying it to a specific viewport tier — this document assumes that range applies most naturally at desktop width, where it fits without needing 44px-driven key shrinkage |

**This entire table is the "how many octaves fit" open question the task instructions asked to be flagged.** Flag severity: **IMPORTANT** — it directly affects `PianoKeyboard`'s implementation (`docs/UX/COMPONENT_SPECIFICATIONS.md` §3's `visibleOctaveRange` prop) and cannot be fully finalized without the usability check §3/§9 call for.

## 6. Touch Target Sizing

- **Minimum 44px per key on any touch-capable viewport** (§9) — this is a hard floor, not a mobile-only nicety; a tablet in touch mode should hold the same floor.
- On desktop with mouse/trackpad input, the 44px floor is not required by the source spec (mouse precision doesn't need the same target size touch does), but shrinking keys dramatically below it purely to fit more octaves should still be weighed against general usability, not just the letter of the touch-target rule.
- `KeyComponent`'s `touchTargetMinPx` prop (`docs/UX/COMPONENT_SPECIFICATIONS.md` §4) should read this floor from a shared responsive-config source, not be hardcoded per breakpoint independently in multiple places.

## 7. Navigation Responsive Behavior

Per §9 and `docs/UX/NAVIGATION.md`: bottom tab bar on mobile, left rail on desktop. This document's addition: treat the mobile→desktop nav switch as happening at the same breakpoint boundary as the general mobile/desktop split (proposed at 1024px above), not a separate, independently-tuned breakpoint — having the nav pattern and the piano layout pattern flip at different widths would create an inconsistent, jarring mid-range experience.

## 8. One-Handed Portrait Baseline

Per §9: every screen assumes one-handed portrait use as the baseline. Concretely, this means:
- Primary actions (CTAs, the "ask the tutor why" entry point, hint button) should sit within comfortable thumb reach in the lower two-thirds of a mobile portrait viewport, not require a reach to the top of the screen.
- Landscape mobile is progressive enhancement, not a redesign — the same component set should simply reflow, not present fundamentally different functionality.
- This baseline directly conflicts, in principle, with the *unvalidated* assumption named in §3 of `PRODUCT_SPEC.md` that mobile-first is even the right primary form factor for practicing an instrument — see §9 below for how this document carries that tension forward rather than resolving it.

## 9. The Deeper Open Question This Document Inherits

Per `PRODUCT_SPEC.md` §3: *"That mobile-first is the right primary form factor for practicing an instrument that benefits from a wider screen — worth a short usability check before over-investing in the mobile layout."* This is a distinct, more fundamental open question than the breakpoint-pixel-values question in §5 above: it's not "what exact octave count fits on a phone," it's "should phone-sized viewports be the primary design target for this product at all." This document, and the rest of this docs batch, follows `PRODUCT_SPEC.md`'s own instruction to build mobile-first anyway (since that's the committed MVP direction per §7/§9), while explicitly carrying forward the spec's own flag that this hasn't been validated. **Flag severity: IMPORTANT**, matching the source spec's own framing — this isn't a documentation gap, it's an acknowledged unvalidated product assumption that this document is not in a position to resolve.

## 10. Cross-References

- `docs/PRODUCT_SPEC.md` §3, §9 — source of every constraint and caveat above.
- `docs/UX/UX_PRINCIPLES.md` §4 — the mobile-first principles this document operationalizes.
- `docs/UX/COMPONENT_SPECIFICATIONS.md` §3 — `PianoKeyboard`'s `visibleOctaveRange` and `touchTargetMinPx` props, which this document's numbers feed directly into.
- `docs/UX/SCREEN_SPECIFICATIONS.md` §8, §11 — Lesson Player and Piano Free Play, the two screens where mobile-vs-desktop piano behavior differs most.

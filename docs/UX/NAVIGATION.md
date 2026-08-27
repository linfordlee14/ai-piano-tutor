# Navigation

## 1. Purpose

Defines the top-level navigation structure, drawn from `docs/PRODUCT_SPEC.md` §9 (User Experience Architecture). Explains not just what the 5-item structure is, but why it is deliberately flatter than a naive reading of the original brief would produce — that reasoning is load-bearing for any future request to "just add one more tab."

## 2. The 5-Item Structure

| Tab | Contains | Platform placement |
|---|---|---|
| **Home** | Dashboard — greeting, streak, XP, "Continue lesson" CTA, today's practice card, weak-area callout, quick links to Piano/Tutor | Bottom tab bar (mobile) / left rail (desktop) |
| **Learn** | Lesson Path — linear list/tree of levels with locked/unlocked/completed states | Same |
| **Piano** | Free Play — full piano, live note/chord recognition, zero scaffolding | Same |
| **Tutor** | Gemini chat | Same |
| **Profile** | Settings, account info — **and** Progress/Stats, which lives here rather than as its own top-level tab | Same |

Source: §9.

## 3. Why It's Flatter Than a Naive Reading of the Brief

A literal reading of the original brief's feature list (piano, lessons, practice drills, AI tutor, progress/stats, profile/settings) suggests six or seven top-level destinations. §9 deliberately collapses this to five, for one stated reason: **"a 7-item mobile bottom nav is a usability tax with no offsetting benefit at this scope."**

Two features get folded into existing tabs rather than getting their own:

- **Practice** (Note ID game, scale drill, chord drill, free play — the Practice Hub screen, §10) lives *inside* Home/Learn, not as a sixth tab.
- **Progress** (accuracy over time, weak notes/chords, lessons completed, practice minutes — the Progress/Stats screen, §10) lives inside Profile, not as a sixth or seventh tab.

This is a direct, explicit design decision recorded in §9, not an oversight — if a future contributor proposes adding "Practice" or "Progress" back as top-level tabs, that is a reopening of this decision, not a bug fix, and should be evaluated against the same usability-tax reasoning that produced the five-item structure in the first place.

## 4. Open Question: Where Exactly Does Practice Hub's Entry Point Live?

**Flagged as an open question, not silently resolved.** §9 states that Practice "lives inside Home/Learn," but neither §9 nor §10 wires an explicit entry point for it:

- §10's Dashboard (Home) description lists its quick links explicitly as "quick links to **Piano/Tutor**" — Practice Hub is not named among them.
- §10's Learn (Lesson Path) description is "Linear list/tree of levels with locked/unlocked/completed states" — no mention of a Practice Hub entry point either.
- §10 does give Practice Hub its own full screen description ("Entry points to Note ID game, scale drill, chord drill, and free play... this is where a learner goes to drill, not to be taught something new") — so the screen itself is fully specified, but *how a learner navigates to it* is not.

**This document's working assumption**, used consistently in `docs/UX/SCREEN_SPECIFICATIONS.md`: Practice Hub is reached via a persistent, prominent entry point at the top of the **Learn** tab (e.g., a "Practice" button/segment above or alongside the Lesson Path list), since §9 names Learn — not Home — first when describing where Practice lives ("Practice and Progress live *inside* Home/Learn"), and Practice Hub is conceptually a sibling of structured lessons (drilling material already taught) rather than a dashboard summary widget. This is a reasonable placement consistent with the stated principle, but it is **an assumption filled in by this document, not a decision already made in `PRODUCT_SPEC.md`.** Flag severity: **IMPORTANT** — it affects concrete IA/navigation implementation and should be confirmed (or corrected) before the Learn and Practice Hub screens are built.

## 5. Screens That Exist But Are Not Top-Level Nav Destinations

For clarity, since §10 documents these as full screens with their own specs but they are deliberately not tabs:

| Screen | Reached from |
|---|---|
| Landing (pre-auth) | Not part of the 5-item nav at all — it's the pre-auth entry surface, replaced by Home once a session (anonymous or real) exists |
| Sign in / Sign up | Reached from Landing, or from Profile for an anonymous session wanting to upgrade |
| Onboarding / Quick Skill Check | A one-time (skippable) flow, not a persistent nav destination |
| Lesson Player | Reached from Learn (tapping an unlocked lesson) or Home's "Continue lesson" CTA |
| Practice Hub | Reached from within Learn — see the open question in §4 above |
| Note Identification Game | Reached from Practice Hub |
| Progress / Stats | Reached from Profile |
| Error / Degraded-service states | Not a screen — a banner/card pattern overlaid on any screen, see `docs/UX/SCREEN_SPECIFICATIONS.md` |

## 6. Cross-References

- `docs/UX/SCREEN_SPECIFICATIONS.md` — full per-screen specs, including entry points for every screen listed in §5 above.
- `docs/UX/UX_PRINCIPLES.md` — the mobile-first principles this flatter structure is in service of.
- `docs/UX/RESPONSIVE_DESIGN.md` — how the nav itself changes shape between mobile (bottom tab bar) and desktop (left rail).

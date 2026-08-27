# Definition of Done

## 1. Purpose

One reusable, cross-milestone checklist. Every milestone in `docs/ROADMAP/` (M0 through M5) is checked against this same bar, in addition to its own specific acceptance criteria — this document exists so that bar is stated once, precisely, instead of drifting into six slightly different versions across the individual milestone files. An `M*.md` file references this document rather than repeating its content.

**Source of truth:** this checklist synthesizes the cross-cutting requirements stated throughout `docs/PRODUCT_SPEC.md` — §7 (MVP scope discipline), §14 (the AI/deterministic boundary), §19 (security architecture), §26 (testing strategy), §29 (acceptance criteria) — into one operational gate.

## 2. Scope

Applies to every milestone, M0–M5, without exception. A milestone is not "done" because its own named acceptance criteria (stated in its own `M*.md` file) pass — it is done when *both* its own criteria *and* every applicable item below hold. Some items apply only from certain milestones onward (e.g., "no Gemini credential in the bundle" is vacuously true before M4 introduces any Gemini call at all, but must actively hold from M4 forward) — each milestone's own file notes which items are live for it.

## 3. The Checklist

**1. Tests pass.**
Every test suite named as required for this milestone in `docs/TESTING/TESTING_STRATEGY.md` §5 (Test Suite Map) passes in CI — not "passes locally" or "passed last time someone ran it." A milestone with a red or skipped required suite is not done, regardless of how the feature looks manually.

**2. This milestone's specific acceptance criteria are met.**
Every checkbox in the milestone's own `M*.md` "Acceptance Criteria" section is verifiably true, traced back to its source in `docs/PRODUCT_SPEC.md` §29 where one exists. "Verifiably" means demonstrated by a passing automated test or an explicitly recorded manual check (e.g. the Android latency spot-check, the screen-reader pass) — not asserted from memory.

**3. No Phase-2+ feature leaked in.**
The milestone's own "Out of Scope" list (every `M*.md` file has one) is actually respected in the shipped code — MIDI input, minor/7th/diminished/augmented chords and inversions as lesson content, additional scales beyond the MVP set, the song learning module, Gemini-personalized daily practice, achievements beyond streak/XP, offline PWA caching, microphone input, and admin CMS tooling do not appear anywhere in M0–M5, per `docs/PRODUCT_SPEC.md` §7's explicit deferred-features table. A feature being small, easy, or "basically free to add while I'm in here" is not an exception — `docs/PRODUCT_SPEC.md` §30 names scope creep from the brief's own comprehensiveness as a named risk specifically because "small, easy" additions are how it happens.

**4. Documentation updated if architecture changed.**
If implementing this milestone required deviating from what a `docs/` specification said — a type shape needed a field no one anticipated, a security rule needed a different mechanism than sketched, an API contract shifted — the relevant specification document is updated (or, if the deviation resolves a conflict between two documents, recorded in `docs/DECISIONS.md` per `docs/ROADMAP/DEVELOPMENT_METHODOLOGY.md` §4) as part of the same milestone's completion, not left for someone to notice later that the code and the docs disagree.

**5. Security rules were not weakened to make a test pass.**
If a test was failing because a Firestore Security Rule correctly denied something, the fix is to fix the code calling it — never to loosen the rule so the test goes green. This applies with special force to the two rules `docs/PRODUCT_SPEC.md` §26 calls out as the ones "where a silent regression would become a real cheating or content-integrity bug": a client cannot write its own score/progress/AI-memory data, and a client cannot write `lessons`. Any change that touches `firestore.rules` in a milestone's diff gets explicit scrutiny against `docs/TESTING/SECURITY_TESTING.md` before merge, not a rubber stamp because "the tests pass now."

**6. No secrets exposed.**
The Gemini API credential (from M4 onward) never appears in client code, a public repo, or a `NEXT_PUBLIC_*` environment variable — verified by the CI bundle-grep check (`docs/TESTING/SECURITY_TESTING.md` §7), not a code-review habit. `.env.local` and any equivalent local secret file stay gitignored. This item is vacuously satisfied for M0–M3 (no Gemini credential exists yet to leak) but becomes live and must be actively checked from M4 onward.

## 4. Applying This Checklist Without Turning It Into a Rubber Stamp

This checklist is a floor, not a substitute for judgment. Two failure modes to watch for, in opposite directions:

- **Checking the box without the substance.** "Tests pass" because the test was written to assert something trivial, or "acceptance criteria met" because a criterion was quietly reworded to something easier to satisfy, defeats the purpose. Each item above should be traceable to a real, specific piece of evidence (a CI run, a named test, a dated manual-check record).
- **Gold-plating past the milestone's actual scope in the name of "done."** This checklist does not ask a milestone to be perfect or complete beyond what its own spec defines — item 3 exists specifically to prevent the opposite failure, where "let's just also handle X while we're here" quietly expands scope. Meeting this checklist means the milestone did exactly what it was supposed to do, verifiably — not more, not less.

## 5. Out of Scope

- Milestone-specific acceptance criteria themselves — those live in each `M*.md` file, not here (this document is the shared *additional* bar, not a replacement for milestone-specific detail).
- Release/deployment sign-off procedures beyond what CI already gates (e.g. a manual go/no-go meeting) — that's a process concern for whoever owns launch operations, not a documentation-repository concern.

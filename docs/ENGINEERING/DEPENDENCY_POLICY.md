# Dependency Policy

## 1. Purpose

The standing policy that governs adding any new dependency to any workspace package in this repository — not a catalog of current dependencies (that's `docs/ENGINEERING/TECH_STACK.md`), but the rule that produced that catalog and that governs anything added after it.

The policy itself is not new — it's `docs/PRODUCT_SPEC.md`'s own stated principle, applied consistently: "every dependency needs a reason" (§1, §32 — the exact phrase §32 uses to justify the deliberately-excluded list: "Each would add a dependency without a corresponding need at this stage... every dependency needs a reason, and none of these currently have one"). This document exists so that principle is a checkable rule, not a one-time judgment call the original stack happened to apply.

## 2. Scope

Applies to any new npm package added to any `package.json` in this monorepo (`apps/web`, `packages/music-engine`, `packages/shared-types`, `functions/`). Does not re-litigate `docs/ENGINEERING/TECH_STACK.md`'s current list — this policy governs what gets added *from here forward*.

## 3. Requirements

**Every new dependency requires a stated justification, recorded in the PR that adds it, answering three questions:**

1. **What problem does this solve** that nothing already in `docs/ENGINEERING/TECH_STACK.md` solves?
2. **Why not build it in-house?** In-house is the default assumption at this stage unless the dependency saves meaningfully more engineering time than its ongoing maintenance and security-surface cost. (This is exactly the reasoning §12 already applied to adopt Tone.js: hand-rolling Web Audio scheduling and envelopes was assessed as high effort for low benefit — the dependency won on the merits, not by default.)
3. **Why not something already in the stack** that could plausibly cover this need? (E.g., don't add a second state-management library when Zustand already covers client-only state and Firestore's own SDK cache already covers server state, §21 — the existing stack should be the first thing checked, not an afterthought.)

**Higher burden of proof for `packages/music-engine`.** Per `docs/ENGINEERING/CODING_STANDARDS.md`'s zero-framework-import rule and `docs/ENGINEERING/PROJECT_STRUCTURE.md` §4's explanation of why that package exists as a physical boundary, any dependency proposed for `packages/music-engine` beyond `packages/shared-types` should be treated as a design smell first and an ordinary dependency decision second. The entire point of that package is staying small, pure, and framework-independent — a "well-justified" dependency there still erodes the property the package exists to guarantee.

**Baseline hygiene applies regardless of justification.** `docs/PRODUCT_SPEC.md` §19: dependency scanning (`npm audit`/Dependabot) runs as a matter of course. A dependency that was well-justified at add-time can still develop a vulnerability later — the justification requirement above is a gate at add-time, not a substitute for ongoing scanning.

## 4. The Excluded Set — This Policy Already Applied

`docs/ENGINEERING/TECH_STACK.md` §4 has the full reasoning per item; this is the short version, as a worked example of the policy rather than a second catalog:

| Excluded | The policy's three questions, answered briefly |
|---|---|
| Redux | Zustand already solves the one need (state outside React's render cycle); no gap Redux fills |
| GraphQL | No multi-client API problem exists; Firestore SDK + typed callables already fit the one client's needs |
| A separate REST API | Callable Cloud Functions already solve auth/CORS boilerplate; a REST layer reintroduces it |
| A third-party analytics SDK | Firebase Analytics + BigQuery already cover the MVP metric set |
| Firebase Storage for public assets | The CDN already serves static files; Storage adds an auth-rule layer with no benefit here |
| Any CMS for MVP | ~12–15 static JSON files don't have a volume problem yet; a CMS solves one that doesn't exist |

## 5. Process

State the three-question justification in the PR description that adds the dependency (`docs/ENGINEERING/CODING_STANDARDS.md` §3.5 owns PR conventions generally). No separate approval workflow is specified beyond normal PR review — this policy's job is to make sure the *reasoning* is written down and reviewable, not to add process for its own sake.

## 6. Out of Scope

- The current dependency list and its per-item reasoning — `docs/ENGINEERING/TECH_STACK.md`.
- The specific import-boundary enforcement mechanism for `packages/music-engine` — `docs/ENGINEERING/CODING_STANDARDS.md`.
- Lesson-content authoring tools (not an npm dependency) — `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`.

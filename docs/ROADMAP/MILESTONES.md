# Milestones — Master Index

## 1. Purpose

The master overview of AI Piano Tutor's six MVP milestones (M0–M5). This document reproduces the milestone table from `docs/PRODUCT_SPEC.md` §28 and gives each milestone a one-paragraph orientation, but **the individual milestone spec file is authoritative over this summary if the two ever differ in detail.** This file is a map; the `M*.md` files are the territory.

**Source of truth:** `docs/PRODUCT_SPEC.md` §28 (Development Milestones), §29 (Acceptance Criteria), §7 (MVP Scope), §34 (Final MVP Definition). Methodology governing how these milestones are meant to be built: `docs/ROADMAP/DEVELOPMENT_METHODOLOGY.md`. Shared completion bar: `docs/ROADMAP/DEFINITION_OF_DONE.md`.

## 2. Scope

Covers: the six-milestone index, each milestone's one-line objective/acceptance/dependency, the overall dependency graph, and pointers to each milestone's detail file. Does not cover: the detailed scope, file/component list, or full acceptance criteria for any individual milestone — those live in the corresponding `M*.md` file, one per milestone, all in this same directory.

## 3. The Milestone Table (Reproduced From `docs/PRODUCT_SPEC.md` §28)

| # | Milestone | Objective | Acceptance | Depends on | Detail file |
|---|---|---|---|---|---|
| M0 | Foundations | Prove the toolchain before any feature exists | An anonymous user hits a deployed URL and sees an empty dashboard shell; CI deploys on push; security rules default-deny from commit one | — | `M0_FOUNDATIONS.md` |
| M1 | Piano that makes sound | The instrument works standalone before any lesson wraps around it | A learner can play any of the 12 major triads and see it correctly recognized; `music-engine` unit tests pass | M0 | `M1_MUSIC_ENGINE.md` |
| M2 | First guided lesson, end to end | Prove the full loop for one lesson before authoring the rest | The canonical E2E flow (open lesson → identify note → press key → get result → progress saved) passes as an automated Playwright test | M1 | `M2_FIRST_LESSON.md` |
| M3 | Full MVP content + Note ID game | The app is a complete beginner course, not a tech demo | A learner can progress linearly from Level 1 through the intro-to-triads lesson with no missing or broken step | M2 | `M3_MVP_CURRICULUM.md` |
| M4 | AI tutor | The AI layer, built last and on top, exactly as `docs/PRODUCT_SPEC.md` §14 architects it | The missed-F worked example produces a correct, on-tone response; disabling Gemini entirely breaks nothing else | M2 | `M4_GEMINI_TUTOR.md` |
| M5 | Hardening & launch readiness | Turn a working demo into something safe to point real users at | Full `docs/PRODUCT_SPEC.md` §29 acceptance list passes: accessibility pass, security rules tests, account deletion/export, analytics wired, perf budget met, sample attribution credited | M1–M4 | `M5_HARDENING.md` |

## 4. Reading the Dependency Graph Correctly

```
        M0
         │
        M1
         │
        M2
        ╱  ╲
      M3    M4
        ╲  ╱
        M5
```

**M2 is the fork point, and this is deliberate, not an oversight.** Per the table's own "Depends on" column: M3 depends on M2, and M4 *also* depends on M2 — **not on M3.** `docs/PRODUCT_SPEC.md` §28's own dependency table is unambiguous here, and the reasoning is explicit in the M4 row's objective ("built last and on top") combined with what M2 actually delivers: one real, scored, Firestore-persisted lesson. M4 (the AI tutor) needs real scored exercises to give feedback and build tutor context on — that's what M2 provides — not the full ~12–15 lesson curriculum, which is M3's job. M3 and M4 are therefore **independent branches that both build on M2** and can proceed in either order, or in parallel, once M2's acceptance criteria are met. M5 is the join point: it depends on **both** M3 and M4 being complete (per the table's `M1–M4` dependency), so in practice the full curriculum and the AI tutor both ship before hardening/launch — but neither blocks the other's start.

**Read `docs/PRODUCT_SPEC.md` §14's "deterministic engine before AI" principle at the right grain.** The non-negotiable rule — Gemini never determines correctness, and the app must be a complete, working, testable product on deterministic logic alone before Gemini is added — is fully satisfied by M4 depending on M2 (a real, working, deterministically-scored lesson already exists and is provable end-to-end before any Gemini call is wired in). It does **not** require M4 to additionally wait on M3's full curriculum — M2 alone already proves the deterministic core works. See `docs/ROADMAP/M4_GEMINI_TUTOR.md` §2 for the full statement of this prerequisite and why it's M2, specifically, not M3.

## 5. One-Paragraph Orientation Per Milestone

- **M0 — Foundations:** an empty deployed shell with CI/CD and default-deny security rules already in place, before a single feature exists. Proves the toolchain, not the product. See `M0_FOUNDATIONS.md`.
- **M1 — Piano that makes sound** *(this cluster's file is named `M1_MUSIC_ENGINE.md` — see that file's own note on why the filename and the milestone's actual scope differ slightly)*: the virtual piano — Tone.js audio, mouse/touch/keyboard input, Free Play with live note/chord recognition — works completely standalone, unit-tested via `packages/music-engine`, before any lesson wraps around it. See `M1_MUSIC_ENGINE.md`.
- **M2 — First guided lesson, end to end:** one real lesson ("Find Middle C"), the Lesson Player, server-side scoring via a Cloud Function, and a dashboard that shows completion — proving the full loop once before authoring the rest of the curriculum. See `M2_FIRST_LESSON.md`.
- **M3 — Full MVP content + Note ID game:** the remaining ~11–14 lessons, the Note Identification game with adaptive difficulty, and streak/XP logic — turning a one-lesson tech demo into a complete beginner course. See `M3_MVP_CURRICULUM.md`.
- **M4 — AI tutor:** the Cloud Functions Gemini proxy, App Check + rate limiting, the Exercise Feedback and Tutor Q&A prompt templates, the AI memory document, the Tutor chat screen, and the degraded-service fallback — built last and strictly on top of the deterministic core. See `M4_GEMINI_TUTOR.md`.
- **M5 — Hardening & launch readiness:** the accessibility pass, full security rules test suite, account deletion/export, analytics wiring, the performance budget, and sample attribution — turning a working demo into something safe to point real users at. See `M5_HARDENING.md`.

## 6. Cross-Cutting References Every Milestone Shares

- **Definition of Done:** `docs/ROADMAP/DEFINITION_OF_DONE.md` — every milestone is checked against this shared bar in addition to its own specific acceptance criteria.
- **Testing:** `docs/TESTING/TESTING_STRATEGY.md` and its category files — each milestone's "Tests Required" section names the specific suites that gate it.
- **Types:** `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — every milestone's data-shape references trace back to this single canonical source.
- **Methodology:** `docs/ROADMAP/DEVELOPMENT_METHODOLOGY.md` — the document hierarchy and conflict-resolution rule governing how each milestone spec was derived from `docs/PRODUCT_SPEC.md`.

## 7. Out of Scope

- Any milestone or feature beyond M5 — Phase 2, 3, and 4 items (`docs/PRODUCT_SPEC.md` §8) have no milestone number and no detail file in this directory. They are future roadmap, not a deferred M6.
- Re-litigating `docs/PRODUCT_SPEC.md` §28's dependency table — this document reproduces and clarifies it; it does not modify it. A change to the dependency structure itself would be a `docs/DECISIONS.md` entry, not an edit made silently in this file.

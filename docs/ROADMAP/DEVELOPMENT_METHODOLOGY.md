# Development Methodology — Spec-Driven Development

## 1. Purpose

States the working method this entire `docs/` repository is an instance of, not just a description of. AI Piano Tutor is built Spec-Driven: the specification is written and internally reconciled *before* implementation, implementation proceeds one milestone at a time against that specification, and any conflict discovered along the way is resolved by editing the specification (and recording the resolution), never by an implementer silently picking whichever reading is convenient.

**Source of truth:** `docs/PRODUCT_SPEC.md` (the root document — see its own framing in the document header: *"This document defines the product before any code is written, per Spec-Driven Development. It is meant to be handed to a coding agent... as the implementation brief."*), §35 (Build Instructions — the concrete recommended build sequence), §28 (Development Milestones).

## 2. Scope

Covers: the document hierarchy and its read/write order, the rule for what happens when two documents disagree, the rule for what happens when a needed decision hasn't been made yet, and how implementation work should be sequenced against this hierarchy. Does not cover: the content of any individual specification document (each owns its own subject matter) or project management mechanics (ticketing, sprint cadence) beyond what `docs/ROADMAP/MILESTONES.md` and the `M*.md` files define.

## 3. The Document Hierarchy

Specification flows in one direction — each layer is written to be consumed by the layer below it, and no layer should need to reverse-engineer intent from the implementation:

```
1. docs/PRODUCT_SPEC.md
   The single root document. States the product, its scope, and every major
   architectural decision, including the ones the original brief got changed
   or pushed back on. Nothing below this layer introduces a NEW product
   decision — it elaborates, makes concrete, or resolves ambiguity within
   what this document already established.
        │
        ▼
2. Architecture & domain specs
   docs/ARCHITECTURE/, docs/MUSIC/, docs/AI/, docs/DATA/, docs/SECURITY/,
   docs/UX/, docs/CONTENT/ — each takes one architectural area PRODUCT_SPEC.md
   introduces at a survey level (e.g. §13's Music Theory Engine, §15's AI
   Prompt Architecture) and specifies it in full implementation detail.
        │
        ▼
3. Engineering contracts
   docs/ENGINEERING/ — TYPES_AND_CONTRACTS.md (canonical shared data shapes),
   PROJECT_STRUCTURE.md (the concrete repo layout), PERFORMANCE_REQUIREMENTS.md,
   CODING_STANDARDS.md. These pin down the exact, cross-cutting technical
   contracts every other layer must agree with byte-for-byte — this is the
   layer where "the docs agree in prose" becomes "the docs agree on a
   TypeScript interface everyone imports from the same package."
        │
        ▼
4. Milestone specs & cross-cutting test specs
   docs/ROADMAP/ (this directory) and docs/TESTING/ — translate the above
   into an ordered, buildable sequence (docs/ROADMAP/MILESTONES.md and its
   M0–M5 detail files) and the testing bar every milestone must clear
   (docs/TESTING/TESTING_STRATEGY.md and its category files). Milestone
   specs are downstream consumers of layers 1–3, not a place where new
   scope gets decided.
        │
        ▼
5. Implementation
   apps/, packages/, functions/, content/ — actual code, written against
   layer 4's milestone specs, which are themselves grounded in layers 1–3.
```

**The rule that makes this real, not aspirational:** a coding agent starting work on any milestone reads layer 1 first, then the specific layer-2/3 documents that milestone's spec names, then the milestone spec itself — never the reverse, and never implementation-first with specs retrofitted afterward. `docs/PRODUCT_SPEC.md` §35 states this explicitly for the build-tooling sequence too: scaffold, then hand precision work to a coding agent milestone by milestone, "not the whole spec at once... [handing] all 35 sections at once and asking it to build everything produces a shallow implementation of everything instead of a solid one of the MVP."

## 4. Source-of-Truth Hierarchy When Documents Conflict

This is the rule every other document in this repository depends on, and it is stated here once, precisely, for all of them to reference:

> **If any two documents conflict, `docs/PRODUCT_SPEC.md` is authoritative — unless the conflict has been explicitly identified and resolved in `docs/DECISIONS.md`.**

`docs/DECISIONS.md` (owned and maintained separately by the project lead, not by this milestone-documentation batch) is the *only* mechanism by which a lower-layer document is allowed to override something `docs/PRODUCT_SPEC.md` states. A lower-layer document cannot simply assert a different answer and expect it to hold — it must be recorded as a deliberate decision, with the conflict named, in `docs/DECISIONS.md`, before it's treated as settled.

**What this means in practice for a coding agent or documentation contributor:**

- Find a place where, say, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` and a milestone spec seem to describe the same thing differently? Check `docs/DECISIONS.md` first. If it's addressed there, follow the decision. If it isn't, `docs/PRODUCT_SPEC.md`'s own statement on the matter governs — and if `docs/PRODUCT_SPEC.md` itself is silent on the specific point, that's a gap, not a conflict (see §5).
- **Never silently pick a side.** An implementer who notices `docs/PRODUCT_SPEC.md` §19 describing a security-rule path one way and `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` describing the underlying data shape another way (see this exact case flagged in this batch's delivery report, concerning `users/{uid}.settings`) does not get to quietly implement whichever reading compiles first. It gets raised — as a `docs/DECISIONS.md` entry if someone with authority to decide is available, or as a flagged, visible open question if not.
- This rule exists specifically so contradictions discovered late (during implementation, or during a later documentation batch like this one) have one unambiguous, low-drama resolution path instead of becoming a debate re-litigated by whoever touches the code next.

## 5. When a Decision Hasn't Been Made Yet (Gaps vs. Conflicts)

A **conflict** (§4) is two documents disagreeing. A **gap** is different: no document has specified an answer at all. `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §1 already establishes the pattern for this: *"If a document needs a field this file doesn't have, that's a gap: raise it in `docs/OPEN_QUESTIONS.md` rather than inventing a divergent shape locally."*

This generalizes beyond just data-shape gaps:

- **A missing field or type** → raised in `docs/OPEN_QUESTIONS.md`, referenced by ID (e.g. `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`'s own `OQ-01`, `Lesson.passThreshold`'s unratified default).
- **A missing numeric/business-logic decision** (a rate-limit cap, a streak-reset timezone rule, an XP-award formula — none of which are specified anywhere in `docs/PRODUCT_SPEC.md` or `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` as of this batch) → same mechanism: flagged, not invented.
- **A missing document** (a referenced spec like `docs/MUSIC/MUSIC_ENGINE_SPEC.md` or `docs/AI/AI_GUARDRAILS.md` that doesn't exist yet at the time a dependent document is written) → referenced by its intended path anyway, so the dependent document doesn't drift once the real one lands, with a note that it's forthcoming rather than a silent assumption about its contents.

**The discipline this requires, stated plainly:** every document in this repository — including every file in this `docs/ROADMAP/` and `docs/TESTING/` batch — was written by extracting from `docs/PRODUCT_SPEC.md` and `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`, not by inventing plausible-sounding details to fill gaps. Where a concrete number or mechanism was needed and neither source document supplied one, the correct move is to say so and use an explicitly-labeled illustrative placeholder if an example is needed for clarity (see `docs/TESTING/UNIT_TESTING.md` §6.5's `passThreshold` handling for a worked example of this pattern), never to quietly assert a specific value as if it were settled.

## 6. Milestone-by-Milestone Sequencing

Per `docs/PRODUCT_SPEC.md` §35, point 5, and §28's own dependency chain: build one milestone at a time, in the order `docs/ROADMAP/MILESTONES.md` defines, feeding only the relevant sections of the specification as context for the milestone currently in progress. Do not begin M2 work while M1's acceptance criteria are unmet; do not let M4 (AI tutor) scope leak into M1–M3's implementation (`docs/PRODUCT_SPEC.md` §14's non-negotiable boundary — the deterministic engine and lesson content must be a complete, working, testable product before Gemini is added on top); do not let any milestone quietly absorb a Phase 2+ feature (`docs/PRODUCT_SPEC.md` §7's MVP cut, enforced per-milestone in each `M*.md` file's explicit "out of scope" section).

**Recommended tooling sequence** (`docs/PRODUCT_SPEC.md` §35 — supporting detail, not the core methodology point above): scaffold Firebase resources and a first-draft UI shell with AI Studio Build/Antigravity, move the repository to GitHub immediately so a browser-based builder never becomes the system of record, then hand the precision work — `packages/music-engine`, Firestore security rules, the Cloud Functions Gemini proxy and scoring logic, and the full test suite — to a coding agent with full repo access, milestone by milestone.

## 7. Definition of Done

Every milestone's completion is checked against one shared, reusable checklist rather than each milestone spec repeating its own version — see `docs/ROADMAP/DEFINITION_OF_DONE.md`. Individual `M*.md` files state their own *specific* acceptance criteria (what must be true) and reference the shared Definition of Done for the *cross-cutting* bar (tests pass, no scope leakage, docs updated, security rules not weakened, no secrets exposed) that applies identically to all six.

## 8. Out of Scope

- Sprint/ticket-level project management — this document governs specification and milestone sequencing, not day-to-day task tracking.
- The actual content of `docs/DECISIONS.md` — owned and populated by the project lead, not authored or pre-filled by this batch.
- Retroactively resolving every gap this batch discovered — those are surfaced as flagged open questions (per §5) in this batch's delivery report, for the lead to adjudicate via `docs/DECISIONS.md` or `docs/OPEN_QUESTIONS.md`, not silently resolved here.

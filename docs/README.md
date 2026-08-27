# AI Piano Tutor — Documentation System

## 1. What this is

This is the complete Spec-Driven Development (SDD) documentation set for **AI Piano Tutor**, a web app that teaches complete beginners piano through a virtual keyboard, structured lessons, note-ID/scale/chord drills, and a Gemini-powered AI tutor layered on top of a deterministic music engine.

**Nothing in this repository is application code.** No React components, no Cloud Functions, no Firebase resources, no `packages/music-engine` implementation exist yet. This is the specification an implementation effort (human or AI coding agent) builds from, milestone by milestone. If you are looking for the app itself, it doesn't exist yet — you're looking at the plan for it.

## 2. Source-of-truth hierarchy

When two documents disagree, resolve the conflict in this order — never the reverse:

1. **`PRODUCT_SPEC.md`** (this directory) — the ground-truth product specification. Everything else in `docs/` is derived from it, elaborates on it, or fills a gap it left open — nothing here overrides it.
2. **`ENGINEERING/TYPES_AND_CONTRACTS.md`** — the canonical shape of every shared data object. If a document uses a field or type this file doesn't have, that's a gap to raise in `OPEN_QUESTIONS.md`, not a reason to invent a divergent shape locally.
3. **`ARCHITECTURE/`** — how the system is put together: components, boundaries, data flow.
4. **`ENGINEERING/`** (the rest), `MUSIC/`, `AI/`, `SECURITY/`, `DATA/`, `UX/`, `CONTENT/` — domain-specific detailed specification, all consistent with 1–3 above.
5. **`ROADMAP/`** — how the above gets built, in order, milestone by milestone.
6. **`TESTING/`** — how every requirement above gets verified.

`DECISIONS.md` and `OPEN_QUESTIONS.md` sit alongside this hierarchy, not inside it: they record where documents 2–6 had to make a judgment call `PRODUCT_SPEC.md` didn't settle, or where a real gap remains. Check them before assuming a detail not covered above is simply undecided by omission — it may already have a proposed answer, or a tracked reason it's deliberately still open.

## 3. The one rule that governs everything

Every document in this repository, in every domain, is written downstream of one architectural boundary that is never softened, never re-litigated per feature, and never quietly worked around:

```
USER INPUT → INPUT ADAPTER → NoteEvent → MUSIC ENGINE → DETERMINISTIC VALIDATION
  → UI / SCORE / PROGRESS → OPTIONAL GEMINI EXPLANATION
```

**Gemini never determines whether a note, chord, or scale was played correctly. Gemini never calculates accuracy, timing, or a score. Gemini never determines lesson completion, XP, or streaks. Gemini never validates user-submitted performance data.** All of that is the exclusive, non-negotiable responsibility of `packages/music-engine` — a framework-independent package with zero imports of React, Next.js, Firebase, or any AI SDK. Gemini receives an already-computed, already-persisted result as read-only input and explains it in natural language. It does not recompute, sanity-check, second-guess, or contradict it.

If you are implementing anything and find yourself about to hand Gemini a raw `NoteEvent[]`, ask it to "check" a score, or let an AI response influence whether an exercise counts as passed — stop. That is this project's one hard boundary, and it exists so a product that presents itself as a piano *teacher* can actually be trusted.

## 4. Directory map

| Directory | What it covers | Start here if you need... |
|---|---|---|
| `PRODUCT_SPEC.md` (root) | The full 35-section ground-truth specification | The original vision, scope, and every numbered requirement's origin |
| `PRODUCT/` | Requirements, personas, user stories, user flows, MVP scope, future roadmap | "What are we building and for whom, precisely?" |
| `UX/` | Design principles, navigation, every MVP screen's full spec, components, responsive/accessibility rules | "What does this screen look like and do, in every state?" |
| `ARCHITECTURE/` | System/application architecture, data flow, music-engine/audio/AI/Firebase/security architecture | "How do the pieces fit together, and where are the boundaries?" |
| `ENGINEERING/` | Canonical types (`TYPES_AND_CONTRACTS.md`), tech stack, project structure, coding standards, error handling, performance, dependency policy | "What's the exact shape of X, and what are the engineering rules?" |
| `MUSIC/` | The deterministic music engine's detailed spec: notes, chords, scales, scoring algorithm (with worked examples), input normalization, keyboard mapping | "Exactly how does the engine decide what's correct?" |
| `AI/` | Every Gemini feature's full contract, the literal production prompt text, context/memory model, guardrails, error handling, cost control | "What does Gemini actually receive and return, and how is it kept safe/cheap/on-boundary?" |
| `SECURITY/` | Requirements, auth, authorization, the actual `firestore.rules` skeleton, secrets management, App Check, privacy | "Is this allowed, and how is it enforced?" |
| `DATA/` | Firestore schema, data model, access rules, indexes, retention | "What's actually stored, where, and for how long?" |
| `CONTENT/` | Lesson architecture, content schema, the actual MVP lesson list, authoring guidelines | "What are the real lessons, and how do I author one?" |
| `TESTING/` | Testing strategy, unit/integration/E2E/security/AI/accessibility/performance testing | "How do I verify this is actually correct?" |
| `OPERATIONS/` | Deployment, environment config, monitoring, analytics, cost monitoring, incident response | "How does this run in production?" |
| `ROADMAP/` | Methodology, the M0–M5 milestone breakdown, one detailed spec per milestone, Definition of Done | "What do I build first, and when am I actually done?" |

Plus, at this level: `AI_CODING_AGENT_INSTRUCTIONS.md` (the operating manual for whoever/whatever implements this), `REQUIREMENTS_TRACEABILITY.md` (every requirement ID traced end to end), `DECISIONS.md` (the decision log), `OPEN_QUESTIONS.md` (everything still genuinely open), `CHANGELOG.md` (how this documentation set itself has changed).

## 5. How to read this — human vs. coding agent

**If you're a person getting oriented:** `PRODUCT_SPEC.md` in full, then `PRODUCT/MVP_SCOPE.md`, then skim `ARCHITECTURE/SYSTEM_ARCHITECTURE.md` for the big picture, then `ROADMAP/MILESTONES.md` for the build order. Read `OPEN_QUESTIONS.md`'s BLOCKING section before assuming M0 can start immediately.

**If you're an AI coding agent about to implement something:** go to `AI_CODING_AGENT_INSTRUCTIONS.md` first — it is written specifically for you, and it names the exact read-order for a given implementation task. Do not start writing code from this README alone.

## 6. Current status

As of 2026-08-27: documentation-only. No milestone has started. `OPEN_QUESTIONS.md` lists three BLOCKING items (`OQ-01` lesson pass threshold, `OQ-14` XP/streak formula, `OQ-15` Gemini rate-limit numbers) that need Linford's sign-off before the milestones that depend on them (M2, M3, M4 respectively) can be considered fully specified — M0 and M1 do not depend on any BLOCKING item and can start immediately. See `DECISIONS.md` for what's already been proposed against each, and `CHANGELOG.md` for the full history of this documentation set.

## 7. Keeping this system honest

If implementation reveals that an architecture document was wrong, or a milestone's scope needs to change, **update the document — don't let code silently diverge from what it describes.** This is the entire point of doing this as Spec-Driven Development rather than writing code straight from the product brief. See `ROADMAP/DEVELOPMENT_METHODOLOGY.md` for how that update loop is supposed to work, and add an entry to `CHANGELOG.md` when it happens.

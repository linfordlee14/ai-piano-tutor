# Changelog

All notable changes to this documentation set are recorded here. This project has not started implementation — every entry so far describes documentation being created or corrected, not application behavior shipping.

## 2026-08-27 — Initial documentation system created

**Added**

- `docs/PRODUCT_SPEC.md` — the 35-section ground-truth product specification, carried in unchanged from the prior specification effort.
- The full `docs/` Spec-Driven Development documentation system derived from it: `PRODUCT/` (6 files), `UX/` (6 files), `ARCHITECTURE/` (8 files), `ENGINEERING/` (7 files, including the foundational `TYPES_AND_CONTRACTS.md`), `MUSIC/` (7 files), `CONTENT/` (4 files), `AI/` (7 files), `SECURITY/` (8 files), `DATA/` (5 files), `OPERATIONS/` (6 files), `TESTING/` (8 files), `ROADMAP/` (9 files) — 81 files total, plus this changelog and the five other top-level synthesis documents (`README.md`, `AI_CODING_AGENT_INSTRUCTIONS.md`, `REQUIREMENTS_TRACEABILITY.md`, `DECISIONS.md`, `OPEN_QUESTIONS.md`).

**Corrected during the same pass's internal QC review** (full detail in `docs/DECISIONS.md` §2):

- `TYPES_AND_CONTRACTS.md` §4.8 (`AIMemorySummary`) — corrected from "never client-readable" to "owner-only client-readable," resolving a contradiction against `PRODUCT_SPEC.md` §19 and the shipped `FIRESTORE_SECURITY_RULES.md`. Propagated to `AI_ERROR_HANDLING.md`, `M4_GEMINI_TUTOR.md`, and `M5_HARDENING.md`, which had each independently narrated the same contradiction as still-open. (D-01)
- `TYPES_AND_CONTRACTS.md` §4.2 (`Lesson`) — added a `hints?: string[]` field, closing a gap where `PR-LESSON-05` (escalating hints) had no field to represent it.
- `TYPES_AND_CONTRACTS.md` — added three new formal contracts (§4.16–4.18): `FunctionResult<T>`/`FunctionErrorCode`, `ExerciseFeedbackPromptInput`, `MotivationNudgePromptInput`, each previously existing only as informal, repeated JSON shapes across other documents. (D-03, D-04)
- A three-way `FunctionErrorCode` naming conflict, introduced while adding the type above and then caught by the requirements-traceability pass, was reconciled onto `ERROR_HANDLING.md`'s original naming (`UNAUTHENTICATED`, `VALIDATION_FAILED`, `RATE_LIMITED`, `GEMINI_UNAVAILABLE`, `NOT_FOUND`, `INTERNAL`) across seven files: `TYPES_AND_CONTRACTS.md`, `ERROR_HANDLING.md`, `AI_ERROR_HANDLING.md`, `AI_COST_CONTROL.md`, `AI_GUARDRAILS.md`, `GEMINI_SPEC.md`, `REQUIREMENTS_TRACEABILITY.md`. (D-04)
- The field-scoped `users/{uid}` write rule (settings embedded, not a separate sub-path) was cross-checked between `TYPES_AND_CONTRACTS.md` and `FIRESTORE_SECURITY_RULES.md` and confirmed already consistent — no file changes, decision recorded for traceability. (D-14)
- The `practiceSessions` write-cooldown's actual enforcement mechanism (Cloud Function logic, not a Firestore rule — Admin SDK writes bypass rules) was confirmed consistent across `FIRESTORE_SECURITY_RULES.md`, `SECURITY_TESTING.md`, and `INTEGRATION_TESTING.md`. (D-02)

**Known gaps surfaced, not yet closed** (full detail in `docs/OPEN_QUESTIONS.md`):

- Three BLOCKING items need Linford's ratification: `Lesson.passThreshold` (OQ-01), the XP-award/streak-grace-period formula (OQ-14), and the Gemini per-user daily rate-limit numbers plus counter storage shape (OQ-15).
- `SECURITY_TESTING.md` claims App Check test coverage that does not actually exist anywhere in `docs/TESTING/` (OQ-28) — flagged rather than fixed in this pass, since closing it requires authoring new test content, not a consistency correction.
- Firestore offline persistence (`PR-PLATFORM-04`) is fully specified in `ERROR_HANDLING.md` but scheduled in no M0–M5 milestone and tested nowhere (OQ-29) — same reasoning as above.
- `docs/PRODUCT/MVP_SCOPE.md`'s Drills summary table under-lists what's actually in scope (scale/chord drills, not just Note ID) relative to `PRODUCT_SPEC.md` §10 and the milestone specs that already build all three (OQ-16) — a one-line table fix, recommended but not yet made.
- Anonymous, never-linked user data has no retention/cleanup policy, and this pass deliberately left it **Open** rather than defaulting to "no action," since inaction was assessed as the weaker option on POPIA/GDPR grounds (D-13/OQ-20).
- A previously-unaddressed edge case in anonymous-to-authenticated account linking (the target credential already belonging to a separate account) was newly identified during this pass, based on Firebase Auth's documented behavior rather than found in any prior document (OQ-27).

**Not done in this pass, by design:** no application code, no dependency installs, no Firebase resources, no deployment. See `AI_CODING_AGENT_INSTRUCTIONS.md` for what happens next.

## 2026-08-28 — Post-creation QC reconciliation pass

**Mechanical fixes (stale references, factual errors, broken cross-references):**

- `CODING_STANDARDS.md` §3.4 — removed stale `FunctionResult<T>` open question; type was formalized in D-04 (`TYPES_AND_CONTRACTS.md` §4.16). Updated code comment and checklist item.
- `PROJECT_STRUCTURE.md` §2 — removed stale "once written" qualifier for `MUSIC_ENGINE_ARCHITECTURE.md`; file exists.
- `LESSON_ARCHITECTURE.md` §1 — corrected "15 lessons" to "14 lessons" to match `BEGINNER_CURRICULUM.md`.
- `TESTING_STRATEGY.md` §8 — fixed broken section reference: Android latency spot-check is `PERFORMANCE_TESTING.md` §5, not §6.
- `GEMINI_SPEC.md` §7 — fixed broken cross-reference: Theory Explanation template text is in `AI_PROMPTS.md` §4, not §3.
- `GEMINI_SPEC.md` §8 — fixed broken cross-reference: Motivation/Nudge template text is in `AI_PROMPTS.md` §5, not §4.
- `AI_PROMPTS.md` §1 — corrected false claim that section numbering matches `GEMINI_SPEC.md`; order matches, numbers differ (each document numbers independently).
- Seven stale "not yet written" / "parallel batch" disclaimers removed across: `DATA_ACCESS_RULES.md` §3, `DATA_RETENTION.md` §4, `FIRESTORE_SECURITY_RULES.md` header, `SECURITY_REQUIREMENTS.md` §6 and header, `GEMINI_SECURITY.md` header, `PRIVACY.md` header. All referenced files now exist.

**New tracked items (full detail in `docs/OPEN_QUESTIONS.md` and `docs/DECISIONS.md`):**

- OQ-30 / D-17 — `tempo` vs `tempoBpm` field-naming mismatch between `PRODUCT_SPEC.md` §17 and `TYPES_AND_CONTRACTS.md` §4.2. Proposed: `tempoBpm` as canonical (NON-BLOCKING).
- OQ-31 / D-18 — `ExerciseVerdict` uses string note names (`correctNotes`, `missedNotes`, `extraNotes`) despite `TYPES_AND_CONTRACTS.md` §3's MIDI-canonical rule. Proposed: acknowledge as deliberate display-layer exception (NON-BLOCKING).

**Verified, no action needed:**

- OQ-28 confirmed: `SECURITY_TESTING.md` §10's claim that App Check coverage exists in `AI_TESTING.md` and `INTEGRATION_TESTING.md` is false — neither file contains App Check assertions.
- OQ-29 confirmed: Firestore offline persistence (`PR-PLATFORM-04`) remains unscheduled in any M0–M5 milestone.
- No Phase 2+ scope leakage found in any MVP-scoped document or milestone file.
- `MILESTONES.md` fully consistent with individual M-files (dependencies, objectives, acceptance criteria).
- All five AI prompt templates consistent across all 7 `AI/` files (input schemas, output contracts, cost tiers, callable routing).
- Firestore security rules consistent across `SECURITY_REQUIREMENTS.md`, `FIRESTORE_SECURITY_RULES.md`, and `DATA_ACCESS_RULES.md`.

## Format going forward

Each future entry should record: the date, what was added/changed/removed, which milestone or decision it relates to, and — for anything touching `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` or `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` specifically — a cross-reference to the `docs/DECISIONS.md` entry that authorized it. An architecture or contract change discovered necessary during implementation (per `docs/ROADMAP/DEVELOPMENT_METHODOLOGY.md`'s update-loop) belongs here the same day it's made, not batched up later.

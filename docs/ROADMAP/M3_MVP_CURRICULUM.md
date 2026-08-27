# M3 — Full MVP Content + Note ID Game

## 1. Objective

Turn the one-lesson tech demo M2 proved end to end into a complete beginner course. By the end of M3, a learner can progress linearly from Level 1 through the intro-to-major-triads lesson with no missing or broken step (`docs/PRODUCT_SPEC.md` §28's literal M3 acceptance line) — the remaining ~11–14 lessons exist and are sequenced, the Note Identification game is playable with adaptive difficulty, streak and XP accrue from real practice, and the Dashboard/Progress screens reflect a learner's actual history instead of M2's single-lesson placeholder state.

**A note on this file's name:** per this documentation batch's scoping instruction, this file is named `M3_MVP_CURRICULUM.md`, which foregrounds only half of `docs/PRODUCT_SPEC.md` §28's own milestone title, "Full MVP content **+ Note ID game**." The Note Identification game (§3 below) is fully in scope for this milestone, equally with the lesson content — the filename is a documentation-batch naming choice, not a signal that the game is secondary or deferred.

**Source of truth:** `docs/PRODUCT_SPEC.md` §5 (User Stories — Note ID Game, Practice & Progress), §6 (Core Features), §7 (MVP Scope — lesson sequence, scale/chord lesson scope), §9 (UX Architecture), §10 (Onboarding, Learn, Practice Hub, Note ID Game, Progress/Stats screens), §17 (Firestore Data Model), §28 (M3 row), §29. Contracts: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2–4.3 (`Lesson`, `Exercise`), §4.7 (`ProgressSummary`), §4.15 (`UserProfile` — `streak`, `xp`).

## 2. Prerequisites

**M2** must be done: the Lesson Player, `getExerciseFeedback` (deterministic-only), the `lessons`/`practiceSessions`/`progressSummary` security rules, and Auth UI (sign-in/sign-up/linking) all work and are tested against one real lesson. M3 authors many more lessons and one game mode against that same, already-proven mechanism — it does not change `packages/music-engine` (fully built and tested in M1) and it does not depend on M4's AI layer in any way (see §4 below).

## 3. Exact Scope

- **The remaining ~11–14 lessons** (bringing the total to the ~12–15 named in `docs/PRODUCT_SPEC.md` §7), authored as static JSON in `content/lessons/` and seeded the same way M2's single lesson was, covering the full stated progression: keyboard orientation → note names → octaves → simple melodies → basic rhythm → intro to major triads (§7, "the original brief's Levels 1–6, roughly"). Every lesson's `Lesson.level`/`prerequisiteLessonId` chain forms one single linear, acyclic path (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 — no branching paths in MVP). Exact lesson-by-lesson pedagogical content is owned by `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`/`docs/CONTENT/BEGINNER_CURRICULUM.md` (parallel batch); this milestone consumes that content against the schema it defines, it does not author the pedagogy itself.
- **Guided scale lessons:** C major plus 3–4 more common keys (§6, §7) — each using the engine's already-built ordered/unordered scale validation (M1), wrapped in real `Lesson` content for the first time.
- **Chord lessons:** major triads, root position only (§6, §7) — using the engine's already-built chord detection (M1). Per `docs/PRODUCT/MVP_SCOPE.md` §4's explicit note: the *engine* already detects minor triads too (built in M1, since detection must cover all 12 major and 12 minor triads per §29), but only major-triad *lesson content* ships in M3 — no minor-triad lesson exists, even though a learner who happened to play a minor triad in Free Play would still see it correctly recognized.
- **Note Identification game** (`Exercise.kind: 'noteId'`), one mode, adaptive difficulty (§6, §10): difficulty (`Exercise.difficulty`) steps up after a run of correct answers and down after a run of misses, visibly indicated the moment it changes — never inferred silently (`docs/PRODUCT/USER_STORIES.md` §5.2). Difficulty state is session-scoped (held in the Zustand engine/UI store, §21), not persisted to Firestore — consistent with `Exercise` being static app config, not a Firestore-persisted entity (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.3).
- **Practice Hub screen** (§10): entry points to the Note ID game, scale drill, chord drill, and Free Play, with "no lesson scaffolding." `Exercise.kind` already types `'scaleDrill'` and `'chordDrill'` as first-class kinds alongside `'noteId'` (§4.3), and §10 names both explicitly as Practice Hub entry points — M3 builds all three drill kinds, scoped to exactly the same content already taught in this milestone's own lessons (C major + the 3–4 additional keys; major triads, root position), inventing no new pedagogical content. *(See §4 below — flagged as a reasonable but non-explicit extrapolation from `docs/PRODUCT/MVP_SCOPE.md`'s narrower summary table.)*
- **Streak + XP** (`UserProfile.streak`, `.xp`, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15): computed and written server-side as an extension of `getExerciseFeedback`'s existing write path (see §4 below).
- **Deterministic daily practice session** (§7's explicit pushback: "next incomplete lesson + review your last 3 misses" — *not* AI-generated in MVP) and the **Dashboard's "one weak-area callout"** (§6): both computed from the learner's own `practiceSessions` history, which §17 names directly as powering "Progress/Stats and weak-area detection" (see §4 below).
- **Dashboard**, upgraded from M2's single-lesson placeholder to its full MVP form (§10): greeting, streak, XP, "Continue lesson" CTA reflecting the true furthest-incomplete lesson, today's practice card, one weak-area callout, quick links to Piano/Tutor. The Tutor quick link is present as navigation only — the Tutor screen itself doesn't exist as a working AI feature until M4 (see §11).
- **Learn (Lesson Path) screen** (§10): the full linear list/tree of levels with locked/unlocked/completed states, each unlock gated on the prerequisite lesson's `passThreshold` being met — this requires `getExerciseFeedback` to cascade-unlock the next lesson(s) on a pass, which M2's single-lesson scope never had reason to exercise (see §4 below).
- **Onboarding / Quick Skill Check screen** (§10): 3–4 skippable questions, used only to set a starting lesson index, never gatekeeping; default is skip → start at Level 1. Deferred to M3 specifically because, per M2's own out-of-scope note, it "is meaningless with only one lesson seeded" and "becomes a real feature once there's a path to skip into" — which M3 is what provides.
- **Progress/Stats screen** (under Profile, §10): accuracy over time, weak notes/chords, lessons completed, practice minutes — deferred here from M2's out-of-scope list for the same reason (needs real multi-lesson history to be meaningful).

## 4. Design Decisions Inferred For This Milestone (Flagged, Not Silently Assumed)

Several mechanisms this milestone needs are not spelled out mechanically in `docs/PRODUCT_SPEC.md` — each resolved here in the direction the more specific/authoritative document already supports, per `docs/ROADMAP/DEVELOPMENT_METHODOLOGY.md` §5's gap-handling rule, and named explicitly so a future implementer doesn't have to re-derive or silently re-decide them differently:

- **Weak-area callout and daily-practice generator read `practiceSessions` directly, never `aiMemory`.** `docs/PRODUCT_SPEC.md` §17 states `practiceSessions` "powers Progress/Stats **and weak-area detection**" directly — so both features query the learner's own recent `practiceSessions` (via the `(userId, completedAt desc)` composite index already specified in §17) client-side, the same way any other owner-readable Firestore data is read in this stack (§20, §21). They deliberately do **not** depend on the `aiMemory/summary` document, which doesn't exist until M4, whose aggregation logic is explicitly scoped to `functions/src/ai/` (`docs/ENGINEERING/PROJECT_STRUCTURE.md`), and which exists to serve `TutorRequest` assembly, not the deterministic dashboard. This is what keeps M3 fully independent of M4 in practice, matching `docs/ROADMAP/MILESTONES.md` §4's dependency graph (M3 and M4 are both independent children of M2 — neither should require the other to exist).
- **Streak/XP writes extend `getExerciseFeedback`'s existing transaction, not a new write.** `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md` PERF-3 already establishes one write (or one batch/transaction) per completed exercise attempt; M3 adds `xp`/`streak` fields to that same transaction rather than issuing a second write, keeping PERF-3's guarantee intact as the number of fields written grows.
- **Exact XP-award amount and the streak grace/reset rule are open, not invented.** Neither `docs/PRODUCT_SPEC.md` nor `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` specifies an XP formula (how much XP a lesson vs. a drill round awards) or the exact streak-break semantics (whether a single missed calendar day resets `streak.count` to `0`/`1`, whether multiple sessions on the same day count once) beyond "calendar-day based" (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15). `docs/ROADMAP/DEVELOPMENT_METHODOLOGY.md` §5 names an XP-award formula as its own worked example of a gap that must be flagged, not guessed. Implement both behind a single, clearly-named, clearly-documented constant/function (e.g. `computeXpAward()`, `applyStreakUpdate()`) so the *mechanism* is correct and tested independent of the eventual real values — the same pattern `docs/TESTING/UNIT_TESTING.md` §6.5 already uses for `Lesson.passThreshold`.
- **Practice Hub's scale/chord drills are in scope**, even though `docs/PRODUCT/MVP_SCOPE.md` §2's summary table names only "Note Identification game" under "Drills." `docs/PRODUCT_SPEC.md` §10 and `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.3 both independently and specifically commit to `'scaleDrill'`/`'chordDrill'` existing — read here as elaboration of the summary table rather than a contradiction of it (§7's MVP bullet list names the flagship Note ID game; it doesn't say drills are the *only* Practice Hub content). Flagged for explicit ratification in this batch's delivery report rather than asserted as uncontested.
- **Onboarding-answer submission needs a Function**, since `currentLessonId`/`currentLevel` are Function-written-only fields (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15) and account-document creation is already an Auth `onCreate` trigger (`docs/SECURITY/FIRESTORE_SECURITY_RULES.md` §4.1). `docs/ENGINEERING/PROJECT_STRUCTURE.md` names no existing home for this one-time profile-setup logic; `functions/src/account/` is the closest fit by analogy (profile lifecycle, not scoring or AI) but this isn't a ratified path — flagged as a minor, non-blocking open item.

## 5. Files/Components Expected

```
content/lessons/
  l02-*.json … l1x-*.json          # the remaining ~11–14 lessons, same schema as M2's l01
  (scale + chord lesson entries included in this same set, not a separate collection)

functions/src/scoring/
  applyStreakAndXp.ts               # extends the existing write transaction — see §4
  unlockNextLessons.ts               # cascades progressSummary[nextLessonId].status: 'unlocked'
                                      # on a pass, per Lesson.prerequisiteLessonId
  validateAttempt.ts                 # extended (not rewritten) to also resolve exerciseId-based
                                      # attempts (noteId/scaleDrill/chordDrill) against static
                                      # Exercise config, alongside the existing lessonId path
functions/src/account/
  completeOnboarding.ts               # (proposed home, see §4) sets initial currentLessonId/
                                       # currentLevel from skill-check answers

apps/web/lib/exercises/
  noteIdPool.ts, scaleDrillPool.ts, chordDrillPool.ts   # static Exercise/ExercisePoolItem config

apps/web/
  app/(app)/learn/                   # full lesson-path list/tree, locked/unlocked/completed states
  app/(app)/practice/                # Practice Hub route
  app/(app)/practice/note-id/         # Note ID game route
  app/(app)/practice/scales/           # scale drill route
  app/(app)/practice/chords/            # chord drill route
  app/(app)/onboarding/                  # Quick Skill Check route
  app/(app)/profile/progress/             # Progress/Stats sub-route
  components/practice-hub/                 # PracticeHubGrid, DifficultyIndicator
  components/dashboard/                     # WeakAreaCallout, TodaysPracticeCard, StreakBadge, XpBadge
  hooks/useWeakAreas.ts                      # client-side aggregation over recent practiceSessions (§4)
  hooks/useDailyPracticePlan.ts               # same data source, deterministic rule (§7)
```

## 6. Technical Requirements

- `getExerciseFeedback`'s write path (already atomic per M2/PERF-3) now writes `PracticeSession`, `ProgressSummary` (including any newly-unlocked lessons), **and** `UserProfile.xp`/`.streak` together, in the same transaction/batch — never as separate writes that could partially fail (`docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md` PERF-3).
- The same Function now accepts and correctly resolves **either** a `lessonId`-based or an `exerciseId`-based `ExerciseAttempt` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4 already types both as optional/mutually exclusive) — drill attempts (Note ID, scale drill, chord drill) load their definition from the static `Exercise` config bundled with the client/deploy, not from Firestore, while lesson attempts continue to load from the `lessons` collection exactly as in M2.
- Lesson unlock cascade logic reads `Lesson.prerequisiteLessonId` to determine which lesson(s), if any, transition from `'locked'` to `'unlocked'` when the current one is passed — this logic exists for the first time in M3 because M2's single-lesson scope had nothing to cascade into.
- Streak logic is calendar-day based on `streak.lastPracticeDate` (`YYYY-MM-DD`), computed from server time, never client-supplied time (consistent with `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4's "server does not trust client clock for scoring" principle, extended here to streak/XP as the same class of server-authoritative computation).
- The weak-area callout and daily-practice-plan hooks (§4, §5) query `practiceSessions` directly, bounded (e.g. `limit(N)` on the existing composite index) — this is a deliberate, small, separate read from the Dashboard's single flattened `progressSummary` read (`docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md` PERF-4), not a violation of it; PERF-4's guarantee is specifically about progress data, not this secondary aggregation.

## 7. Data Structures / Contracts Touched

`Lesson` (§4.2) — the full ~12–15-lesson set now exists, exercising the `level`/`prerequisiteLessonId` chain for the first time as an actual multi-node graph rather than a single node. `Exercise`, `ExercisePoolItem` (§4.3) — first real static config authored against this shape, for all three drill kinds. `ProgressSummary`/`LessonProgress` (§4.7) — the `status` transition machinery (`locked` → `unlocked` → `completed`) is exercised for the first time across multiple lessons. `UserProfile.streak`/`.xp` (§4.15) — first real writes to these fields; both were present in the type since M0/M1 but untouched until now.

## 8. Tests Required

- **`docs/TESTING/E2E_TESTING.md` §5.4** (full lesson-sequence progression) — this milestone's own literal gate, and explicitly named in that document as only runnable "once M3's curriculum lands."
- **`docs/TESTING/E2E_TESTING.md` §5.5** (cross-device/cross-session resume) — deferred here from M2 specifically because it's "more naturally exercised once there's more than one lesson to distinguish states across" (M2's own out-of-scope note).
- **`docs/TESTING/INTEGRATION_TESTING.md` §5** — server-side re-validation tests extend to cover `exerciseId`-based attempts (drills), not just `lessonId`-based ones, and to assert the streak/XP fields update correctly and atomically alongside `practiceSessions`/`progressSummary`.
- **`docs/TESTING/UNIT_TESTING.md`** — no change expected to `packages/music-engine` itself in this milestone; if any is needed, it re-triggers that suite's full requirements, not a partial re-check.
- **`docs/TESTING/ACCESSIBILITY_TESTING.md`** — axe-core now runs against every newly added core screen (Practice Hub, Note ID Game, Onboarding, Progress/Stats, the full Learn path) per that document's §4 screen list, not just the M0–M2 subset.
- A rules/Function test asserting the lesson-unlock cascade only unlocks the *correct* next lesson(s) per `prerequisiteLessonId` — not all lessons, and not the wrong one — since this is new logic this milestone introduces (no equivalent exists in `docs/TESTING/INTEGRATION_TESTING.md` yet because M2 had nothing to cascade into; add it as an extension of that document's §5 test group).

## 9. Acceptance Criteria

Reproduced and expanded from `docs/PRODUCT_SPEC.md` §28 (M3 row) and §29:

- [ ] A learner can progress linearly from Level 1 through the intro-to-triads lesson with no missing or broken step (§28's literal M3 acceptance line).
- [ ] A learner can complete the full MVP lesson sequence with no dead end or broken step (§29).
- [ ] The Note Identification game's difficulty visibly steps up/down as the learner improves/struggles (`docs/PRODUCT/USER_STORIES.md` §5.2).
- [ ] Streak and XP are visible on the Dashboard and increment correctly from real completed sessions, calendar-day based (`docs/PRODUCT/USER_STORIES.md` §8.2).
- [ ] The Dashboard's weak-area callout and today's-practice card reflect real, changing `practiceSessions` history — not static or one-time (`docs/PRODUCT/USER_STORIES.md` §7.2, §8.1).
- [ ] `UserProfile.currentLessonId` drives "Continue lesson" and reflects the true furthest-incomplete point (`docs/PRODUCT/USER_STORIES.md` §8.3).

## 10. Definition of Done

Checked against `docs/ROADMAP/DEFINITION_OF_DONE.md` in full. Item 5 ("security rules not weakened") is live and specifically relevant here: the lesson-unlock cascade and streak/XP writes are new server-side logic touching already-rules-protected paths — any temptation to loosen a rule to make a new field "just work" from the client must be resisted; the fix is always in the Function, never the rule (per that document's own framing). Item 6 remains vacuous — still no Gemini credential exists at M3.

## 11. Out of Scope for This Milestone

- **Any Gemini/AI involvement whatsoever** — including the `aiMemory` document, the Tutor screen as a working feature (its nav entry exists per §3, but it opens nothing real yet), and Gemini-personalized versions of the daily-practice plan or weak-area suggestion — all M4, and the pushback in `docs/PRODUCT_SPEC.md` §7 is explicit that the deterministic version is what MVP ships, not a placeholder for the AI version.
- **MIDI input, additional scales/chords beyond C major + 3–4 keys and major-triad root position, inversions as pedagogical content, the song learning module, achievements beyond streak/XP, offline PWA caching** — all Phase 2+ (`docs/PRODUCT_SPEC.md` §7, §8); none of these belong in "full MVP curriculum," which means the *complete MVP-scoped* curriculum, not an expanded one.
- **Account deletion, data export** — M5, per this documentation batch's task framing and M2's own out-of-scope note.
- **Any change to `packages/music-engine`'s detection or scoring algorithms** — fully built and tested in M1; M3 authors content and UI against that already-frozen engine surface, it does not modify it.

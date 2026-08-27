# M4 — AI Tutor

## 1. Objective

The AI layer, built last and strictly on top of the deterministic core, exactly as `docs/PRODUCT_SPEC.md` §14 architects it. By the end of M4: the missed-F worked example (§13) produces a correct, on-tone response, and disabling the Gemini Function entirely breaks nothing else (§28's literal M4 acceptance line) — every other screen, including the exercise-feedback and scoring path M2 already proved, keeps working exactly as before. Gemini is added, never substituted for anything the deterministic engine already does.

**A note on this file's name:** this file is named `M4_GEMINI_TUTOR.md`, while `docs/PRODUCT_SPEC.md` §28 names this milestone "AI tutor." Both names refer to the same thing — Gemini is this product's only AI provider (§14), so "the AI layer" and "the Gemini layer" are interchangeable here — but the filename foregrounds the specific technology rather than the product-level feature name §28 uses. Noted for consistency with this batch's naming-discrepancy-flagging convention, not because the scope differs.

**Source of truth:** `docs/PRODUCT_SPEC.md` §14 (AI/Gemini Architecture — the non-negotiable boundary), §15 (AI Prompt Architecture), §16 (Firebase Architecture — App Check), §19 (Security), §20 (API Architecture), §28 (M4 row), §29, §31 (Cost Considerations). Contracts: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8–4.10 (`AIMemorySummary`, `TutorRequest`, `TutorResponse`). Guardrails and error handling, already written in full: `docs/AI/AI_GUARDRAILS.md`, `docs/AI/AI_ERROR_HANDLING.md` — this milestone spec does not restate their content, it implements against it.

## 2. Prerequisites

**M2** must be done — one real, server-scored lesson exists, `getExerciseFeedback` computes and persists a real `ExerciseVerdict`, and `practiceSessions` has at least one real record to build tutor context from. **M4 does not depend on M3.** Per `docs/ROADMAP/MILESTONES.md` §4: M3 and M4 are independent branches that both build on M2 and "can proceed in either order, or in parallel, once M2's acceptance criteria are met" — M4 needs *a* real scored exercise to explain and build memory from, which M2 alone already provides; it does not need the *full* ~12–15-lesson curriculum M3 delivers. If M3 and M4 are built sequentially rather than in parallel, M4 may in practice land after M3 — that is a scheduling choice, not a dependency requirement, and this milestone's own scope does not assume M3's curriculum, drills, streak, or XP exist.

## 3. Exact Scope

- **`getExerciseFeedback` gains its AI-explanation half** — this is the additive extension M2 §4 explicitly forward-referenced ("added in M4... as an additive extension of the already-shipped, already-tested M2 behavior... not a rebuild"). The Function's deterministic scoring, `PracticeSession`/`ProgressSummary` write, and (from M3) streak/XP write are **unchanged**; M4 adds one more step after scoring completes: call the Exercise Feedback prompt template (cheapest tier, `docs/PRODUCT_SPEC.md` §15) with `{exerciseType, expected, played, accuracyScore, missedNotes, attemptNumber, learnerLevel}` — already-summarized fields only, never the raw `NoteEvent[]` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 3) — and attach the resulting `feedbackText` to the response. Per `docs/AI/AI_ERROR_HANDLING.md` §5.1, **every** failure mode of this call (down, slow, rate-limited, invalid output, or the app's own daily cap exhausted) is absorbed silently: the Function still returns `{ ok: true, data: { verdict, feedbackText } }`, with `feedbackText` replaced by the deterministic canned line built from `verdict` alone. The `verdict` itself, and whether the exercise counts as passed/persisted, is **never** affected by this call succeeding or failing — this is the single most important behavioral guarantee in this milestone.
- **New callable Function, `getTutorResponse`** (`docs/PRODUCT_SPEC.md` §20): serves three of the five templates from §15 through one callable, per mode:
  - **Tutor Q&A** (mid tier): the client-facing signature accepts only `{ learnerQuestion: string }` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9) — the Function assembles the rest of `TutorRequest` (`recentExerciseContext`, `learnerMemorySummary`, `skillLevel`) server-side from Firestore, so a learner can never fabricate their own performance context to steer the tutor's answer.
  - **Theory Explanation** (cheap, cacheable): keyed on `(concept, skillLevel)`, checked against a cache before any live Gemini call — per `docs/PRODUCT_SPEC.md` §15, "likely the single highest-leverage cost optimization in the whole AI layer."
  - **Motivation/Nudge** (cheapest, mostly pre-written): the common cases (e.g. a streak milestone) resolve from a small pre-written pool with **zero** live Gemini calls; only genuinely novel cases fall through to a live call.
- **`aiMemory/summary`** (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8) is written for the first time in this milestone: a single document, rewritten in place (never appended to) from recent `practiceSessions`, computed by aggregation logic living in `functions/src/ai/` (`docs/ENGINEERING/PROJECT_STRUCTURE.md`). See §4 below for its read-access resolution.
- **Guardrails, implemented exactly as `docs/AI/AI_GUARDRAILS.md` already specifies** (this milestone does not re-derive them, it builds against them): per-template output-length caps and cap-violation handling (§3 of that document), the no-invented-facts check — upstream in the system instruction plus the downstream note-name-token check for Exercise Feedback specifically (§4), the `validateOutput()` → `fallback()` algorithm for every template (§5), and the three-layer prompt-injection defense on `learnerQuestion` (length cap, delimiting/no-concatenation, zero tool-calling access — §7).
- **Error handling, implemented exactly as `docs/AI/AI_ERROR_HANDLING.md` already specifies:** the `GEMINI_UNAVAILABLE`/`RATE_LIMITED` code taxonomy (§6), and the per-feature fallback mapping (§5) — Exercise Feedback silent, Tutor Q&A/Theory Explanation surfaced as a distinct fallback-card UI component, Motivation/Nudge silent via the pool.
- **AI Tutor screen** (`docs/PRODUCT_SPEC.md` §10): chat interface, quick-suggestion chips (including "What should I practice today?" — which reuses M3's already-built deterministic daily-plan logic directly, **not** a Gemini call, per `docs/PRODUCT/USER_STORIES.md` §7.2), a persistent subtle indicator that recent-session context is loaded, and the exact fallback card from `docs/AI/AI_ERROR_HANDLING.md` §5.2 on `GEMINI_UNAVAILABLE`/`RATE_LIMITED`.
- **"Ask the tutor why" becomes real** — the Lesson Player entry point M2 shipped as an inert placeholder now opens the Tutor screen pre-loaded with the just-completed exercise's context (§10).
- **App Check enforcement extends** to `getTutorResponse` and the AI-explanation call inside `getExerciseFeedback` (§16, §19) — App Check itself was registered at M0 and already enforced on `getExerciseFeedback`'s core path since M2; M4 extends enforcement to the new Gemini-calling surface specifically.
- **Per-user rate limiting**, atomically checked and incremented inside the Function *before* any Gemini call is attempted (§19) — the exact daily cap value and where the counter is persisted are open (see §4 below).
- **Gemini credential provisioning:** stored as a Cloud Functions secret (Secret Manager), read only inside server-side execution — never in client code, never a `NEXT_PUBLIC_*` variable (§19). This is the first milestone where a real Gemini credential exists in the system at all.
- **Cost instrumentation** (§31): a per-call log recording template name, tier, token counts, and cache-hit/miss — "catching a cost regression in week one is cheap; catching it after a usage spike is not."

## 4. Design Decisions Inferred For This Milestone (Flagged, Not Silently Assumed)

- **`aiMemory/summary` is client-readable by the owning uid.** This milestone spec originally had to resolve a contradiction between `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8 (which said "never read... directly by the client") and both `docs/PRODUCT_SPEC.md` §19 and the shipped `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` §5.4 (`allow read: if isOwner(uid)`), landing on the shipped rules file as ground truth. **That contradiction has since been fixed at the source:** `TYPES_AND_CONTRACTS.md` §4.8 now correctly states `AIMemorySummary` is owner-readable client-side, matching this milestone's resolution exactly (`docs/DECISIONS.md` D-01). Nothing about this milestone's design changes — `aiMemory/summary` is owner-readable in M4, as always planned — this note is updated only so a reader doesn't go looking for a contradiction that no longer exists.
- **Rate-limit counter persistence and the exact daily cap value are open.** No field for a per-user daily Gemini-call counter exists anywhere in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`'s `UserProfile` or any other contract, and no cap number is given in `docs/PRODUCT_SPEC.md`. `docs/TESTING/AI_TESTING.md` §7 already flags exactly this gap ("do not invent a specific Firestore field path in this document"). M4 must implement *some* atomic, race-safe counter mechanism (a natural candidate: a small server-only field or subdocument scoped per user per calendar day, written only by Cloud Functions) — but the exact shape is a genuine open decision for whoever ratifies it, not something this milestone spec invents as settled fact.

## 5. Files/Components Expected

```
functions/src/ai/
  promptTemplates.ts          # the 4 MVP templates' system instructions + input/output contracts
  validateOutput.ts             # per-template cap/shape/no-invented-facts checks (AI_GUARDRAILS.md §3-4)
  fallback.ts                    # per-template deterministic fallback content (AI_GUARDRAILS.md §5)
  getTutorResponse.ts              # the callable: Tutor Q&A / Theory Explanation / Motivation modes
  aiMemory.ts                       # recomputes aiMemory/summary from recent practiceSessions
  rateLimiter.ts                     # atomic per-user daily counter check + increment (see §4)
  geminiClient.ts                     # thin wrapper around the Gemini SDK — the ONLY module in this
                                       # repo permitted to import it (functions/ only, never apps/web
                                       # or packages/music-engine)
functions/src/getExerciseFeedback.ts   # extended (not rewritten): scoring path unchanged, one added
                                        # step calling promptTemplates' Exercise Feedback template

apps/web/
  app/(app)/tutor/                      # AI Tutor chat route — now a real feature, not just a nav link
  components/tutor/                      # ChatWindow, QuickSuggestionChip, ContextLoadedIndicator,
                                          # GeminiUnavailableFallbackCard (docs/AI/AI_ERROR_HANDLING.md §5.2)
  lib/functions/getTutorResponse.ts       # typed callable wrapper, one per docs/ENGINEERING/
                                           # PROJECT_STRUCTURE.md §5's "one wrapper per Function" rule
```

## 6. Technical Requirements

- The Gemini SDK is a dependency of `functions/` **only** — never `apps/web`, and under no circumstance `packages/music-engine`, whose zero-AI-SDK-import rule (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 rule 2) predates and is unaffected by this milestone.
- Every Gemini call in the system is configured for plain-text or structured-output generation **only** — zero tools, zero function-calling access registered, for any template, ever (`docs/AI/AI_GUARDRAILS.md` §7.3) — verified by an automated check on the call configuration itself, not behavior alone.
- `validateOutput()` gates every response before it can reach a client, for every template, with no call site permitted to bypass it "just this once" (`docs/AI/AI_GUARDRAILS.md` §11).
- The rate-limit check runs, and can reject, **before** any Gemini call is attempted — a request that's already over the daily cap must never reach the Gemini API at all (`docs/AI/AI_ERROR_HANDLING.md` §4).
- Model tier is selected per template exactly as `docs/PRODUCT_SPEC.md` §15 assigns it (cheapest: Exercise Feedback, Motivation/Nudge, Theory Explanation; mid: Tutor Q&A) — no specific model version is hardcoded anywhere in this milestone's implementation; §14 requires re-checking current pricing/tiers at implementation time rather than trusting a spec-frozen model ID.
- The Daily Practice Plan template (§15, tagged `(Phase 2)`) is **not built** in this milestone under any circumstance — see §11.

## 7. Data Structures / Contracts Touched

`AIMemorySummary` (§4.8) — first real document of this shape, written and re-read within this same milestone. `TutorRequest`/`TutorResponse` (§4.9–4.10) — first real implementation; `TutorRequest.recentExerciseContext` is the first place an `ExerciseVerdict` is consumed by something other than the UI/`PracticeSession` write path it was originally produced for — confirm the same `ExerciseVerdict` object is passed through unmodified, never recomputed or reinterpreted by anything in `functions/src/ai/`.

## 8. Tests Required

- **`docs/TESTING/AI_TESTING.md` — the primary gate for this milestone**, in full: per-template contract tests (§5), fallback-trigger tests for every template including the required negative case (§6), rate-limiting including the atomic-under-concurrency case (§7), and the human-in-the-loop Exercise Feedback eval set running on its own non-CI-blocking cadence (§8).
- **`docs/TESTING/E2E_TESTING.md` §5.3** (Gemini-unavailable degraded state) — this milestone's own literal browser-level proof that disabling Gemini breaks nothing else; must pass against the **full** reference scenario (§4 of that document) with Gemini mocked off.
- **`docs/TESTING/INTEGRATION_TESTING.md`** — `getExerciseFeedback`'s extended write path (§3 above) is re-verified: the AI-explanation step's success or failure must have zero effect on whether `PracticeSession`/`ProgressSummary`/streak/XP are written correctly.
- **`docs/TESTING/SECURITY_TESTING.md` §7** — the CI bundle-grep for leaked Gemini credentials becomes live and meaningful for the first time at this milestone (M0–M3 had no credential to leak); this check must pass from M4's first merge onward, not deferred to M5.
- The canonical missed-F worked example (`docs/PRODUCT_SPEC.md` §13, §29), run through the real Exercise Feedback template end to end (mocked Gemini response in CI; the real API in the periodic human eval set per `docs/TESTING/AI_TESTING.md` §8) — asserting the `feedbackText` is grounded in and never contradicts `missedNotes: ['F']`/`accuracy: 0.8`.

## 9. Acceptance Criteria

Reproduced and expanded from `docs/PRODUCT_SPEC.md` §28 (M4 row) and §29:

- [ ] The missed-F worked example produces a correct, on-tone `feedbackText` (§28's literal M4 acceptance line).
- [ ] Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly, not assumed (§28, §29).
- [ ] A tutor response never states a note/chord/theory fact that contradicts the structured data it was given (§29).
- [ ] Per-user rate limiting demonstrably blocks a scripted burst of tutor calls beyond the configured cap (§29).
- [ ] No Gemini credential appears anywhere in the built client bundle (§29 — first live-checkable at this milestone).
- [ ] Exercise Feedback's Gemini call failing, timing out, or being rate-limited never blocks, delays, or alters the `ExerciseVerdict` the learner sees (`docs/AI/AI_ERROR_HANDLING.md` §11).

## 10. Definition of Done

Checked against `docs/ROADMAP/DEFINITION_OF_DONE.md` in full. Item 6 ("no secrets exposed") becomes **live** starting this milestone for the first time — the CI bundle-grep check must actually run and pass on every PR from M4 onward, not remain a vacuous placeholder as it was through M0–M3. Item 5 ("security rules not weakened") is directly relevant to the `aiMemory` read-access resolution in §4 above: the rule that ships is the one already written in `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`, not a newly loosened one invented to make this milestone's UI easier to build.

## 11. Out of Scope for This Milestone

- **The Daily Practice Plan Gemini template** — explicitly tagged `(Phase 2)` in `docs/PRODUCT_SPEC.md` §15, and explicitly pushed back on for MVP in §7 ("calling Gemini to generate a practice plan from zero signal is pure cost with no accuracy benefit"). The "What should I practice today?" chip in the Tutor UI calls M3's already-built deterministic generator, full stop — this is the single most likely place someone re-litigates scope mid-build, and it must not happen.
- **Gemini determining, validating, or influencing any note/chord/scale correctness, accuracy/timing/consistency score, lesson pass/fail, or XP/streak value** — the non-negotiable boundary (`docs/PRODUCT_SPEC.md` §14), unconditional, not softened anywhere in this milestone.
- **Any modification to `packages/music-engine`** — this milestone only ever *calls* the already-frozen engine's output (`ExerciseVerdict`) as read-only input to a prompt; it never touches the engine itself.
- **MIDI input, additional lesson content, achievements beyond streak/XP, the song learning module, offline PWA caching, admin CMS** — Phase 2+ (`docs/PRODUCT_SPEC.md` §7, §8), unrelated to this milestone regardless.
- **Account deletion / data export** — M5. Note specifically: once M4 ships, `aiMemory/summary` is a real document that must be included in both operations' full-subtree cascade — `docs/ROADMAP/M5_HARDENING.md` is responsible for extending `deleteAccount`/`exportUserData` to cover it; this milestone only needs to leave that document's path predictable and stable for M5 to pick up.

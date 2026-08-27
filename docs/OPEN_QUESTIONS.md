# Open Questions

## 1. Purpose

This is the single consolidated register of every unresolved ambiguity, missing requirement, or technical risk surfaced while writing this documentation set from `PRODUCT_SPEC.md`. It exists because the instruction governing this entire documentation pass was explicit: **document contradictions and gaps, never silently resolve them.** Every item below was found by one or more of the eight documentation sub-passes (PRODUCT/UX, ARCHITECTURE, ENGINEERING, MUSIC/CONTENT, AI, SECURITY, DATA/OPERATIONS, TESTING/ROADMAP) or by the final reconciliation pass across all of them, then deduplicated here.

Items that were found to already have a clean, unambiguous resolution once cross-checked against `PRODUCT_SPEC.md` and the shipped `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` were **corrected directly in the affected documents** (most in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`) rather than left open — those are recorded in `docs/DECISIONS.md` as **Resolved**, not listed here as open questions. What remains below are genuine judgment calls, missing numbers, or product decisions that need Linford's sign-off, not something a documentation pass has the authority to decide on its own.

## 2. How to use this document

- **BLOCKING** — an implementation agent literally cannot write correct code for this piece without a decision; guessing produces a wrong or arbitrary result baked into the system.
- **IMPORTANT** — affects correctness, security, or cost in a real way, but has a stated, reasonable, clearly-labeled default an implementation agent can proceed with with while flagging it was assumed, not confirmed.
- **NON-BLOCKING** — polish, future-proofing, or a cosmetic inconsistency in the source material. Safe to ignore for M0–M5.

Every item has a stable `OQ-##` ID. IDs are not resequenced when items are resolved — a resolved item's ID is retired (see `docs/DECISIONS.md`) rather than reused. Where a document elsewhere in `docs/` already references an `OQ-##` inline (e.g. `TYPES_AND_CONTRACTS.md`'s `Lesson.passThreshold` and `Lesson.hints` fields), that ID is authoritative and matches the entry here.

## 3. Summary table

| ID | Title | Severity | Primary doc(s) |
|---|---|---|---|
| OQ-01 | `Lesson.passThreshold` has no ratified default | BLOCKING | `ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2, `CONTENT/BEGINNER_CURRICULUM.md` §5 |
| OQ-14 | XP-award formula and streak grace-period semantics unspecified | BLOCKING | `ROADMAP/M3_MVP_CURRICULUM.md`, `ENGINEERING/TYPES_AND_CONTRACTS.md` §4.15 |
| OQ-15 | Gemini per-user daily rate-limit numbers and counter storage unspecified | BLOCKING | `AI/AI_COST_CONTROL.md` §5, `AI/GEMINI_SPEC.md` §10/§14 |
| OQ-02 | `Lesson.hints` tiering/repeat policy not ratified | IMPORTANT | `ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2, `CONTENT/CONTENT_AUTHORING_GUIDELINES.md` |
| OQ-03 | Chord-match fallback policy (per-note vs. all-or-nothing) | IMPORTANT | `MUSIC/CHORD_SYSTEM.md` §4.5/§6.1 |
| OQ-04 | `ExerciseVerdict.consistency` needs attempt history the engine's pure contract can't supply | IMPORTANT | `MUSIC/SCORING_SYSTEM.md` §4.4 |
| OQ-05 | Note-name octave-suffix display convention was inferred, not stated | IMPORTANT | `MUSIC/NOTE_SYSTEM.md` §4.5 |
| OQ-06 | `Score.breakdown` fields: raw or already-weighted? | IMPORTANT | `MUSIC/SCORING_SYSTEM.md` §4.5 |
| OQ-07 | Batch of unvalidated scoring numeric constants | IMPORTANT | `MUSIC/SCORING_SYSTEM.md` |
| OQ-08 | `packages/shared-types` vs. `packages/music-engine` type-ownership boundary | IMPORTANT | `ENGINEERING/PROJECT_STRUCTURE.md` §4.1 |
| OQ-09 | `deleteAccount` must explicitly cascade-delete subcollections | IMPORTANT | `DATA/DATA_RETENTION.md` §5 |
| OQ-16 | `MVP_SCOPE.md`'s Drills summary omits scale/chord drills that are actually in scope | IMPORTANT | `PRODUCT/MVP_SCOPE.md`, `ROADMAP/M3_MVP_CURRICULUM.md` |
| OQ-17 | Anonymous-account-cycling bypasses per-user Gemini rate limits | IMPORTANT | `AI/AI_COST_CONTROL.md` §7 |
| OQ-20 | No retention/cleanup policy for anonymous, never-linked user data | IMPORTANT | `DATA/DATA_RETENTION.md` §4 |
| OQ-21 | No callable Function named for Theory Explanation / Motivation-Nudge | IMPORTANT | `AI/GEMINI_SPEC.md` §4 |
| OQ-22 | No exact UI trigger specified for Theory Explanation / Motivation-Nudge | IMPORTANT | `AI/GEMINI_SPEC.md` §7/§8 |
| OQ-27 | Anonymous→authenticated linking when the target credential is already in use | IMPORTANT | (gap — not addressed in any doc; see §5 below) |
| OQ-10 | Sharps-only note spelling (no flats) for MVP | NON-BLOCKING | `MUSIC/NOTE_SYSTEM.md` |
| OQ-11 | MVP scale key count: 3 committed vs. "3–4" allowed | NON-BLOCKING | `CONTENT/BEGINNER_CURRICULUM.md` |
| OQ-12 | No explicit `matchMode` field; ordered/unordered is type-inferred | NON-BLOCKING | `MUSIC/SCALE_SYSTEM.md` §4.4 |
| OQ-13 | Minor input-layer details unspecified (key-remap validation, `event.code` vs `.key`, glissando, tempo bounds) | NON-BLOCKING | `MUSIC/INPUT_SYSTEM.md`, `MUSIC/KEYBOARD_MAPPING.md` |
| OQ-18 | Three ROADMAP milestone filenames are narrower than their actual scope | NON-BLOCKING | `ROADMAP/M1_MUSIC_ENGINE.md`, `M3_MVP_CURRICULUM.md`, `M4_GEMINI_TUTOR.md` |
| OQ-19 | CI/CD provider and monorepo build-tool not chosen | NON-BLOCKING | `ENGINEERING/TECH_STACK.md`, `PROJECT_STRUCTURE.md` |
| OQ-23 | `learnerLevel` vs. `skillLevel` field-naming inconsistency | NON-BLOCKING | `ENGINEERING/TYPES_AND_CONTRACTS.md` §4.17 |
| OQ-24 | "Cheap" vs. "Cheapest" cost-tier naming inconsistency in `PRODUCT_SPEC.md` | NON-BLOCKING | `AI/GEMINI_SPEC.md` §14 |
| OQ-25 | `PRODUCT_SPEC.md` §5 tags Daily Practice Plan `[MVP]` unqualified; §6/§7 correctly scope only the rule-based version to MVP | NON-BLOCKING | `AI/GEMINI_SPEC.md` §14 |
| OQ-26 | No proposed home named for onboarding-answer processing logic | NON-BLOCKING | `ENGINEERING/PROJECT_STRUCTURE.md` |
| OQ-28 | `SECURITY_TESTING.md` claims App Check test coverage that does not actually exist anywhere | IMPORTANT | `TESTING/SECURITY_TESTING.md` §10, `SECURITY/APP_CHECK.md` §10 |
| OQ-29 | Firestore offline persistence (PR-PLATFORM-04) is fully specified but scheduled in no milestone and tested nowhere | IMPORTANT | `ENGINEERING/ERROR_HANDLING.md` §5.2, `ROADMAP/` (no M-file covers it) |
| OQ-30 | `tempo` vs `tempoBpm` field-naming mismatch between spec and engineering docs | NON-BLOCKING | `PRODUCT_SPEC.md` §17, `ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 |
| OQ-31 | `ExerciseVerdict` uses string note names despite MIDI-canonical rule | NON-BLOCKING | `ENGINEERING/TYPES_AND_CONTRACTS.md` §3, §4.3 |

## 4. BLOCKING

### OQ-01 — `Lesson.passThreshold` has no ratified default

`TYPES_AND_CONTRACTS.md` §4.2 requires every `Lesson` to carry a `passThreshold: number` (0–1) but `PRODUCT_SPEC.md` never gives a number, and no default has been ratified. This blocks any real lesson-completion/unlock logic and the M2 acceptance criteria that depend on it.

**Recommendation on record (not ratified):** `0.8` uniformly across all MVP lessons — strict enough to mean something, forgiving enough for a genuine beginner (`CONTENT/BEGINNER_CURRICULUM.md` §5). Every scoring worked example in `MUSIC/SCORING_SYSTEM.md` and every seed lesson in `CONTENT/` uses `0.8` consistently, so switching to a ratified value later is a single-constant change, not a rewrite.

**Needs:** Linford to confirm `0.8`, or specify a different value (uniform or per-lesson).

### OQ-14 — XP-award formula and streak grace-period semantics unspecified

`UserProfile.streak`/`UserProfile.xp` exist as fields (`TYPES_AND_CONTRACTS.md` §4.15), and streak logic is stated to be "calendar-day based," but nothing in `PRODUCT_SPEC.md` or `TYPES_AND_CONTRACTS.md` specifies: how much XP a given exercise/lesson award is worth, whether missing exactly one calendar day breaks a streak or grants a grace day, or how a streak's `lastPracticeDate` reconciles across timezones for a learner who practices near midnight.

**Mitigated, not resolved:** `ROADMAP/M3_MVP_CURRICULUM.md` requires the mechanism be built behind two named, isolated functions — `computeXpAward()` and `applyStreakUpdate()` — specifically so the actual formula/thresholds can be dropped in without restructuring the calling code once decided. `ROADMAP/DEVELOPMENT_METHODOLOGY.md` §5 uses this exact gap as its own worked example of "flag, don't invent."

**Needs:** Linford to specify an XP formula (even a simple flat-rate one) and streak grace-period rule before M3 can be considered complete.

### OQ-15 — Gemini per-user daily rate-limit numbers and counter storage unspecified

`PRODUCT_SPEC.md` §19/§29 require "a daily cap on tutor calls for free-tier users" and an acceptance criterion that per-user rate limiting "demonstrably blocks a scripted burst... beyond the configured cap" — but never states the actual number, and no field exists anywhere in `TYPES_AND_CONTRACTS.md` to persist a per-day counter.

**Recommendation on record (not ratified):** `AI/AI_COST_CONTROL.md` §5 proposes starting values — **20 tutor calls/day + 100 exercise-feedback calls/day for authenticated users; 5 tutor calls/day + 25 exercise-feedback calls/day for anonymous users** (anonymous gets a materially tighter cap given OQ-17 below). Storage: a Function-only-written counter, either a field on `UserProfile` or a small `users/{uid}/aiUsage/{date}` subdocument — not yet chosen (see D-11 in `DECISIONS.md`).

**Needs:** Linford to ratify the numbers (or supply different ones) and confirm the counter's storage shape before M4 rate-limiting can be implemented and before `PRODUCT_SPEC.md` §29's rate-limit acceptance criterion can be tested.

## 5. IMPORTANT

### OQ-02 — `Lesson.hints` tiering/repeat policy not ratified

`PR-LESSON-05` requires escalating hints as an MVP feature, but no field existed in `TYPES_AND_CONTRACTS.md` to represent it — this documentation pass added `hints?: string[]` (§4.2) to close that structural gap, but the exact authoring policy (how many tiers, what happens when a learner exhausts all hints) is not ratified. **Recommendation on record:** 3 tiers, the last hint repeats on further requests (`CONTENT/CONTENT_AUTHORING_GUIDELINES.md`). Needs sign-off before real lesson content is authored at scale.

### OQ-03 — Chord-match fallback policy

When a chord-drill attempt doesn't match any known chord, should `correctNotes`/`missedNotes`/`extraNotes` fall back to a per-note comparison against the target chord (partial credit, more pedagogically forgiving), or report all-or-nothing (simpler, arguably clearer feedback)? **Recommendation on record:** per-note fallback (`MUSIC/CHORD_SYSTEM.md` §4.5/§6.1), consistent with how the scale/melody scoring already works. Needs sign-off before `packages/music-engine`'s chord-scoring path is implemented.

### OQ-04 — Consistency scoring needs attempt history the engine's pure contract can't supply

`ExerciseVerdict.consistency` is defined as "variance across this + recent attempts of the same exercise," but the Music Engine's stated contract is a pure function of `(NoteEvent[], Lesson | Exercise) → ExerciseVerdict` — it has no access to prior attempts. **Proposed resolution on record:** a second, small exported function that takes recent attempt history as an explicit parameter, called by the Cloud Function after loading that history from `PracticeSession` (`MUSIC/SCORING_SYSTEM.md` §4.4). This keeps the core `evaluateAttempt` function pure while still producing the field. Needs sign-off on the exact signature before implementation.

### OQ-05 — Note-name octave-suffix display convention was inferred

Whether a note name renders as bare `"C"` or octave-qualified `"C4"` in `ExerciseVerdict.correctNotes`/`missedNotes`/`extraNotes` was not stated anywhere in `PRODUCT_SPEC.md` — it was inferred from the mandatory canonical worked example (`missedNotes: ['F']`, bare names) and then extended into a full rule for cases the example doesn't cover (`MUSIC/NOTE_SYSTEM.md` §4.5: bare pitch-class names in the common case, octave-qualified only when a verdict spans the same pitch class at two octaves within one attempt). This is binding on every `ExerciseVerdict`-producing code path once implemented. Needs sign-off — a plausible, well-reasoned inference is still an inference, not a stated requirement.

### OQ-06 — `Score.breakdown` fields: raw or weighted?

`Score.breakdown.{correctness,timing,consistency}` (`TYPES_AND_CONTRACTS.md` §4.14) could reasonably mean either the raw 0–1 sub-scores before `ScoringWeights` is applied, or the already-weighted contribution each makes to `Score.value`. The two readings produce different numbers for any UI that renders the breakdown directly (e.g. a progress-detail screen). `MUSIC/SCORING_SYSTEM.md` §4.5 flags this without picking a side. Needs a decision before any UI consumes `breakdown` directly rather than just `value`.

### OQ-07 — Batch of unvalidated scoring numeric constants

Several specific numbers in `MUSIC/SCORING_SYSTEM.md` are clearly labeled PROPOSED, not validated against any real usage data: the 150ms chord-note-clustering window (how close together played notes must be to count as "one chord attempt" rather than sequential single notes), the timing-tolerance bucket point values (1.0 / 0.5 / 0 / 0 for onTime/slightlyOff/late/early), the consistency formula's 5-attempt lookback window and 200ms normalization constant, the extra-notes accuracy-penalty formula, and the 600ms note-spacing used for untimed demo/reference playback. None of these are wrong, exactly — they're reasonable engineering placeholders — but none have been validated against how real beginners actually play. Flagged as a batch because they're the first thing worth revisiting once M2/M3 produce real `PracticeSession` data, not because any one of them is individually blocking.

### OQ-08 — `packages/shared-types` vs. `packages/music-engine` type-ownership boundary

`TYPES_AND_CONTRACTS.md` states its types are exported from `packages/shared-types`, but several of them (`NoteEvent`, `ChordDefinition`, `ScaleDefinition`) are also described as owned/defined by `packages/music-engine`'s own logic in `MUSIC/MUSIC_ENGINE_SPEC.md` and related files. `ENGINEERING/PROJECT_STRUCTURE.md` §4.1 flags the ambiguity without fully resolving which package is the source export and which merely re-exports or imports. Affects M1's exact file layout; low risk either way but should be settled before M1 starts rather than improvised mid-milestone.

### OQ-09 — `deleteAccount` must explicitly cascade-delete subcollections

Not an open decision so much as a flagged implementation risk: Firestore does not recursively delete subcollections when a parent document is deleted. `PRODUCT_SPEC.md` §18's "full data subtree" language could be misread as "delete `users/{uid}` and you're done" — that would silently leave `practiceSessions/*` and `aiMemory/summary` orphaned, still stored, still billed, and still technically queryable. `DATA/DATA_RETENTION.md` §5 already gives a complete proposed operation order (verify identity → delete `practiceSessions` → delete `aiMemory/summary` → delete `users/{uid}` → delete the Firebase Auth record → return success) specifically to prevent this. Listed as IMPORTANT because it's an easy silent correctness/compliance bug to introduce at implementation time even with the right document in hand, not because the answer is actually unknown.

### OQ-16 — `MVP_SCOPE.md`'s Drills summary omits scale/chord drills

`PRODUCT/MVP_SCOPE.md`'s summary table names only "Note Identification game" under Drills, while `PRODUCT_SPEC.md` §10 and `TYPES_AND_CONTRACTS.md`'s `Exercise.kind: 'noteId' | 'scaleDrill' | 'chordDrill'` both independently commit to all three. `ROADMAP/M3_MVP_CURRICULUM.md` already builds all three, reading the summary table's omission as elaboration rather than a real scope contradiction — but a reader who only opens `MVP_SCOPE.md` would be misled. **Recommended fix (not yet made):** a one-line edit to `MVP_SCOPE.md`'s Drills row to list all three explicitly.

### OQ-17 — Anonymous-account-cycling bypasses per-user Gemini rate limits

`PRODUCT_SPEC.md` §18's zero-friction anonymous sign-in means a scripted loop that creates a fresh anonymous session per request gets a fresh per-user rate-limit quota every time (OQ-15's cap is keyed on `uid`, and a new anonymous `uid` is free to create). Not addressed anywhere in `PRODUCT_SPEC.md`. **Proposed mitigation on record:** `AI/AI_COST_CONTROL.md` §7 — layering a device/IP-level or App-Check-attestation-level secondary limit on top of the per-`uid` cap specifically for anonymous users. Needs security/engineering sign-off before this is treated as sufficiently mitigated for launch.

### OQ-20 — No retention/cleanup policy for anonymous, never-linked user data

A visitor who plays anonymously and never signs up leaves a real `users/{uid}` document (and possibly `practiceSessions`/`aiMemory`) with no owner ever confirmed to return, and `PRODUCT_SPEC.md` never addresses this case. `DATA/DATA_RETENTION.md` §4 lays out why this matters (unbounded cost growth uncorrelated with real signups; POPIA/GDPR exposure from indefinite retention with no retention justification, given `PRODUCT_SPEC.md` §18's own framing that this product must handle EU/UK and South African users) and presents two candidate resolutions without picking one: (A) no automatic cleanup in MVP, matching the low-stakes default already proposed for `practiceSessions` retention generally; (B) a scheduled Cloud Function that purges inactive anonymous subtrees after N days (the "scheduled jobs" capability is already named in the architecture). **Needs Linford's decision before public launch** — this does not block the M0–M5 internal-testing build, since a handful of internal test accounts carries none of the real exposure described above.

### OQ-21 — No callable Function named for Theory Explanation / Motivation-Nudge

`PRODUCT_SPEC.md` §20 names exactly two Gemini-touching callables (`getExerciseFeedback`, `getTutorResponse`) even though §15 defines five prompt templates. `AI/GEMINI_SPEC.md` §4 proposes a resolution: Theory Explanation is a deterministic-phrase-match-routed mode of `getTutorResponse` (not a separate callable), and Motivation/Nudge's rare live-call path is folded into `getExerciseFeedback`'s response as an optional field (not a separate callable either) — because the volume for both doesn't justify a dedicated Function, and because Motivation/Nudge is primarily pool-served client-side and rarely reaches the server at all. Needs engineering sign-off before `functions/src/ai/` is structured around this mapping.

### OQ-22 — No exact UI trigger specified for Theory Explanation / Motivation-Nudge

Related to OQ-21 but a UX question, not a backend-routing one: `PRODUCT_SPEC.md` §10 doesn't specify exactly how a learner reaches a theory explanation (a specific screen element? a quick-suggestion chip inside the Tutor chat, as `GEMINI_SPEC.md` §7 proposes?) or how a Motivation/Nudge line is surfaced (a dashboard card? a toast?). `GEMINI_SPEC.md` §7/§8 propose concrete mechanisms consistent with the rest of the UX spec; needs product/UX confirmation, ideally cross-checked against `UX/SCREEN_SPECIFICATIONS.md`'s existing Tutor/Dashboard screen specs before M4 UI work starts.

### OQ-27 — Anonymous → authenticated linking when the target credential is already in use

`PRODUCT_SPEC.md` §18 describes the `linkWithCredential` upgrade path for converting an anonymous session into a real account, but does not address what happens when the email/credential the learner tries to link already belongs to a separate, existing account — a real case Firebase Auth surfaces as an `auth/credential-already-in-use` error, not a hypothetical edge case invented for this document. If it isn't handled explicitly, the anonymous session's local progress (lessons completed, streak, XP) has nowhere to go: it cannot silently merge into the existing account without an explicit decision about how conflicting progress reconciles, and the linking call will simply fail if the UI doesn't handle this error path at all. **This gap was not surfaced by any of the eight documentation sub-passes — it's raised here directly, based on how Firebase Auth's linking API is known to behave, because it sits squarely inside "non-negotiable day-one" auth scope per §18.** Needs a product decision: block the link and prompt the learner to sign in to their existing account instead (discarding or explicitly offering to discard the anonymous session's progress, with a clear warning before it happens), or build an explicit merge flow. A merge flow is materially more implementation work and should not be assumed as the default without confirmation.

### OQ-28 — `SECURITY_TESTING.md` claims App Check test coverage that doesn't exist

`docs/TESTING/SECURITY_TESTING.md` §10 states App Check enforcement (SEC-15/SEC-16) is covered by `AI_TESTING.md`'s rate-limiting scenarios and `INTEGRATION_TESTING.md`'s Function tests. It isn't — neither file contains any App-Check-specific assertion; a concrete test plan exists only in `docs/SECURITY/APP_CHECK.md` §10 and was never wired into `docs/TESTING/`. Raised as IMPORTANT rather than a routine gap because a **false claim of coverage is worse than an honestly-flagged absence** — it's the kind of thing that reads as "done" on review and isn't. Found by the final requirements-traceability pass; not yet fixed, since closing it means authoring real App-Check test cases (new content), not a documentation-consistency correction. **Needs:** either a new test section added to `AI_TESTING.md`/`INTEGRATION_TESTING.md` implementing `APP_CHECK.md` §10's plan, or `SECURITY_TESTING.md` §10 corrected to stop claiming coverage it doesn't have — either way, before M4/M5 sign-off, not after.

### OQ-29 — Firestore offline persistence is specified but unscheduled and untested

`PR-PLATFORM-04` (Firestore offline persistence) is MVP-tagged and fully mechanism-specified in `docs/ENGINEERING/ERROR_HANDLING.md` §5.2 — exact API call (`enableIndexedDbPersistence`), exact banner copy ("Playing offline — your progress will sync later"). But **no milestone file (M0–M5) actually schedules building it**, and no test in `docs/TESTING/` covers it, despite `ERROR_HANDLING.md` §8 itself prescribing the missing test ("simulate a disconnected client, assert the degraded banner appears... simulate reconnect and assert queued writes flush"). The Gemini-unavailable half of the same "never a dead end" principle (§5.1) is fully scheduled (M4) and tested (`E2E_TESTING.md` §5.3) — this is specifically the Firestore-offline half falling through the gap. Found by the final requirements-traceability pass. **Needs:** an explicit home in one of the M0–M5 milestone files (M1 or M5 are the most natural fits — M1 because the piano/lesson UI that needs to keep working offline is built there, M5 because it's a hardening-pass concern) and a corresponding test added to `docs/TESTING/`.

## 6. NON-BLOCKING

### OQ-10 — Sharps-only note spelling

`MUSIC/NOTE_SYSTEM.md` renders all accidentals as sharps for MVP display (no flat spellings). This constrains any future lesson content written in a flat-heavy key, but the committed MVP curriculum (C/G/D/A major) needs no flats at all, so it isn't a real constraint today.

### OQ-11 — MVP scale key count: 3 committed vs. "3–4" allowed

`PRODUCT_SPEC.md` leaves the exact additional-key count at "3–4"; `CONTENT/BEGINNER_CURRICULUM.md` commits to exactly 3 (G, D, A major) to land a clean 14-lesson curriculum. A 4th key (E major) remains a low-risk future addition within the range PRODUCT_SPEC.md already allows, not a scope violation.

### OQ-12 — No explicit `matchMode` field; ordered/unordered is type-inferred

Whether a lesson's expected note sequence must be matched in order or as an unordered set is currently inferred from `Lesson.type` (e.g. `'chord'` → unordered, `'melody'` → ordered) rather than an explicit field (`MUSIC/SCALE_SYSTEM.md` §4.4). The inference rule is fully specified and consistent; an explicit `matchMode` field would be more robust against a future lesson type that doesn't fit the pattern cleanly, but isn't needed for MVP's fixed set of types.

### OQ-13 — Minor input-layer details unspecified

A handful of low-stakes details have no stated policy: exact keyboard-remap validation rules (can a learner map two physical keys to the same note?), whether the computer-keyboard adapter should key off `event.code` (physical position) or `event.key` (layout-dependent character), mouse-drag/glissando behavior on the virtual keyboard, and sanity bounds on `Lesson.tempoBpm`. None of these block M0–M2; each has an obvious, low-risk default an implementer can pick without product input.

### OQ-18 — Three ROADMAP milestone filenames are narrower than their actual scope

`ROADMAP/M1_MUSIC_ENGINE.md`'s actual scope is the full playable-piano stack (input, audio, engine, and UI), broader than its filename suggests relative to `PRODUCT_SPEC.md` §28's own milestone name; `M3_MVP_CURRICULUM.md`'s filename omits the Note ID game, which §28 includes in the same milestone; `M4_GEMINI_TUTOR.md` is named for the one AI provider this product uses, which §28 calls simply "AI tutor." Each file notes its own discrepancy inline. Purely a naming/cosmetic mismatch against the exact filenames this documentation pass was asked to produce — the scope content inside each file is faithful to §28.

### OQ-19 — CI/CD provider and monorepo build-tool not chosen

`PRODUCT_SPEC.md` never names a CI/CD provider or a monorepo build-orchestration tool. `ENGINEERING/TECH_STACK.md`/`PROJECT_STRUCTURE.md` already carry a recommendation: plain npm/pnpm workspaces, no Turborepo/Nx added speculatively, given the project's current size. Revisit if the monorepo grows enough that uncached cross-package rebuilds become a real time cost.

### OQ-23 — `learnerLevel` vs. `skillLevel` field-naming inconsistency

Some prompt-input prose (predating this pass's `ExerciseFeedbackPromptInput` type) used `learnerLevel`; the canonical field across `AIMemorySummary`/`TutorRequest` is `skillLevel`. `TYPES_AND_CONTRACTS.md` §4.17 now documents `skillLevel` as canonical and `learnerLevel` as a pre-existing synonym for the same concept — substantially closed by that formalization, listed here only so the historical naming in `AI_PROMPTS.md`'s worked example text isn't mistaken for a second, different field.

### OQ-24 — "Cheap" vs. "Cheapest" cost-tier naming inconsistency

`PRODUCT_SPEC.md` §15's own cost-tier table labels Theory Explanation "Cheap" where every other reference — including the canonical `TutorResponse.tier` enum, which has only two values (`'cheapest' | 'mid'`) — says "Cheapest." Treated as synonymous throughout `docs/`. A copy-edit fix in the source spec, not a design question.

### OQ-25 — Daily Practice Plan MVP-tagging inconsistency in `PRODUCT_SPEC.md`

`PRODUCT_SPEC.md` §5 tags "one daily practice session generated for me" as unqualified `[MVP]`, while §6/§7 correctly clarify that only the deterministic rule-based version ships in MVP and the Gemini-personalized version is Phase 2. Every downstream document in this set treats Daily Practice Plan as Phase 2/Future consistently — flagged here only because it's a genuine inconsistency inside the source document itself, not because any confusion propagated.

### OQ-26 — No proposed home named for onboarding-answer processing logic

`PRODUCT_SPEC.md` describes an onboarding flow that asks a new learner a few questions, but `ENGINEERING/PROJECT_STRUCTURE.md` doesn't name a specific file/module for the Cloud Function logic that processes those answers into initial `UserProfile`/`AIMemorySummary` state. A reasonable, unratified default (`functions/src/account/completeOnboarding.ts`) has been suggested but not confirmed.

### OQ-30 — `tempo` vs `tempoBpm` field-naming mismatch between spec and engineering docs

`PRODUCT_SPEC.md` §17 lists `tempo` as a `Lesson` field; `TYPES_AND_CONTRACTS.md` §4.2 defines it as `tempoBpm`. The engineering name is more explicit and consistent with the project's convention of qualifying units in field names, but the spec uses the shorter form. An implementation agent reading both documents would encounter two different field names for the same concept. **Recommendation:** use `tempoBpm` (the engineering-doc name) as canonical — it's already the field name in every downstream document (`LESSON_CONTENT_SCHEMA.md`, `MUSIC_ENGINE_SPEC.md`, `BEGINNER_CURRICULUM.md`). A copy-edit fix in the source spec, not a design question.

### OQ-31 — `ExerciseVerdict` uses string note names despite MIDI-canonical rule

`TYPES_AND_CONTRACTS.md` §3 establishes that "MIDI note numbers, not note-name strings, are the canonical representation of pitch everywhere in this codebase." However, `ExerciseVerdict` (`TYPES_AND_CONTRACTS.md` §4.3) defines `correctNotes: string[]`, `missedNotes: string[]`, and `extraNotes: string[]` — all using human-readable note-name strings (e.g. `"F"`, `"C4"`), not MIDI numbers. This is an unacknowledged exception to the stated rule. The likely rationale is that `ExerciseVerdict` is the one type that crosses the server/client boundary specifically for display purposes (the Lesson Player renders these names directly), so human-readable strings are the right choice here — but the §3 rule doesn't document this exception. **Recommendation:** add an inline note to §3 acknowledging that `ExerciseVerdict`'s note-name fields are a deliberate display-layer exception to the MIDI-canonical rule, not an oversight.

## 7. Out of scope

- Fully resolved items are recorded in `docs/DECISIONS.md`, not here.
- This document does not track implementation bugs — only specification-level ambiguity, contradiction, or missing information discovered while writing `docs/`.

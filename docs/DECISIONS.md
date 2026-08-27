# Decisions

## 1. Purpose

This is the project's decision log — the record of every choice made or proposed while turning `PRODUCT_SPEC.md` into the full `docs/` documentation system, in the format specified for this effort: Decision ID, Date, Decision, Context, Options considered, Decision made, Reason, Consequences, Status.

Two kinds of entries appear here, and the **Status** field always distinguishes them:

- **Status: Resolved** — an objective internal-consistency correction. These were not product judgment calls; they were cases where this documentation set contradicted itself (usually against `PRODUCT_SPEC.md` or the shipped `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`, both treated as ground truth) and the correct fix was unambiguous once traced to its source. Made directly, not left open, because leaving an unambiguous internal contradiction "open" would just be a second form of silent inconsistency.
- **Status: Proposed** — a genuine product/engineering judgment call where `PRODUCT_SPEC.md` is silent. A specific, reasoned default is recorded so implementation isn't blocked, but **none of these are ratified** — they need Linford's explicit sign-off before an implementation agent should treat them as final. Every Proposed entry also has a matching `OQ-##` entry in `docs/OPEN_QUESTIONS.md`.
- **Status: Open** — no default is proposed at all, because even the "do nothing" option was judged too consequential to pick silently.

No entry in this log describes a decision that predates this documentation effort — nothing here is a fabricated project history. Every entry was made or surfaced while writing `docs/` from `PRODUCT_SPEC.md`, on 2026-08-27.

## 2. Resolved — internal-consistency corrections

### D-01 — `AIMemorySummary` client read access

**Date:** 2026-08-27
**Decision:** `AIMemorySummary` (`users/{uid}/aiMemory/summary`) is directly readable client-side, owner-only. Writes remain Cloud-Functions-only under all circumstances.
**Context:** `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8 originally stated `AIMemorySummary` is "never read... directly by the client." This contradicted both `docs/PRODUCT_SPEC.md` §19 (which describes `aiMemory/**` as readable by the owning uid) and the shipped `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` §5.4 (`allow read: if isOwner(uid)` on this exact path). Multiple independent documentation passes (AI, SECURITY, TESTING/ROADMAP, and the final traceability pass) all surfaced this same contradiction.
**Options considered:** (a) keep `TYPES_AND_CONTRACTS.md`'s original wording and change `PRODUCT_SPEC.md`'s intent + the shipped rules file to match — rejected, since `PRODUCT_SPEC.md` is the project's ground truth and the rules file is the actual working artifact everything else is instructed to test against; overriding both to match a single contracts-file sentence would be backwards. (b) Correct `TYPES_AND_CONTRACTS.md` to match the other two — adopted.
**Decision made:** Option (b). `TYPES_AND_CONTRACTS.md` §4.8 now states owner-only client read access, with an inline note explaining the correction.
**Reason:** Two independently-authoritative sources (the ground-truth spec and the actual shipped security rules) agreed; only the contracts file, a single author's error, disagreed.
**Consequences:** `docs/AI/AI_ERROR_HANDLING.md` §5.2, `docs/ROADMAP/M4_GEMINI_TUTOR.md` §4, and `docs/ROADMAP/M5_HARDENING.md` §11 all previously narrated this as an open, unresolved contradiction — all three have been updated in this same pass to reflect the resolution. A richer AI-Tutor-unavailable fallback (surfacing `AIMemorySummary.weakNotes` directly) is now buildable without waiting on this question; MVP fallback UI still uses only `ProgressSummary` regardless (`AI_ERROR_HANDLING.md` §5.2).
**Status:** Resolved.

### D-02 — `practiceSessions` write-cooldown enforcement location

**Date:** 2026-08-27
**Decision:** The exercise-submission abuse-prevention cooldown is enforced inside the `getExerciseFeedback` Cloud Function (application code), not as a Firestore Security Rule.
**Context:** `docs/PRODUCT_SPEC.md` §19's prose describes the cooldown as "a Firestore rule," but `practiceSessions` is Admin-SDK/Cloud-Functions-write-only (per `TYPES_AND_CONTRACTS.md` §4.6) — Admin SDK writes bypass Firestore Security Rules entirely, so a rule can never actually enforce this.
**Options considered:** (a) Make `practiceSessions` writable directly by the client so a Firestore rule could apply — rejected outright, since it would reopen the exact "client self-reports a perfect score" cheating vector this collection's Function-only-write rule exists to close. (b) Enforce the cooldown as application logic inside the Function, before the write. — adopted.
**Decision made:** Option (b), independently arrived at by both the SECURITY and DATA/OPERATIONS documentation passes.
**Reason:** Only option that preserves both the abuse-prevention goal and the Function-only-write trust boundary.
**Consequences:** `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` §5.5 documents the cooldown as Function-side logic, not a rule; `docs/TESTING/SECURITY_TESTING.md` §5.6 and `docs/TESTING/INTEGRATION_TESTING.md` §5 assign the corresponding test to integration testing, not rules-unit-testing. The exact cooldown interval (proposed: 5 seconds, `FIRESTORE_SECURITY_RULES.md`) remains a separate, non-blocking open item — see `docs/OPEN_QUESTIONS.md`.
**Status:** Resolved.

### D-03 — `ExerciseFeedbackPromptInput` and `MotivationNudgePromptInput` formalized as named types

**Date:** 2026-08-27
**Decision:** Two prompt-input shapes that existed only as JSON copy-pasted identically across `PRODUCT_SPEC.md` §15, `docs/ARCHITECTURE/DATA_FLOW.md`, `docs/AI/AI_PROMPTS.md`, and `docs/AI/GEMINI_SPEC.md` are now named types in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.17–4.18.
**Context:** `docs/AI/GEMINI_SPEC.md` §12 explicitly recommended this ("recommend they be formally added there... rather than continuing to exist only as copied JSON shapes across documents").
**Options considered:** (a) Leave the shape informal, repeated in prose across four documents — rejected, since a field added or renamed in one copy has no mechanism to stay in sync with the other three. (b) Formalize as canonical types. — adopted.
**Decision made:** Option (b).
**Reason:** `TYPES_AND_CONTRACTS.md` §1 states its own purpose as "the single canonical source for every shared data shape" — an informally-repeated shape used by two Cloud Functions was a gap against that stated purpose.
**Consequences:** `ExerciseFeedbackPromptInput.skillLevel` is now the canonical field name; existing prose using `learnerLevel` for the same concept is documented as a pre-existing synonym, not a second field (see `docs/OPEN_QUESTIONS.md` OQ-23).
**Status:** Resolved.

### D-04 — `FunctionResult<T>` / `FunctionErrorCode` formalized, and unified across three documents

**Date:** 2026-08-27
**Decision:** `FunctionResult<T>` and a six-value `FunctionErrorCode` union (`UNAUTHENTICATED`, `VALIDATION_FAILED`, `RATE_LIMITED`, `GEMINI_UNAVAILABLE`, `NOT_FOUND`, `INTERNAL`) are now formalized in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.16, using `docs/ENGINEERING/ERROR_HANDLING.md` §4.3's naming as the canonical source.
**Context:** `docs/PRODUCT_SPEC.md` §20 established the `{ok,data}`/`{ok,error}` shape in prose; `ERROR_HANDLING.md` §4 worked out a full six-code taxonomy but flagged that no document exported it as a real type. During this QC pass, a first attempt to formalize it in `TYPES_AND_CONTRACTS.md` invented different names for three of the six codes (`INVALID_ARGUMENT`, `AI_UPSTREAM_ERROR` in place of `VALIDATION_FAILED`, `GEMINI_UNAVAILABLE`) without checking prior art — and `docs/AI/AI_ERROR_HANDLING.md` had separately, independently proposed `RATE_LIMIT_EXCEEDED` where `ERROR_HANDLING.md` used `RATE_LIMITED`, self-flagging that it would need reconciliation once a canonical taxonomy landed. The final traceability pass caught all of this as a three-way naming conflict.
**Options considered:** (a) Pick a fourth, "best of all three" naming — rejected as unnecessary churn across the most documents. (b) Standardize on `ERROR_HANDLING.md`'s original names, since it's the project-wide taxonomy document (more general and more complete than the AI-specific or contracts-file versions) and `AI_ERROR_HANDLING.md` had already explicitly deferred to whatever it landed on. — adopted.
**Decision made:** Option (b). `TYPES_AND_CONTRACTS.md` §4.16 and `docs/AI/AI_ERROR_HANDLING.md` §6 were both corrected to use `ERROR_HANDLING.md`'s exact code names.
**Reason:** Minimizes total edits, follows the document that was both most complete and explicitly deferred to by the other two.
**Consequences:** Seven files touched in total to close every downstream reference: `TYPES_AND_CONTRACTS.md`, `ERROR_HANDLING.md`, `AI_ERROR_HANDLING.md`, `AI_COST_CONTROL.md`, `AI_GUARDRAILS.md`, `GEMINI_SPEC.md`, `REQUIREMENTS_TRACEABILITY.md`. An implementer can now use any one of these documents' code names and get an identical result — this was not true before this pass.
**Status:** Resolved.

### D-14 — Field-scoped `users/{uid}` write rule (not a separate `settings` sub-path)

**Date:** 2026-08-27
**Decision:** `UserProfile.settings` is a field embedded within the single `users/{uid}` document (client-writable, own uid only), not a separate Firestore sub-path — enforced via a field-scoped Security Rule (`diff().affectedKeys().hasOnly(['settings'])`-style check), not a path-level rule.
**Context:** `docs/PRODUCT_SPEC.md` §19's prose could be read as implying `users/{uid}/settings` is a distinct sub-document; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` always modeled `settings` as an embedded field on the same document as every Function-only-written field. The ARCHITECTURE documentation pass flagged this as needing explicit confirmation.
**Options considered:** (a) Split `settings` into its own document/sub-path so a simple path-level rule could apply — rejected as an unnecessary schema change with no benefit, and a second document per user for three small preference fields. (b) Keep `settings` embedded, enforce the client-writable/Function-only-writable boundary with a field-scoped rule. — adopted (this was already how `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` was written).
**Decision made:** Option (b), confirmed correct after cross-checking `FIRESTORE_SECURITY_RULES.md`'s actual implementation against `TYPES_AND_CONTRACTS.md` §4.15's field-level ownership table — both already agreed; this entry formally closes the "needs explicit confirmation" flag rather than changing anything.
**Reason:** No functional benefit to a schema split; the field-scoped rule is simpler and already correctly implemented.
**Consequences:** None — no file changes required, only confirmation.
**Status:** Resolved.

## 3. Proposed — needs Linford's sign-off

### D-05 — `Lesson.passThreshold` default value

**Date:** 2026-08-27
**Decision:** Proposed default: **0.8**, uniformly across every MVP lesson.
**Context:** `TYPES_AND_CONTRACTS.md` §4.2 requires this field; `PRODUCT_SPEC.md` never gives a number. See `docs/OPEN_QUESTIONS.md` OQ-01 (BLOCKING).
**Options considered:** A single uniform threshold across all lessons vs. a per-lesson-authored threshold. A uniform value was chosen for MVP simplicity; per-lesson tuning is possible later without a schema change (the field is already per-`Lesson`).
**Decision made:** 0.8 uniform, proposed by the MUSIC/CONTENT documentation pass and used consistently in every worked scoring example and seed lesson in `docs/MUSIC/` and `docs/CONTENT/`.
**Reason:** Strict enough to mean something as a completion gate; forgiving enough that a genuine beginner isn't blocked by an unreasonably high bar.
**Consequences:** If Linford ratifies a different value, it is a single-constant change — no document structure depends on 0.8 specifically, only on the field existing.
**Status:** Proposed.

### D-06 — `Lesson.hints` tiering and repeat policy

**Date:** 2026-08-27
**Decision:** Proposed: 3 hint tiers per lesson; the third/last hint repeats on further requests rather than the UI running out of hints to show.
**Context:** `PR-LESSON-05` requires escalating hints; no field existed to represent this until this QC pass added `Lesson.hints?: string[]`. See OQ-02 (IMPORTANT).
**Options considered:** A fixed tier count (2, 3, or 4) vs. a variable count per lesson. A fixed count of 3 was chosen for authoring consistency across the ~12–15 MVP lessons.
**Decision made:** 3 tiers, last repeats — proposed by the CONTENT documentation pass.
**Reason:** Matches common escalating-hint patterns (nudge → more specific nudge → near-answer) without over-complicating content authoring for a small MVP curriculum.
**Consequences:** `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` is written against this assumption; a different tier count is a content-authoring-guide edit, not a schema change (the field is `string[]`, any length).
**Status:** Proposed.

### D-07 — Chord-match scoring fallback

**Date:** 2026-08-27
**Decision:** Proposed: when a chord-drill attempt fails to match any known chord, `correctNotes`/`missedNotes`/`extraNotes` fall back to a per-note comparison against the target chord, rather than an all-or-nothing verdict.
**Context:** See OQ-03 (IMPORTANT).
**Options considered:** Per-note fallback (partial credit, more forgiving feedback) vs. all-or-nothing (simpler, arguably clearer). Per-note fallback was chosen for consistency with how scale/melody exercises already score partial attempts.
**Decision made:** Per-note fallback — proposed by the MUSIC documentation pass, `docs/MUSIC/CHORD_SYSTEM.md` §4.5/§6.1.
**Reason:** Pedagogical consistency with the rest of the scoring system; an all-or-nothing chord verdict would be a different feedback style than every other exercise type gets.
**Consequences:** `packages/music-engine`'s chord-scoring path must implement the fallback, not just a boolean match/no-match check.
**Status:** Proposed.

### D-08 — Consistency-scoring needs a second, history-aware function

**Date:** 2026-08-27
**Decision:** Proposed: add a second small exported function from `packages/music-engine` that takes recent attempt history as an explicit parameter, used specifically to compute `ExerciseVerdict.consistency`, called by the Cloud Function after loading that history — rather than changing the core `evaluateAttempt(NoteEvent[], Lesson|Exercise) → ExerciseVerdict` function's signature.
**Context:** See OQ-04 (IMPORTANT).
**Options considered:** (a) Widen the core scoring function's signature to accept history — rejected, since it would make the engine's core function impure/context-dependent for every call, even ones that don't need a consistency score. (b) A second, separate function. — adopted.
**Decision made:** Option (b), proposed by the MUSIC documentation pass, `docs/MUSIC/SCORING_SYSTEM.md` §4.4.
**Reason:** Keeps the core scoring function's pure, two-argument contract intact for the large majority of callers/tests that don't need history.
**Consequences:** Needs an agreed function signature before implementation; not yet specified beyond "takes recent attempt history."
**Status:** Proposed.

### D-09 — Note-name octave-suffix display convention

**Date:** 2026-08-27
**Decision:** Proposed: `ExerciseVerdict` note-name fields use bare pitch-class names ("C") in the common case, and octave-qualified names ("C4") only when a single verdict would otherwise be ambiguous (the same pitch class appearing at two octaves within one attempt).
**Context:** Inferred from the mandatory canonical worked example (`missedNotes: ['F']`, bare names), which is the only place `PRODUCT_SPEC.md` shows this format concretely. See OQ-05 (IMPORTANT).
**Options considered:** Always bare names (simpler, but ambiguous for multi-octave exercises) vs. always octave-qualified (unambiguous, but doesn't match the canonical example's bare `'F'`) vs. bare-by-default with octave-qualification only when needed. The third option was chosen to match the canonical example exactly while still being unambiguous where it matters.
**Decision made:** Bare-by-default, octave-qualified when ambiguous — proposed by the MUSIC documentation pass, `docs/MUSIC/NOTE_SYSTEM.md` §4.5.
**Reason:** The canonical example is the one piece of ground truth available; extending it minimally, rather than contradicting it, was preferred.
**Consequences:** Binding on every `ExerciseVerdict`-producing code path once implemented — worth confirming before M1, since it's harder to change after content and tests are built against one convention.
**Status:** Proposed.

### D-10 — Gemini per-user daily rate-limit starting numbers

**Date:** 2026-08-27
**Decision:** Proposed starting caps: **20 tutor calls/day + 100 exercise-feedback calls/day for authenticated users; 5 tutor calls/day + 25 exercise-feedback calls/day for anonymous users.**
**Context:** `PRODUCT_SPEC.md` §19/§29 require a daily cap and an acceptance criterion that it's enforced, but never state a number. See OQ-15 (BLOCKING).
**Options considered:** No specific alternative numbers were modeled — the proposal is a starting point sized to allow generous normal use while bounding worst-case cost exposure, deliberately tighter for anonymous users given OQ-17 (anonymous-account-cycling bypasses per-`uid` limits).
**Decision made:** The numbers above, proposed by the AI documentation pass, `docs/AI/AI_COST_CONTROL.md` §5.
**Reason:** Sized from Gemini pricing-tier reasoning and expected early-user volume (`PRODUCT_SPEC.md` §31's "a few hundred early users" framing), not from any usage data (none exists pre-launch).
**Consequences:** These are the values `docs/TESTING/AI_TESTING.md`'s rate-limit tests are written against; revisit after real usage data exists.
**Status:** Proposed.

### D-11 — Rate-limit counter storage location

**Date:** 2026-08-27
**Decision:** Proposed: a Function-only-written counter, either a field on `UserProfile` or a small `users/{uid}/aiUsage/{date}` subdocument — not yet chosen between the two.
**Context:** No field exists anywhere in `TYPES_AND_CONTRACTS.md` to persist a per-day call counter. See OQ-15 (BLOCKING, same underlying gap as D-10).
**Options considered:** A field on the existing `UserProfile` document (fewer documents, but grows/churns a hot document on every AI call) vs. a dedicated per-day subdocument (keyed by date, naturally resets, Firestore-TTL-eligible for cleanup — modeled after `docs/AI/AI_COST_CONTROL.md`'s own daily-usage-document pattern). Leaning toward the subdocument approach for the TTL-cleanup benefit, but not decided.
**Decision made:** Not yet — both remain live options pending whoever implements M4.
**Reason:** Both are workable; the choice affects read/write cost patterns more than correctness, so it was left for implementation-time judgment rather than blocking documentation on it.
**Consequences:** M4 cannot be implemented without picking one.
**Status:** Proposed (narrower than D-10: this is "which shape," not "what number").

### D-12 — Theory Explanation / Motivation-Nudge routed through the two existing MVP callables

**Date:** 2026-08-27
**Decision:** Proposed: Theory Explanation is a deterministic-phrase-match-routed mode of `getTutorResponse` (not a separate callable); Motivation/Nudge's rare live-call path is folded into `getExerciseFeedback`'s response as an optional field (also not a separate callable).
**Context:** `PRODUCT_SPEC.md` §20 names only `getExerciseFeedback` and `getTutorResponse` as MVP callables, but §15 defines five prompt templates. See OQ-21/OQ-22 (IMPORTANT).
**Options considered:** (a) Add two more dedicated callables — rejected as disproportionate given both features' actual call volume is low (Theory Explanation is mostly cache-served; Motivation/Nudge is mostly pool-served). (b) Route both through the two existing callables. — adopted.
**Decision made:** Option (b), proposed by the AI documentation pass, `docs/AI/GEMINI_SPEC.md` §4.
**Reason:** Matches `PRODUCT_SPEC.md` §20's stated callable count without under-building either feature.
**Consequences:** `functions/src/ai/` must be structured around this routing table before M4 implementation starts; needs engineering sign-off.
**Status:** Proposed.

### D-15 — Beginner curriculum commitment: 14 lessons, C/G/D/A major

**Date:** 2026-08-27
**Decision:** Proposed: the MVP curriculum ships exactly 14 lessons, covering C major plus 3 additional keys (G, D, A major).
**Context:** `PRODUCT_SPEC.md` leaves lesson count at "~12–15" and additional-key count at "3–4." See OQ-01, OQ-11 (NON-BLOCKING for the key-count part).
**Options considered:** Various points within the stated ranges; 14 lessons / 3 additional keys was chosen as a concrete, buildable instantiation that lands cleanly within both ranges.
**Decision made:** 14 lessons, C/G/D/A major — proposed by the CONTENT documentation pass, `docs/CONTENT/BEGINNER_CURRICULUM.md`.
**Reason:** A concrete curriculum is needed to write worked scoring examples and content schema against; picking a specific point in an approved range is lower-risk than leaving it abstract through M2/M3 planning.
**Consequences:** A 4th key (E major) remains a low-risk, in-range future addition if desired.
**Status:** Proposed.

### D-16 — `deleteAccount` operation order

**Date:** 2026-08-27
**Decision:** Proposed order: (1) verify caller identity, (2) recursively delete `practiceSessions`, (3) delete `aiMemory/summary`, (4) delete `users/{uid}`, (5) delete the Firebase Auth record, (6) return success — implemented idempotently so a partial failure can be safely retried.
**Context:** `PRODUCT_SPEC.md` §18 mandates full-subtree deletion as "non-negotiable day-one scope" but doesn't specify sequencing, and Firestore does not cascade-delete subcollections automatically. See OQ-09 (IMPORTANT).
**Options considered:** Deleting the Auth record first (simpler mental model — "revoke access, then clean up") vs. deleting Firestore data first, Auth record last. The latter was chosen so a partially-failed operation leaves the `uid` still valid and retryable against known, identifiable (if partially deleted) data, rather than orphaned Firestore data with no Auth record left to retry against.
**Decision made:** Firestore-first, Auth-record-last ordering — proposed by the DATA/OPERATIONS documentation pass, `docs/DATA/DATA_RETENTION.md` §5.2.
**Reason:** Safer failure/retry characteristics under Firestore's lack of cross-collection transactions.
**Consequences:** Implementation must be written idempotently, not assumed atomic.
**Status:** Proposed.

### D-17 — `tempoBpm` as canonical field name (not `tempo`)

**Date:** 2026-08-28
**Decision:** Proposed: use `tempoBpm` as the canonical `Lesson` field name, not `tempo`.
**Context:** `PRODUCT_SPEC.md` §17 lists `tempo`; `TYPES_AND_CONTRACTS.md` §4.2 defines `tempoBpm`. Every downstream document (`LESSON_CONTENT_SCHEMA.md`, `MUSIC_ENGINE_SPEC.md`, `BEGINNER_CURRICULUM.md`) already uses `tempoBpm`. See OQ-30 (NON-BLOCKING).
**Options considered:** (a) `tempo` — matches the spec but is ambiguous about units. (b) `tempoBpm` — explicit about units, consistent with the project's convention of qualifying measurements, and already used everywhere downstream. — adopted.
**Decision made:** Option (b), proposed by the QC reconciliation pass.
**Reason:** `tempoBpm` is already the de facto name across all engineering and content documents; `tempo` appears only in the spec's shorthand field list.
**Consequences:** A copy-edit in `PRODUCT_SPEC.md` §17 to say `tempoBpm` would close the gap. No document downstream needs changing.
**Status:** Proposed.

### D-18 — `ExerciseVerdict` note-name fields are a deliberate display-layer exception to the MIDI-canonical rule

**Date:** 2026-08-28
**Decision:** Proposed: acknowledge `ExerciseVerdict.correctNotes/missedNotes/extraNotes` (string note names) as a deliberate exception to `TYPES_AND_CONTRACTS.md` §3's MIDI-canonical rule, not an oversight.
**Context:** §3 states "MIDI note numbers, not note-name strings, are the canonical representation of pitch everywhere in this codebase." §4.3 defines `ExerciseVerdict` with `correctNotes: string[]`, `missedNotes: string[]`, `extraNotes: string[]` — all human-readable note names. `ExerciseVerdict` is the one pitch-carrying type that crosses the server/client boundary specifically for direct UI rendering (the Lesson Player displays these names to the learner). See OQ-31 (NON-BLOCKING).
**Options considered:** (a) Change `ExerciseVerdict` to use MIDI numbers and convert to names client-side — rejected as unnecessary complexity for a display-only type. (b) Keep string names, add an inline note to §3 documenting the exception. — adopted.
**Decision made:** Option (b), proposed by the QC reconciliation pass.
**Reason:** Converting MIDI numbers to note names is a display concern; doing it server-side in the Function that already has the scoring result is simpler than threading MIDI numbers through the client and converting there. The string names are the final display format regardless.
**Consequences:** An inline note added to `TYPES_AND_CONTRACTS.md` §3 closes the gap. No type changes needed.
**Status:** Proposed.

### D-13 — Anonymous, never-linked user data retention

**Date:** 2026-08-27
**Decision:** Not made. Two candidate resolutions are on record without a chosen default. See OQ-20 (IMPORTANT).
**Context:** `PRODUCT_SPEC.md` never addresses what happens to an anonymous learner's data (a real `users/{uid}` document, possibly `practiceSessions`/`aiMemory`) if they never sign up. `PRODUCT_SPEC.md` §18 treats account deletion/export as "non-negotiable day-one scope" specifically because of POPIA/GDPR exposure — the same underlying concern applies here.
**Options considered:** (A) No automatic cleanup in MVP — simplest, but the DATA/OPERATIONS documentation pass explicitly called this "the weaker option given §18's own POPIA/GDPR framing," not a neutral default. (B) A scheduled Cloud Function purging inactive anonymous subtrees after N days — addresses the exposure directly, but needs an inactivity threshold `N` with no basis given anywhere to choose one.
**Decision made:** Neither — deliberately left open rather than defaulting to (A) by inaction, since (A) was flagged as the weaker option on privacy grounds, not a safe default to fall into silently.
**Reason:** This is a genuine privacy-posture decision, not an engineering default — it should be made deliberately by the project lead, not inherited from documentation silence.
**Consequences:** Does not block the M0–M5 internal-testing build (a handful of internal test accounts carries none of the exposure described above) but must be resolved before a real public launch.
**Status:** Open — needs a decision, not a sign-off on a proposal.

## 5. Out of scope

- Items with no proposed resolution and no decision framing (pure information gaps, naming inconsistencies in the source spec, etc.) are tracked in `docs/OPEN_QUESTIONS.md` only, not duplicated here.
- This log does not include ordinary engineering choices already settled directly in `docs/ENGINEERING/TECH_STACK.md` and `docs/ENGINEERING/DEPENDENCY_POLICY.md` (e.g. Next.js, TypeScript, Tailwind, Firebase) — those were given, not decided, by `PRODUCT_SPEC.md` and this task's own instructions.

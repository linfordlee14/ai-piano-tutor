# AI Context Model

**Status:** Draft v1.0 — specification only, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §17, §31; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8, §4.9
**Companion documents:** `docs/AI/GEMINI_SPEC.md`, `docs/AI/AI_PROMPT_ARCHITECTURE.md`, `docs/AI/AI_COST_CONTROL.md`, `docs/SECURITY/PRIVACY.md` (cross-referenced below; owns the privacy-boundary detail this document only summarizes), `docs/ARCHITECTURE/DATA_FLOW.md` §5 (traces the read side of this document's subject)

## 1. Purpose

Define what "AI memory" actually is in this product — one structured, rewritten-in-place Firestore document per learner, never a growing conversation log — and specify how it's computed, what triggers recomputation, and how it's assembled into a `TutorRequest` at call time.

## 2. Scope

In scope: the `AIMemorySummary` shape, its aggregation logic and inputs, the recomputation trigger, its assembly into a live tutor call, and what is deliberately never retained.

Out of scope: the Gemini call itself and its prompt text (`AI_PROMPTS.md`), the full privacy/data-retention policy this document's "what's not retained" section is a summary of (`docs/SECURITY/PRIVACY.md`), cost mechanics beyond why a small context is cheaper (`AI_COST_CONTROL.md`), Firestore schema/index detail beyond the one collection this document concerns (`docs/DATA/FIRESTORE_SCHEMA.md`).

## 3. Why a single structured document, not a growing transcript

`docs/PRODUCT_SPEC.md` frames this as your own instinct in the original brief, made structural rather than left as a footnote (§1's pushback table: "One structured Firestore document per learner, rewritten each session — never raw transcript logs"). The reasoning, stated plainly:

- **A transcript log grows without bound and gets more expensive to use over time, not less.** Every additional session appended to a raw log means either (a) sending more and more history into every future prompt — directly fighting §31's "short, structured-context prompts... keep both input and output token counts small," and the single biggest lever on cost being "called only on explicit user action" (§14) means nothing if the *size* of each call keeps growing — or (b) building a summarization/retrieval layer on top of the log just to keep prompts small, which is real engineering complexity spent solving a problem a structured summary avoids by construction.
- **A structured summary is exactly what a "what does the app know about this learner" context needs to be** — `skillLevel`, `weakNotes`, `weakChords`, `strengths`, `currentGoal`. None of that benefits from being phrased as a transcript; it benefits from being a small, current, directly-usable fact sheet. Rewriting one document in place is the natural shape for "the current state of what's true about this learner," in the same way `ProgressSummary` is a current-state map, not an append-only history of every progress change (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.7).
- **It is a privacy decision as much as a cost one.** A raw transcript log is a growing store of exactly the kind of content `docs/PRODUCT_SPEC.md` §25 says should never be duplicated into a less-protected copy — and even correctly scoped under `users/{uid}/**`, a transcript is strictly more sensitive, more retention-liability, and more breach-impact than a five-field summary that never contains a learner's actual words. Full policy: `docs/SECURITY/PRIVACY.md`.

The append-only record that *does* exist — `practiceSessions` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.6) — is not a substitute for a transcript log and isn't one either: it's structured, scored, deterministic performance data, not conversation content, and `AIMemorySummary` is computed *from* it, not appended alongside a copy of it.

## 4. Data structure

Canonical shape, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8 — reproduced here because this document is the one place its lifecycle is explained, not because this document owns the type:

```typescript
interface AIMemorySummary {
  skillLevel: 'beginner' | 'returning';
  weakNotes: string[];      // may be empty, note names
  weakChords: string[];     // may be empty, chord names
  strengths: string[];      // may be empty
  currentGoal: string | null; // explicit null if none set
  lastUpdated: FirebaseFirestore.Timestamp;
}
```

**Firestore path:** `users/{uid}/aiMemory/summary` — a single document, rewritten in place, never appended to. There is exactly one of these per learner, ever; there is no `aiMemory/{summaryId}` collection of historical summaries.

## 5. Computation

**Owner:** `functions/src/ai/` — aggregation logic lives alongside the prompt templates it feeds, not inside `packages/music-engine` (which stays a pure `(NoteEvent[], Lesson | Exercise) → ExerciseVerdict` function with zero Firestore dependency, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5, rule 2) and not client-side (the client never reads or writes this document directly — `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.8).

**Inputs:** the learner's recent `practiceSessions` records (`users/{uid}/practiceSessions/{sessionId}`), specifically each session's embedded `verdict: ExerciseVerdict`. The composite index on `(userId, completedAt desc)` (`docs/PRODUCT_SPEC.md` §17) makes "the learner's N most recent sessions" a single indexed query — the same query pattern already used to power the Progress/Stats screen and to read `recentExerciseContext` for a Tutor Q&A call (`docs/ARCHITECTURE/DATA_FLOW.md` §5.2, step 3c).

**Aggregation, field by field:**

| Field | Derived from |
|---|---|
| `skillLevel` | Set once, at onboarding, from the Quick Skill Check (`docs/PRODUCT_SPEC.md` §10) — `'beginner'` by default, `'returning'` if the learner self-identifies as having played before. Not recomputed from session data; a product decision the learner made, not an inference. |
| `weakNotes` | Note names that appear disproportionately often in `verdict.missedNotes` across the recent-session window — the same underlying signal `docs/PRODUCT_SPEC.md` §25 notes is "essentially free... already computed for the AI memory doc... just aggregate it" for the Analytics "most-missed notes/chords" metric. |
| `weakChords` | Same aggregation, for chord-type exercises specifically, using `verdict.missedNotes`/`extraNotes` on chord-drill attempts. |
| `strengths` | The inverse signal — notes/chords/exercise types with consistently high `verdict.accuracy` and `passed: true` across recent attempts. |
| `currentGoal` | Either the learner's next incomplete `Lesson` (from `ProgressSummary`) or an explicit learner-set goal if that's ever surfaced in the UI — `null` if neither applies yet. |
| `lastUpdated` | Set to the recomputation's own write time. |

⚠ **Open note (non-blocking):** `docs/PRODUCT_SPEC.md` doesn't specify the exact window size ("recent" — last N sessions? last N days?) or the exact statistical threshold for "disproportionately often" that promotes a note into `weakNotes`. This document doesn't invent a specific number for the same reason `docs/MUSIC/SCORING_SYSTEM.md` treats scoring weights as tunable exercise metadata rather than a hardcoded constant (`docs/PRODUCT_SPEC.md` §13) — this is exactly the kind of threshold that should be tunable post-launch from real data, not locked into a spec before any learner has used the app. Flag for whoever implements `functions/src/ai/computeMemorySummary.ts` to pick a starting value (e.g., last 10 sessions, a note appearing in `missedNotes` in ≥40% of its attempts) and revisit once usage data exists.

## 6. What triggers recomputation

`docs/PRODUCT_SPEC.md` does not specify the exact trigger point. Two options exist, and this document states the recommended default explicitly rather than leaving a real design decision silently unmade — consistent with `docs/ARCHITECTURE/DATA_FLOW.md` §4.4, which reaches the identical conclusion independently while tracing Flow 1:

**Recommended default: synchronous, inside `getExerciseFeedback`, immediately after the `PracticeSession` write.** Every exercise completion already invokes this Function; appending a recomputation step there adds no new trigger, no new moving part, and no new infrastructure to maintain. It also guarantees `AIMemorySummary` is never more than one exercise stale — critically, this means a `getTutorResponse` call made later in the *same session* (a very common pattern: finish an exercise, tap "ask the tutor why") always reads a memory summary that reflects the attempt the learner is asking about.

**Alternative (not recommended for MVP): an `onCreate` Firestore trigger on `practiceSessions`.** Decouples recomputation from the scoring request path, which only matters if aggregation ever becomes expensive enough to threaten `getExerciseFeedback`'s own response latency (`docs/PRODUCT_SPEC.md` §22) — not a concern at MVP's aggregation complexity (a handful of array operations over ≤10 recent sessions). Worth keeping in mind as a Phase 2 change if that changes, not worth the added infrastructure now.

⚠ **IMPORTANT, flagged for sign-off:** this is a real design decision `docs/PRODUCT_SPEC.md` leaves open. It is documented here (and independently, convergently, in `docs/ARCHITECTURE/DATA_FLOW.md`) as a recommendation, not a ratified requirement.

## 7. Assembly into a `TutorRequest`

At the moment `getTutorResponse` is called (`GEMINI_SPEC.md` §6), the Function reads the current `AIMemorySummary` document fresh — it is never cached client-side, never held across calls, and never supplied by the client:

```
getTutorResponse({ learnerQuestion })
  → verify caller's ID token
  → read users/{uid}/aiMemory/summary            → learnerMemorySummary
  → read most recent users/{uid}/practiceSessions  → recentExerciseContext (verdict, or null)
  → assemble TutorRequest {
      learnerQuestion,        // the only client-supplied field
      recentExerciseContext,  // ExerciseVerdict | null — the single most recent session, not the aggregate
      learnerMemorySummary,   // AIMemorySummary — the aggregate
      skillLevel: learnerMemorySummary.skillLevel  // duplicated for template convenience
    }
```

Two distinct pieces of context, deliberately not collapsed into one: `recentExerciseContext` is the single most recent attempt (sharp, specific — "here's exactly what just happened"), while `learnerMemorySummary` is the aggregate trend (broad — "here's the pattern over time"). A question like "why does my chord sound wrong" is best answered from the former; "what should I focus on" is best answered from the latter; the Tutor Q&A system instruction (`AI_PROMPTS.md` §3.1) is written to draw on whichever is actually relevant, not both indiscriminately. Full field-by-field trace: `docs/ARCHITECTURE/DATA_FLOW.md` §5.2.

## 8. What is explicitly not retained

**No raw conversation history is retained beyond what's needed to serve the current request.** A `learnerQuestion` is read, used to render one prompt, sent to Gemini, and the response is returned to the client — none of that text is written to Firestore, appended to `AIMemorySummary`, or persisted anywhere server-side once the call completes (`docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §4.3: `TutorResponse` objects render into the chat UI but aren't persisted as a growing client-side log beyond the current session's component state either). The only durable trace that a tutor conversation ever happened is whatever `AIMemorySummary`'s *next* recomputation derives from aggregate `PracticeSession` data — which, per §5 above, never includes conversation content as an input in the first place.

This is a **cost and privacy decision together**, not two separate ones that happen to agree:

- **Cost:** every byte of retained transcript is a byte some future prompt would either have to include (fighting the token-minimization goal in `docs/PRODUCT_SPEC.md` §31) or a summarization system would have to process to avoid including — either way, retained history is a recurring cost the structured-summary model avoids entirely. Full mechanics: `AI_COST_CONTROL.md`.
- **Privacy:** `docs/PRODUCT_SPEC.md` §25's privacy boundary states it directly for analytics ("never send tutor conversation content... into the general analytics stream") and this document extends the same logic to the AI memory store itself — conversation content is not "more correctly scoped" data that's safe to keep just because it lives under `users/{uid}/**`; it's data with no product justification for retention past the request that used it, and data that isn't retained can't be breached, subpoenaed, or exported by mistake. Full policy: `docs/SECURITY/PRIVACY.md`.

## 9. Security considerations

- `AIMemorySummary` is readable only by the owning `uid` and writable only by Cloud Functions using the Admin SDK (`docs/PRODUCT_SPEC.md` §19) — enforced redundantly by Firestore Security Rules and by the Function never accepting a client-supplied memory object (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5, rule 1).
- Because `learnerMemorySummary` and `recentExerciseContext` are entirely server-assembled, a learner cannot fabricate a flattering (or attention-grabbing) performance history to manipulate the tutor's tone or answers — the context-fabrication vector is closed the same way the score-tampering vector is (`docs/ARCHITECTURE/DATA_FLOW.md` §3).
- No field of `AIMemorySummary` or `TutorRequest` ever contains a raw `NoteEvent[]` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5, rule 3) — Gemini only ever sees already-summarized data, never anything it could be tempted to "recompute" from.

## 10. Performance considerations

- Reading `AIMemorySummary` and the most recent `PracticeSession` at Tutor Q&A call time is two single-document/single-row-limit-1 Firestore reads, not a scan — cheap, and already gated behind explicit user action so the 1–3 second Gemini latency budget (`docs/PRODUCT_SPEC.md` §22) dominates total call time regardless.
- Keeping the summary small (five fields, none of them large arrays by design — `weakNotes`/`weakChords`/`strengths` are expected to stay short lists, not accumulate indefinitely) keeps every downstream prompt small, which is the direct mechanism behind §31's token-count goal.

## 11. Dependencies

- `packages/shared-types` — `AIMemorySummary`, `PracticeSession`, `ExerciseVerdict` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md`).
- Firestore composite index on `practiceSessions (userId, completedAt desc)` (`docs/PRODUCT_SPEC.md` §17) — required for both the recomputation aggregation and the `recentExerciseContext` read to stay single-query operations.
- `functions/src/ai/` — owns both the aggregation logic (this document) and the prompt templates that consume its output (`AI_PROMPTS.md`).

## 12. Testing requirements

- Unit tests on the aggregation function itself, with fixture `PracticeSession[]` arrays as input and an exact expected `AIMemorySummary` as output — pure-function testable the same way `packages/music-engine` is (`docs/PRODUCT_SPEC.md` §26), since aggregation is deterministic arithmetic over structured data, not an AI call.
- Integration test (Firebase Emulator Suite) asserting recomputation actually fires on the chosen trigger point (§6) and that a `getTutorResponse` call made immediately after an exercise completion reads the freshly recomputed summary, not a stale one.
- Security-rules test asserting a client cannot write `users/{uid}/aiMemory/summary` directly (`docs/PRODUCT_SPEC.md` §29's client-cannot-write-progress acceptance criterion, applied to this collection specifically).

## 13. Out of scope

- The Gemini call and prompt text that consumes this context → `AI_PROMPTS.md`.
- Full data-retention and privacy policy → `docs/SECURITY/PRIVACY.md`.
- Cost mechanics and instrumentation → `AI_COST_CONTROL.md`.
- Firestore schema/index detail beyond `aiMemory/summary` and the `practiceSessions` index this document depends on → `docs/DATA/FIRESTORE_SCHEMA.md`.
- The deterministic scoring computation that produces the `ExerciseVerdict` records this document aggregates → `docs/MUSIC/MUSIC_ENGINE_SPEC.md`.

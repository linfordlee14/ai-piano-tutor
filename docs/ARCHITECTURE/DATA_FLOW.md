# Data Flow

**Status:** Draft v1.0 — architecture documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §11, §13, §14, §15, §17, §20, §22; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` (all type shapes below are reproduced by name, not redefined — this document is not the source of truth for any type's shape)
**Companion documents:** `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md`, `docs/ARCHITECTURE/AI_ARCHITECTURE.md`, `docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md`, `docs/ARCHITECTURE/SECURITY_ARCHITECTURE.md`

## 1. Purpose

Trace the two most important operations in AI Piano Tutor end to end, at the level of precision `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5 explicitly asks for ("Any client-side 'preview' verdict computed optimistically... is explicitly provisional and is overwritten by the server's response — see `docs/ARCHITECTURE/DATA_FLOW.md`"). This document is that pointer's destination.

1. **A learner completes a lesson exercise.**
2. **A learner asks the tutor a question.**

For each: what crosses the network at each step, in which direction, using which type; and — the distinction that matters most for a real-time app with a deterministic-vs-AI trust split — what is **client-computed-but-provisional** versus **server-authoritative**.

**The non-negotiable rule both traces make concrete:** Gemini must never determine whether a note or chord is correct, calculate accuracy, calculate timing, calculate scores, determine lesson completion, determine XP, determine streaks, or validate user-submitted performance data. Both flows below are constructed so that every one of those computations happens inside `packages/music-engine` (Flow 1, §4) or by reading already-server-written data (Flow 2, §5) *before* Gemini is ever invoked — Gemini's only input in either flow is a result the deterministic engine or Firestore has already finished producing, and its only output is prose that is never written back into a scored field. Full boundary and reasoning: `AI_ARCHITECTURE.md` §3.

## 2. Scope

In scope: the two flows above, traced field-by-field using exact type names from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`.

Out of scope: the Daily Practice Plan flow (`generateDailyPlan`, Phase 2 — not built, not traced here), the account-deletion/export flows (straightforward request/response, not architecturally interesting enough to warrant a trace), and the internal algorithm used to compute `ExerciseVerdict` (→ `docs/MUSIC/MUSIC_ENGINE_SPEC.md`) or to assemble a Gemini prompt string (→ `docs/AI/AI_PROMPTS.md`).

## 3. The provisional/authoritative distinction, stated once

Both flows below depend on one rule, so it's stated here rather than repeated twice:

> **The Engine Core (`packages/music-engine`) runs in two places: client and server, using the identical code, imported from the same package by both `apps/web` and `functions/`.** The client-side run exists purely for latency — instant visual feedback while a key is being judged. The server-side run, inside `getExerciseFeedback`, is the one that is ever persisted, ever affects XP/streak/progress, and is ever treated as true. A client-side `ExerciseVerdict` is a *preview*; nothing downstream of the network round trip reads it as fact.

This is not a redundant safety net around a trustworthy client — it is the load-bearing mechanism that makes the score-tampering vector (`SECURITY_ARCHITECTURE.md`) impossible: a modified client can compute and display any `ExerciseVerdict` it wants locally, but that object is never sent to the server and never written anywhere. Only the raw `NoteEvent[]` is sent, and the server recomputes independently.

## 4. Flow 1 — Learner completes a lesson exercise

### 4.1 Sequence

```mermaid
sequenceDiagram
    participant Input as Input Layer
    participant EngineC as Engine Core (client)
    participant UI as UI Layer
    participant Sync as Sync Layer
    participant Fn as Cloud Function<br/>getExerciseFeedback
    participant Engine as Engine Core (server, same code)
    participant FS as Firestore
    participant Gemini as Gemini API

    Input->>EngineC: NoteEvent (per key press, local)
    EngineC-->>UI: provisional ExerciseVerdict (local, instant)
    Note over Input,UI: Closed local loop — no network yet.<br/>Repeats per note until exercise ends.
    Input->>Sync: exercise ends, buffered NoteEvent[]
    Sync->>Fn: ExerciseAttempt { noteEvents, lessonId/exerciseId,<br/>startedAt, submittedAt, deviceType, attemptNumber }
    Note over Fn: Every field treated as attacker-controlled.
    Fn->>FS: load Lesson/Exercise definition (server-trusted, not from client)
    Fn->>Engine: evaluate(noteEvents, lessonDef)
    Engine-->>Fn: authoritative ExerciseVerdict
    Fn->>FS: write PracticeSession (embeds verdict)
    Fn->>FS: write/update ProgressSummary (on UserProfile doc)
    Fn->>Gemini: Exercise Feedback prompt<br/>(assembled from the verdict just computed)
    Gemini-->>Fn: feedback prose
    Fn-->>Fn: validate prose against output contract;<br/>fall back to canned message if invalid
    Fn-->>Sync: { ok: true, data: { verdict, feedbackText } }
    Sync-->>UI: replace provisional verdict with authoritative one
    Note over UI: Learner may now tap "ask the tutor why" — see Flow 2,<br/>a separate, deeper request to getTutorResponse.
```

### 4.2 Step-by-step, with exact types

| # | Step | Crosses network? | Type | Client-provisional or server-authoritative |
|---|---|---|---|---|
| 1 | Input adapter (mouse/touch/keyboard) emits one event per key action | No | `NoteEvent` | N/A — raw input |
| 2 | Audio Layer plays sound; Engine Core (client) evaluates against the active `Lesson`/`Exercise` and produces a running verdict for instant key-coloring | No | `ExerciseVerdict` (client-side instance) | **Client-provisional.** Used only for UI (§3), never sent anywhere, never persisted. |
| 3 | Exercise ends. Sync Layer packages the full buffered event sequence | No (assembly step) | `ExerciseAttempt` | N/A — assembly of already-local data |
| 4 | Client calls `getExerciseFeedback(attempt)` | **Yes — client → Cloud Function** | `ExerciseAttempt` (containing `noteEvents: NoteEvent[]`) | The *only* thing that crosses here. No client-computed verdict, score, or `compositeScore` is ever included. |
| 5 | Function loads the `Lesson` (or `Exercise`) definition it needs to score against | Yes — Function → Firestore (or static config for `Exercise`) | `Lesson` / `Exercise` | Server-authoritative by construction — the client never supplies its own copy of the definition to score against. |
| 6 | Function re-runs `packages/music-engine` server-side against the attempt's raw `noteEvents` and the definition from #5 | No (server-internal) | Produces `ExerciseVerdict` | **Server-authoritative.** This is the only `ExerciseVerdict` instance that ever leaves the Function as fact. |
| 7 | Function writes the result | Yes — Function → Firestore | `PracticeSession` (embeds the verdict from #6), updated `ProgressSummary` entry (as part of `UserProfile`) | Server-authoritative; written via Admin SDK, bypassing client security rules entirely (`SECURITY_ARCHITECTURE.md`). |
| 8 | Function assembles the Exercise Feedback prompt from the verdict computed in #6 (not from the client's attempt) and calls Gemini | Yes — Function → Gemini | Prompt input: `{exerciseType, expected, played, accuracyScore, missedNotes, attemptNumber, learnerLevel}` (`docs/PRODUCT_SPEC.md` §15) — assembled server-side from the `ExerciseVerdict` + `Lesson` metadata | Gemini's output is prose only; it does not modify, produce, or get asked to sanity-check any field of the verdict (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.5 — see `AI_ARCHITECTURE.md`). |
| 9 | Function validates Gemini's response (length cap, non-empty, on-contract) and falls back to a canned message on failure | No (server-internal) | — | If validation fails, the *deterministic* verdict from #6 is still returned — a bad Gemini response never blocks the scored result. |
| 10 | Function returns to client | **Yes — Cloud Function → client** | `{ ok: true, data: { verdict: ExerciseVerdict, feedbackText: string } }` (error case: `{ ok: false, error: { code, message } }`) | Server-authoritative. This overwrites the client's provisional verdict from step 2 in the UI/Zustand. |
| 11 | *(Optional, separate request)* Learner taps "ask the tutor why" | See Flow 2 | `{ learnerQuestion }` | Not part of this flow's round trip — a distinct call to `getTutorResponse`. |

### 4.3 Why step 10 bundles the Gemini call into `getExerciseFeedback`

`docs/PRODUCT_SPEC.md` §14 lists Gemini's call triggers as "explicit learner action... **or at the end of a scored exercise**" — end-of-exercise is its own trigger, distinct from the "ask why" tap. §15's *Exercise Feedback* template output ("what was right first, the specific miss named, one concrete next step") is exactly the short, automatic feedback a learner expects immediately after an attempt, not a deeper Q&A. §16's App Check row also refers to "the Gemini-calling **Function**s" (plural), consistent with there being two Gemini-calling functions in MVP (`getExerciseFeedback` and `getTutorResponse`), not one. Read together, `getExerciseFeedback` is documented here as performing **both** the deterministic re-scoring **and** the automatic short Gemini feedback call, in one round trip — the separate "ask the tutor why" entry point (`docs/PRODUCT_SPEC.md` §10) is a deeper, conversational follow-up through `getTutorResponse` (Flow 2), grounded in the `PracticeSession` this flow just wrote. This reading is flagged as an open question in this document's parent task report — it is the most textually consistent interpretation across §14/§15/§16/§20, but the two-call-vs-one-call shape of `getExerciseFeedback`'s response is a real API contract decision worth an explicit implementer sign-off before `functions/src/ai/getExerciseFeedback.ts` is written.

### 4.4 What never happens in this flow

- The client never sends a `compositeScore`, `accuracy`, or `passed` value to the server — only raw `NoteEvent[]` inside `ExerciseAttempt`.
- The server never trusts `ExerciseAttempt.attemptNumber`, `startedAt`, or `submittedAt` for scoring purposes — they are explicitly informational/analytics-only (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4); the server derives its own attempt count from existing `PracticeSession` records if attempt-number-dependent logic is needed.
- Gemini never sees a raw `NoteEvent[]` — only the already-summarized verdict fields (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5, rule 3).
- `AIMemorySummary` is not shown as written inside this flow's diagram because `docs/PRODUCT_SPEC.md` does not specify the exact trigger point for its recomputation (synchronously inside `getExerciseFeedback` vs. an asynchronous Firestore-triggered function reacting to new `PracticeSession` writes). This document's recommended default — stated explicitly because an architecture doc should not leave a real design decision silently unmade — is **synchronous, inside `getExerciseFeedback`**, immediately after the `PracticeSession` write in step 7: it is the simplest option (one fewer moving part / trigger to maintain), and it guarantees `AIMemorySummary` is never stale by the time a `getTutorResponse` call in the same session reads it (Flow 2, step 3b). The alternative (an `onCreate` Firestore trigger on `practiceSessions`) remains available as a Phase 2 change if the aggregation work ever becomes expensive enough to move off the request path. Flagged as an open question in the parent report.

## 5. Flow 2 — Learner asks the tutor a question

### 5.1 Sequence

```mermaid
sequenceDiagram
    participant UI as Tutor UI
    participant Fn as Cloud Function<br/>getTutorResponse
    participant FS as Firestore
    participant Gemini as Gemini API

    UI->>Fn: { learnerQuestion: string }
    Note over Fn: This is the ENTIRE client payload.<br/>Everything else is assembled here, server-side.
    Fn->>Fn: verify caller's ID token (context.auth)
    Fn->>FS: read users/{uid}/aiMemory/summary
    FS-->>Fn: AIMemorySummary
    Fn->>FS: read most recent users/{uid}/practiceSessions/{id}<br/>(ordered by completedAt desc)
    FS-->>Fn: PracticeSession (or none)
    Fn->>Fn: assemble TutorRequest {<br/>  learnerQuestion,<br/>  recentExerciseContext: verdict from that session, or null,<br/>  learnerMemorySummary,<br/>  skillLevel<br/>}
    Fn->>Fn: length-cap learnerQuestion; check + atomically<br/>increment per-user rate-limit counter
    Fn->>Gemini: Tutor Q&A prompt, rendered from TutorRequest
    Gemini-->>Fn: raw response text
    Fn-->>Fn: validate against TutorResponse contract;<br/>fall back to canned message if invalid
    Fn-->>UI: { ok: true, data: TutorResponse }
```

### 5.2 Step-by-step, with exact types

| # | Step | Crosses network? | Type | Client-provisional or server-authoritative |
|---|---|---|---|---|
| 1 | Learner types a question (or taps a quick-suggestion chip) and submits | No | Plain string, held in Zustand as a chat draft | N/A |
| 2 | Client calls `getTutorResponse({ learnerQuestion })` | **Yes — client → Cloud Function** | `{ learnerQuestion: string }` — **this is the entire client-supplied payload.** No `recentExerciseContext`, no `learnerMemorySummary`, no `skillLevel` is accepted from the client even if a modified client tried to send them (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9: "the client-facing Cloud Function signature accepts only `{ learnerQuestion: string }`") | The question text itself is the only thing that could be called "client-provisional" here, and it isn't a verdict — it's the actual input to the conversation. |
| 3a | Function verifies the caller via Firebase ID token (Admin SDK) | Implicit in #2's call | — | Server-authoritative identity; a client-supplied `uid` is never trusted on its own (`docs/PRODUCT_SPEC.md` §18). |
| 3b | Function reads `users/{uid}/aiMemory/summary` | Yes — Function → Firestore | `AIMemorySummary` | Server-authoritative — this document was itself written by a prior `getExerciseFeedback` call (or its recomputation step, §4.4), never by the client. |
| 3c | Function reads the most recent `users/{uid}/practiceSessions/{sessionId}` (the composite index on `(userId, completedAt desc)` from `docs/PRODUCT_SPEC.md` §17 makes this a single indexed query) | Yes — Function → Firestore | `PracticeSession`, specifically its embedded `verdict: ExerciseVerdict` | Server-authoritative; `null` if the learner has no sessions yet (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9 requires `recentExerciseContext` to explicitly be `ExerciseVerdict \| null`, not omitted). |
| 4 | Function assembles the full `TutorRequest` from #2 + #3b + #3c, entirely server-side | No (server-internal) | `TutorRequest { learnerQuestion, recentExerciseContext, learnerMemorySummary, skillLevel }` | Server-authoritative by construction — this is the specific mechanism that prevents a learner from fabricating their own performance context to manipulate the tutor's answer (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.9). |
| 5 | Function caps `learnerQuestion` length and checks/atomically increments the per-user rate-limit counter | Yes — Function → Firestore (counter read/increment) | — | Server-authoritative gate; a request over the cap short-circuits to `{ ok: false, error }` before Gemini is ever called (`SECURITY_ARCHITECTURE.md`). |
| 6 | Function renders the Tutor Q&A prompt template from `TutorRequest` and calls Gemini | Yes — Function → Gemini | Prompt input matches the `TutorRequest` shape (`docs/PRODUCT_SPEC.md` §15); Gemini receives no raw `NoteEvent[]`, only the already-summarized verdict/memory fields | Gemini's output is the *only* non-deterministic element in this entire flow — everything upstream of it (identity, memory, recent performance) is deterministic and server-assembled. |
| 7 | Function validates the raw response (well-formed, within the ≤150-word cap unless more was asked for, on-contract) | No (server-internal) | — | On failure: deterministic canned fallback message, never a broken screen (`docs/PRODUCT_SPEC.md` §15). |
| 8 | Function returns to client | **Yes — Cloud Function → client** | `{ ok: true, data: TutorResponse { text, suggestedAction?, tier, cached } }` | Server-authoritative; `cached: true` specifically when this answer came from the theory-explanation cache (`docs/PRODUCT_SPEC.md` §15) rather than a live Gemini call — see `AI_ARCHITECTURE.md` for the caching mechanism itself. |
| 9 | Client renders `TutorResponse.text` in the chat UI | No | — | Not persisted as a growing transcript client-side; the only durable trace of this exchange is whatever `AIMemorySummary` recomputation later derives from aggregate `PracticeSession` data — not from tutor conversation content itself (`docs/PRODUCT_SPEC.md` §25's privacy boundary: tutor conversation content is never sent to analytics or retained as a transcript). |

### 5.3 What never happens in this flow

- The client never supplies its own `AIMemorySummary` or `recentExerciseContext` — even a fully compromised client cannot influence what "performance context" the tutor reasons from, because that data is read server-side from Firestore documents the client cannot write (`SECURITY_ARCHITECTURE.md`).
- Gemini is never called before the rate-limit check in step 5 passes — cost and abuse protection happens strictly before the expensive call, not after.
- No `NoteEvent[]` of any kind appears anywhere in this flow — Flow 2 operates entirely on already-summarized data (verdicts, memory), consistent with `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5's rule that Gemini-facing types never contain raw note data.

## 6. Cross-flow summary — network boundary crossings

| Direction | Flow 1 (exercise) | Flow 2 (tutor question) |
|---|---|---|
| Client → Function | `ExerciseAttempt` (raw `NoteEvent[]` + metadata) | `{ learnerQuestion: string }` only |
| Function → Firestore (read) | `Lesson`/`Exercise` definition | `AIMemorySummary`, most recent `PracticeSession` |
| Function → Firestore (write) | `PracticeSession`, `ProgressSummary` (on `UserProfile`), `AIMemorySummary` (recomputation, per §4.4) | rate-limit counter increment only |
| Function → Gemini | Exercise Feedback prompt (from server-computed verdict) | Tutor Q&A prompt (from server-assembled `TutorRequest`) |
| Function → Client | `{ verdict: ExerciseVerdict, feedbackText }` | `TutorResponse` |

In both flows, the pattern is identical at its core: **the client sends the least amount of raw, unprocessed data possible (events, or a bare question string); the server assembles everything else from data it alone can write; the client receives back only fully-computed, authoritative results.** This single pattern is what makes both the score-tampering vector and the context-fabrication vector structurally closed, not just policy-closed.

## 7. Out of scope

- The `generateDailyPlan` flow (Phase 2, not built — `docs/PRODUCT_SPEC.md` §7, §8).
- `deleteAccount` / `exportUserData` — straightforward request/response, not traced here.
- The internal `packages/music-engine` algorithm that turns `(NoteEvent[], Lesson)` into an `ExerciseVerdict` → `docs/MUSIC/MUSIC_ENGINE_SPEC.md`.
- The exact Gemini prompt template text and JSON-schema/structured-output configuration → `docs/AI/AI_PROMPTS.md`.
- The theory-explanation cache's key/storage mechanism → `docs/AI/AI_COST_CONTROL.md`.

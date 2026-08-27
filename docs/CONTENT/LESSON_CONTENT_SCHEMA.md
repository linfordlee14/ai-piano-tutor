# Lesson Content Schema

**Status:** Draft v1.0 — content documentation, no implementation yet
**Source of truth:** `docs/PRODUCT_SPEC.md` §17, §27; `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 (`Lesson`), §4.14 (`ScoringWeights`); `docs/ENGINEERING/PROJECT_STRUCTURE.md` §3 (`content/lessons/`, `scripts/seedLessons.ts`)
**Companion documents:** `docs/CONTENT/LESSON_ARCHITECTURE.md` (the runtime behavior this content drives), `docs/CONTENT/BEGINNER_CURRICULUM.md` (the 14 real lessons authored against this schema), `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` (how to write the prose fields this schema defines the shape of), `docs/MUSIC/SCORING_SYSTEM.md` (the `ScoringWeights` mechanics §Validation rules enforces), `docs/ENGINEERING/CODING_STANDARDS.md` §3.1 (Zod recommended as the validation mechanism this document assumes)

---

## 1. Purpose

Define the exact JSON shape of one lesson content file under `content/lessons/*.json` (`docs/ENGINEERING/PROJECT_STRUCTURE.md` §3) — the **authoring-time** format a human or agent writes by hand — and how it maps onto the **runtime** `Lesson` TypeScript interface (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2) that `packages/music-engine` and the rest of the app actually consume. The two are closely related but not identical: this document's schema is what gets committed to the repo and reviewed in a pull request; `Lesson` is what a Cloud Function or client reads back out of Firestore after `scripts/seedLessons.ts` loads it. Where the two diverge, this document says so explicitly (§4.2).

## 2. Scope

In scope: the full JSON field list for a lesson content file, its mapping onto `Lesson`, the one field this document proposes adding beyond `Lesson`'s current canonical shape (`hints`), validation rules enforced at seed time, and a complete worked example.

Out of scope: which 14 lessons actually exist and what each one's content is (`docs/CONTENT/BEGINNER_CURRICULUM.md` — this document only specifies the *shape*, not the *content*, though its one worked example is a real MVP lesson, not a synthetic one), the runtime unlocking/completion/hint-escalation mechanics that consume this content once seeded (`docs/CONTENT/LESSON_ARCHITECTURE.md`), tone/style guidance for the prose inside `instructions`/`hints` (`docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`), the scoring formulas `scoringWeights` feeds (`docs/MUSIC/SCORING_SYSTEM.md`).

## 3. Requirements

- **One JSON file per lesson**, filename matching its `id` (`content/lessons/{id}.json`), loaded into the `lessons` Firestore collection by `scripts/seedLessons.ts` — never hand-authored per-document in the Firestore console (`docs/PRODUCT_SPEC.md` §17; `docs/DATA/FIRESTORE_SCHEMA.md` §4.4).
- **The seed script is the validation boundary, not runtime.** `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14 states the model directly: "a Lesson/Exercise seed file that fails this check should fail content validation at build/seed time, not at runtime." Every rule in §4.4 below runs inside `scripts/seedLessons.ts`, before a single document reaches Firestore — a malformed lesson file fails CI/the seed run, not a live learner's session.
- **Every field this schema defines must map onto a `Lesson` field with no semantic gap**, except the one explicitly-flagged proposed addition (§4.2) — this document is not a license to invent authoring convenience fields that have no runtime meaning.
- **Zod is the recommended validation mechanism** (`docs/ENGINEERING/CODING_STANDARDS.md` §3.1's project-wide recommendation, explicitly delegated to this document by `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §6) — a `LessonContentSchema` Zod object whose `z.infer` type is checked against `packages/shared-types`' `Lesson` at compile time, so the two shapes can't silently drift apart.

## 4. Architecture

### 4.1 Full field list

| Field | Type | Required? | Maps to `Lesson` field | Notes |
|---|---|---|---|---|
| `id` | `string` | required | `id` | Stable slug, must equal the filename minus `.json` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §3: "Firestore auto-IDs unless a stable slug is specified"). |
| `title` | `string` | required | `title` | Learner-facing lesson name, shown in the Learn tab's lesson list (`docs/PRODUCT_SPEC.md` §10). |
| `level` | `integer` | required | `level` | Strictly increasing, unique across all lessons — `docs/CONTENT/LESSON_ARCHITECTURE.md` §4.2's "one strictly increasing integer per lesson" rule. |
| `type` | `'orientation' \| 'noteId' \| 'octave' \| 'melody' \| 'rhythm' \| 'scale' \| 'chord'` | required | `type` | Exact union from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 — no other string value is valid. |
| `expectedSequence` | `ExpectedNote[]` | required (may be `[]`) | `expectedSequence` | `[]` only for `type: 'orientation'` (`docs/MUSIC/MUSIC_ENGINE_SPEC.md` §5.2's auto-pass case) — §4.4's validation rules reject an empty sequence on any other `type`. |
| `expectedSequence[].midiNote` | `integer, 0–127` | required | `ExpectedNote.midiNote` | Same range as `NoteEvent.midiNote` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1). |
| `expectedSequence[].expectedTimeMs` | `integer ≥ 0` | optional | `ExpectedNote.expectedTimeMs` | Omit entirely for a lesson where order matters but timing doesn't (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2's own comment) — see §4.4's "timing weight must be zero" rule, which this field's presence/absence directly governs. |
| `tempoBpm` | `integer` | optional | `tempoBpm` | Only meaningful for `type: 'melody' \| 'rhythm'` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2) — required when any `expectedTimeMs` is present, since a tempo-less timed lesson has no meaningful demonstration playback rate (`docs/CONTENT/LESSON_ARCHITECTURE.md` §4.3). |
| `instructions` | `string` | required | `instructions` | Learner-facing prose — see `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` for tone/style. |
| `prerequisiteLessonId` | `string \| null` | required | `prerequisiteLessonId` | Explicit `null` for the first lesson in the chain — never omitted (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2: "explicit null for the first lesson, not omitted"). |
| `scoringWeights` | `{ correctness, timing, consistency }` | required | `scoringWeights` | Each `0–1`; must sum to `1.0 ± 0.001` — §4.4's validation rule, `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14. |
| `passThreshold` | `number, 0–1` | required | `passThreshold` | No ratified default (`OQ-01`) — `docs/CONTENT/BEGINNER_CURRICULUM.md` §5 recommends `0.8` uniformly; author against that recommendation until/unless it's superseded. |
| `hints` | `string[]` | required — **proposed field, see §4.2** | *(none yet — proposed addition)* | Ordered, tier 1 first; the last element repeats for any attempt beyond the array's length (`docs/CONTENT/LESSON_ARCHITECTURE.md` §4.4). |

### 4.2 `hints` — a proposed schema field pending confirmation against the canonical types

`docs/CONTENT/LESSON_ARCHITECTURE.md` §4.4 identifies a real, **BLOCKING** gap: `Lesson` (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2) has no field to store hint content at all, even though escalating hints are a required MVP core-loop feature (`docs/PRODUCT_SPEC.md` §9, §10; PR-LESSON-05) that multiple documents across this batch already assume works. This document does not have authority to edit `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` — it adopts `docs/CONTENT/LESSON_ARCHITECTURE.md` §4.4's proposed resolution (`hints: string[]`, 3-tier, last-element-repeats) into this schema **now**, so a usable content file can actually be authored today, while flagging — consistently with that document — that this field is a proposal awaiting confirmation, not yet a canonical `Lesson` field. If `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` is updated to add `hints` (or a differently-named/shaped equivalent) with a different exact shape, this schema and every content file authored against it need a corresponding, mechanical update — a real but low-risk migration, since it only touches one field.

### 4.3 Complete worked example — `content/lessons/l09-basic-rhythm.json`

This document shows `l09-basic-rhythm` rather than `l01`/`l02` (the curriculum's literal first lessons — see `docs/CONTENT/BEGINNER_CURRICULUM.md` §4.1) as its one complete worked example specifically because it is the first MVP lesson to populate **every** field meaningfully, including the two optional ones (`tempoBpm`, every note's `expectedTimeMs`) — `l01`/`l02`'s own files are strict subsets of this shape (`l01`'s `expectedSequence` is `[]`; neither uses `tempoBpm`) and are trivial to derive once every field here is understood. `docs/CONTENT/BEGINNER_CURRICULUM.md` §4.1 is authoritative for what `l01`–`l14` each actually are; this is authoritative for the file *format* all fourteen share.

```json
{
  "id": "l09-basic-rhythm",
  "title": "Basic Rhythm",
  "level": 9,
  "type": "rhythm",
  "expectedSequence": [
    { "midiNote": 60, "expectedTimeMs": 0 },
    { "midiNote": 62, "expectedTimeMs": 500 },
    { "midiNote": 64, "expectedTimeMs": 1000 },
    { "midiNote": 62, "expectedTimeMs": 1500 },
    { "midiNote": 60, "expectedTimeMs": 2000 }
  ],
  "tempoBpm": 120,
  "instructions": "You already know these five notes — C, D, E, D, C. This time, play them in steady time, one every half second, like a slow, even heartbeat. Watch the demonstration first, then try to match its timing, not just its notes.",
  "prerequisiteLessonId": "l08-simple-melody-two",
  "scoringWeights": {
    "correctness": 0.4,
    "timing": 0.5,
    "consistency": 0.1
  },
  "passThreshold": 0.8,
  "hints": [
    "Try tapping your foot or nodding along to a steady beat before you start playing — that's the pace to match.",
    "Focus on hitting C, D, and E right on the beat rather than rushing ahead to the next note.",
    "Play each note exactly on the half-second mark: C, then D a half-second later, then E, then D, then C — like counting '1, 2, 3, 4, 5' evenly."
  ]
}
```

**Field-by-field notes on this example, cross-referencing the mechanics that consume it:**
- `expectedSequence` — five notes, `expectedTimeMs` spaced exactly `500ms` apart at `tempoBpm: 120` (a quarter note at 120bpm is 500ms) — this populates every `NoteTimingResult` bucket path in `docs/MUSIC/SCORING_SYSTEM.md` §4.2 when a real attempt is scored against it, unlike `l07`'s canonical example (§4.9 of that document), which deliberately has no `expectedTimeMs` at all.
- `scoringWeights` sums to exactly `1.0` (`0.4 + 0.5 + 0.1`) — passes the seed-time validation rule in §4.4 below, and is a real, non-default weighting (contrast `docs/MUSIC/SCORING_SYSTEM.md` §4.5's `0.6/0.25/0.15` default), demonstrating that `ScoringWeights` genuinely is per-lesson authored metadata, not a constant (`docs/PRODUCT_SPEC.md` §13).
- `hints` — three tiers, general → specific → near-explicit, matching `docs/CONTENT/LESSON_ARCHITECTURE.md` §4.4's proposed escalation and `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`'s tone guidance (encouraging, never punitive — PR-LESSON-04).
- `passThreshold: 0.8` — `docs/CONTENT/BEGINNER_CURRICULUM.md` §5's recommended MVP default, used here as authored, not as a placeholder standing in for something else.

### 4.4 Validation rules (enforced by `scripts/seedLessons.ts`, at seed time)

| # | Rule | Source |
|---|---|---|
| 1 | `id` matches the filename (minus `.json`) exactly. | This document — a basic authoring-integrity check. |
| 2 | `id` is unique across every file in `content/lessons/`. | `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §3. |
| 3 | `level` is a unique integer across all lessons, and the full set of `level`s forms a contiguous run starting at `1` (no gaps) — `docs/CONTENT/LESSON_ARCHITECTURE.md` §4.2. | `docs/CONTENT/LESSON_ARCHITECTURE.md` §4.2. |
| 4 | `prerequisiteLessonId` is either `null` (exactly one lesson may have this) or the `id` of another lesson in the set — no dangling references. | `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2. |
| 5 | The full `prerequisiteLessonId` graph is a single acyclic chain (no branches, no cycles, every lesson reachable from the one `null` root) — MVP has no branching paths. | `docs/PRODUCT_SPEC.md` §17; `docs/PRODUCT/MVP_SCOPE.md` §2. |
| 6 | `expectedSequence` is `[]` only when `type === 'orientation'`; every other `type` requires at least one entry. | `docs/MUSIC/MUSIC_ENGINE_SPEC.md` §5.2 (the auto-pass case is `'orientation'`-only by curriculum convention, not enforced by the type system itself — this rule is what makes that convention a checked invariant rather than a house style). |
| 7 | Every `expectedSequence[].midiNote` is an integer in `[0, 127]`. | `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.1. |
| 8 | Within one lesson, `expectedSequence[].expectedTimeMs` is either present on **every** entry or **none** — no partially-timed sequence. | This document, closing a gap `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 leaves open per-note; a mixed sequence would leave `docs/MUSIC/SCORING_SYSTEM.md` §4.2's "notes with no `expectedTimeMs` are excluded from timing scoring entirely" rule producing a partially-scored, hard-to-explain result no MVP lesson needs. |
| 9 | If any `expectedTimeMs` is present, `tempoBpm` is required. | `docs/CONTENT/LESSON_ARCHITECTURE.md` §4.3 (demonstration playback needs a rate). |
| 10 | `scoringWeights.correctness + timing + consistency` sums to `1.0`, tolerance `±0.001`. | `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14, verbatim. |
| 11 | If `expectedSequence` has no `expectedTimeMs` on any note (or `type === 'orientation'`/`'chord'` — the latter per `docs/MUSIC/CHORD_SYSTEM.md` §4.1, chord timing is judged by clustering, not `expectedTimeMs`), `scoringWeights.timing` and `scoringWeights.consistency` must both be `0`. | `docs/MUSIC/SCORING_SYSTEM.md` §4.2, as extended by this batch — see `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md` §How to choose ScoringWeights. |
| 12 | `passThreshold` is in `[0, 1]`. | `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2. |
| 13 | `hints` has at least one entry. | `docs/CONTENT/LESSON_ARCHITECTURE.md` §4.4 — a lesson with an empty `hints` array would silently break the "never a hard stop" escalation requirement (PR-LESSON-05) the moment a learner needs a second attempt. |
| 14 | `tempoBpm`, when present, is a positive integer in a musically sane range (this document recommends `40`–`208` — standard largo-to-prestissimo bounds) rather than an unbounded number a typo could produce. | This document — a sanity bound, not extracted from `docs/PRODUCT_SPEC.md`, flagged **NON-BLOCKING** since it only guards against an obvious authoring typo, not a real design decision. |

A file failing any rule above fails the seed script's run entirely (non-zero exit, CI fails) — per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14, this is "content validation at build/seed time, not runtime," so a malformed lesson never reaches a learner in the first place.

## 5. Data structures

This document's schema and `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`'s `Lesson`/`ExpectedNote`/`ScoringWeights` are field-for-field identical except for `hints` (§4.2, proposed). No other divergence exists — a content file, once validated, is loaded into Firestore as a `Lesson` document with no field transformation beyond Firestore's own `Timestamp`/plain-object handling (which doesn't apply here — `Lesson` carries no `Timestamp` field).

## 6. Interfaces / contracts

**Recommended Zod shape** (implementation detail — `scripts/seedLessons.ts`'s actual concern, sketched here only to make §4.4's rules concrete, not as a contract another package depends on):

```typescript
const ExpectedNoteSchema = z.object({
  midiNote: z.number().int().min(0).max(127),
  expectedTimeMs: z.number().int().min(0).optional(),
});

const ScoringWeightsSchema = z.object({
  correctness: z.number().min(0).max(1),
  timing: z.number().min(0).max(1),
  consistency: z.number().min(0).max(1),
}).refine(w => Math.abs(w.correctness + w.timing + w.consistency - 1) <= 0.001);

const LessonContentSchema = z.object({
  id: z.string(),
  title: z.string(),
  level: z.number().int().positive(),
  type: z.enum(['orientation', 'noteId', 'octave', 'melody', 'rhythm', 'scale', 'chord']),
  expectedSequence: z.array(ExpectedNoteSchema),
  tempoBpm: z.number().int().min(40).max(208).optional(),
  instructions: z.string(),
  prerequisiteLessonId: z.string().nullable(),
  scoringWeights: ScoringWeightsSchema,
  passThreshold: z.number().min(0).max(1),
  hints: z.array(z.string()).min(1),
});
```

Cross-file rules (§4.4's #2, #3, #4, #5) run separately, over the whole `content/lessons/` set at once, after every individual file passes `LessonContentSchema` — a single file can't validate its own uniqueness or chain-acyclicity in isolation.

## 7. Dependencies

`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` (`Lesson`, `ExpectedNote`, `ScoringWeights` — the shapes this schema mirrors); `docs/ENGINEERING/CODING_STANDARDS.md` §3.1 (Zod recommendation); no runtime dependency of its own — this is a content format, not code.

## 8. Error handling

- A file failing schema validation (§Interfaces) or a cross-file rule (§4.4 #2–5) fails the seed script with a specific, actionable error naming the file and the exact rule violated — never a generic "seed failed" with no pointer to which file or field.
- A `hints` array present but empty (§4.4 #13) is a validation failure, not a runtime fallback — this document does not define a "no hints available" UI state, because content authoring should never produce one.

## 9. Testing requirements

- Every rule in §4.4's table, each as its own test case against a synthetic fixture file (a deliberately-broken lesson JSON per rule), asserting the seed script rejects it.
- The worked example (§4.3) itself, asserted to pass validation cleanly — a positive-case test alongside the negative ones above.
- `docs/CONTENT/BEGINNER_CURRICULUM.md`'s actual 14 files, run through this validation as part of CI, not just as a one-time manual check when they were first authored — a later edit to any one of them must re-validate against the whole set (uniqueness, chain-acyclicity) every time.

## 10. Acceptance criteria

- [ ] `scripts/seedLessons.ts` rejects a `ScoringWeights` that doesn't sum to `1.0` at seed time, not runtime (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14).
- [ ] `scripts/seedLessons.ts` rejects a broken/cyclic `prerequisiteLessonId` chain.
- [ ] The worked example in §4.3 validates cleanly and, once seeded, round-trips through Firestore as a `Lesson` object with no field loss (`hints` aside — see §4.2 on that field's provisional status).

## 11. Out of scope

- Which 14 lessons exist and their actual content → `docs/CONTENT/BEGINNER_CURRICULUM.md`
- Runtime unlocking/hint-escalation/completion behavior → `docs/CONTENT/LESSON_ARCHITECTURE.md`
- Tone/style guidance for `instructions`/`hints` prose → `docs/CONTENT/CONTENT_AUTHORING_GUIDELINES.md`
- The scoring formulas `scoringWeights` parameterizes → `docs/MUSIC/SCORING_SYSTEM.md`
- `Exercise`/drill content (a different, non-Firestore, non-this-schema static config) → `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.3

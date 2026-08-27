# Future Roadmap

## 1. Purpose

Restates `docs/PRODUCT_SPEC.md` §8 (Future Roadmap) as its own standalone document. Nothing here ships in MVP — see `docs/PRODUCT/MVP_SCOPE.md` for the authoritative in/out boundary. This document exists so the future plan can be referenced, estimated against, and revised without editing the MVP boundary document itself.

## 2. Full Roadmap Table

Reproduced exactly as `PRODUCT_SPEC.md` §8, same phase/value/complexity structure and same notes.

| Feature | Phase | Value | Complexity | Note |
|---|---|---|---|---|
| MIDI keyboard support | 2 | Medium | Low | Web MIDI API is mature outside Safari; straightforward once the engine already does note detection |
| Song learning module | 2 | High | Medium | Gate on public-domain/original content only until a licensing conversation happens |
| Expanded chords & scales | 2 | Medium | Low | Mechanical extension of the deterministic engine (§13) |
| Gemini-personalized practice plans | 2 | High | Medium | Needs real usage data first (see §7) |
| PWA + offline lesson caching | 2 | Medium | Medium | See §23 for what's realistically cacheable |
| Achievements beyond streak/XP | 2 | Low–Medium | Low | Keep restrained — brief correctly warns against feeling childish |
| Admin CMS for content | 3 | Medium | Medium | Only worth it once lesson volume makes hand-editing JSON painful |
| Microphone-based audio input | 3 | High | High | Real acoustic-piano pitch detection is a genuinely hard, novel feature — validate demand first |
| Progress reports | 3 | Low–Medium | Low | Cheap once the data model (§17) already tracks the right fields |
| Native mobile app | 4 | Medium | High | Needs a proven web MVP first; don't fork effort early |
| Sight-reading / sheet music | 4 | High | High | A different product surface; scope as its own spec when it's time |
| Multiplayer / social challenges | 4 | Low–Medium | Medium | Speculative — no evidence yet that beginners want competitive piano |
| Human-teacher marketplace | 4 | Unclear | High | Needs its own business case before any engineering |

## 3. By Phase

Grouped for readability. Content is identical to §2 above — this is a view, not a second source of truth.

### Phase 2 — Straightforward extensions of a proven MVP

Prerequisite for all Phase 2 work: a shipped, validated MVP. Every item below is either a mechanical extension of something MVP already built (expanded chords/scales, MIDI) or explicitly gated on data/decisions MVP doesn't yet have (Gemini-personalized plans need usage data; the song module needs a licensing conversation).

| Feature | Value | Complexity | Note |
|---|---|---|---|
| MIDI keyboard support | Medium | Low | Web MIDI API is mature outside Safari; straightforward once the engine already does note detection |
| Song learning module | High | Medium | Gate on public-domain/original content only until a licensing conversation happens |
| Expanded chords & scales | Medium | Low | Mechanical extension of the deterministic engine (§13) |
| Gemini-personalized practice plans | High | Medium | Needs real usage data first (see §7) |
| PWA + offline lesson caching | Medium | Medium | See §23 for what's realistically cacheable |
| Achievements beyond streak/XP | Low–Medium | Low | Keep restrained — brief correctly warns against feeling childish |

### Phase 3 — Operational maturity and harder technical bets

| Feature | Value | Complexity | Note |
|---|---|---|---|
| Admin CMS for content | Medium | Medium | Only worth it once lesson volume makes hand-editing JSON painful |
| Microphone-based audio input | High | High | Real acoustic-piano pitch detection is a genuinely hard, novel feature — validate demand first |
| Progress reports | Low–Medium | Low | Cheap once the data model (§17) already tracks the right fields |

### Phase 4 — Speculative / different product surfaces

Every item in this phase either needs a proven web MVP first (native app), is a genuinely different product (sight-reading), is explicitly speculative with no supporting evidence yet (multiplayer), or needs its own business case before any engineering starts (teacher marketplace).

| Feature | Value | Complexity | Note |
|---|---|---|---|
| Native mobile app | Medium | High | Needs a proven web MVP first; don't fork effort early |
| Sight-reading / sheet music | High | High | A different product surface; scope as its own spec when it's time |
| Multiplayer / social challenges | Low–Medium | Medium | Speculative — no evidence yet that beginners want competitive piano |
| Human-teacher marketplace | Unclear | High | Needs its own business case before any engineering |

## 4. Notes on Items Not Explicitly Phased

`docs/PRODUCT_SPEC.md` §18 (Authentication) mentions that additional auth providers (e.g., Apple Sign-In) are additive and low-effort to add later, but §8's roadmap table does not list this as a numbered phase item. Rather than inventing a phase assignment, this is carried in `docs/PRODUCT/PRODUCT_REQUIREMENTS.md` as `PR-AUTH-10`, tagged `[Future — Phase unspecified]`. Flagged as a non-blocking open question — worth a deliberate phase assignment whenever this roadmap is next revised.

## 5. Non-Negotiable Rule Still Applies to Every Future Item

Nothing on this roadmap changes the boundary in `docs/PRODUCT_SPEC.md` §14: Gemini-personalized practice plans (Phase 2) still consume `ExerciseVerdict`/`AIMemorySummary` data the engine produced — Gemini does not gain scoring, correctness, or completion authority just because personalization becomes richer. If a future implementer designs the Phase 2 personalized-plan feature such that Gemini starts deciding what counts as a "weak area" from raw note data rather than from the engine's own aggregation, that is a violation of this rule, not a natural evolution of it.

## 6. Cross-References

- `docs/PRODUCT/MVP_SCOPE.md` — the authoritative in/out boundary this roadmap sits outside of.
- `docs/PRODUCT/PRODUCT_REQUIREMENTS.md` — every roadmap item also has a stable `[Future — Phase N]`-tagged requirement ID for traceability.
- `docs/PRODUCT_SPEC.md` §8 — original source; this document is a restatement, not a reinterpretation.

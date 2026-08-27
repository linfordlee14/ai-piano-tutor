# Analytics

## 1. Purpose

What product-usage data AI Piano Tutor measures, the tooling used to measure it, and the privacy boundary that governs what is — and is never — sent to the analytics layer.

Source of truth: `PRODUCT_SPEC.md` §25.

## 2. Scope

Product-usage analytics only — DAU/WAU, funnel, engagement metrics. Excludes operational health monitoring (`docs/OPERATIONS/MONITORING.md`), cost-specific tracking (`docs/OPERATIONS/COST_MONITORING.md`), and the AI cost-control levers themselves (`docs/AI/AI_COST_CONTROL.md`, parallel batch).

## 3. What to Measure

`PRODUCT_SPEC.md` §25's MVP-appropriate subset of the brief's fuller analytics list:

- Daily and weekly active learners (DAU/WAU)
- Lesson completion rate
- Practice session count and duration
- Drop-off point in the lesson funnel
- Most-missed notes/chords
- AI tutor usage **volume** — call counts, explicitly not content (see §5)

### 3.1 A Data-Sourcing Distinction Worth Making Explicit

Most of the list above (DAU/WAU, funnel drop-off, session count/duration) is naturally sourced from client-side event tracking — screen views, lesson starts/completions, taps — flowing through the Google Analytics SDK integration (§4).

**Most-missed notes/chords is architecturally different.** `PRODUCT_SPEC.md` §25 is explicit about this: *"this is essentially free — it's already computed for the AI memory doc in section 17, just aggregate it."* That data already exists server-side, computed by Cloud Functions as part of `AIMemorySummary` recomputation (`weakNotes[]`, `weakChords[]` — `TYPES_AND_CONTRACTS.md` §4.8), sourced from the `practiceSessions` log (`docs/DATA/DATA_MODEL.md` §5). Producing this specific metric means aggregating across users' existing Firestore data (most practically via the BigQuery export path, §4, or a scheduled aggregation Function) — **not** instrumenting a new client-side analytics event for it. Treat this one metric as a Firestore-sourced aggregation job, distinct in mechanism from every other metric on this list.

## 4. Tooling

- **Firebase's built-in Google Analytics integration** is sufficient for MVP funnel/engagement tracking. No third-party analytics SDK before there's a specific reason to add one (`PRODUCT_SPEC.md` §25).
- **BigQuery export** (built into Firebase) is the natural upgrade path if deeper querying is ever needed — and is also the practical mechanism for the most-missed-notes/chords aggregation described in §3.1, since that metric needs to query already-persisted Firestore data rather than a fresh client event stream.
- Do not build a custom analytics pipeline for a pre-launch product (`PRODUCT_SPEC.md` §25, explicit instruction).

## 5. Privacy Boundary — Restated Explicitly

`PRODUCT_SPEC.md` §25, stated as a rule: *"collect product-usage events only (screen views, lesson starts/completions, taps) — never send tutor conversation content or the specific notes a learner played into the general analytics stream. That data already lives correctly-scoped in Firestore under the user's own record; duplicating it into an analytics tool would only create a second, less-protected copy."*

This means, concretely:

- **Allowed in the analytics stream:** that a lesson was started/completed, that the tutor was invoked (a count), that a screen was viewed, session duration/counts, funnel-stage transitions.
- **Never in the analytics stream:** the text of any tutor conversation, the specific MIDI notes a learner played, or anything from `PracticeSession.verdict` beyond an aggregate rollup already computed server-side (§3.1's most-missed-notes/chords metric is the one sanctioned exception, and even that is an aggregate across a population, not a per-user raw event stream).

The reasoning stated in §25 is worth carrying forward precisely: this isn't only a privacy-policy compliance point, it's a data-architecture point — the correctly-scoped, access-controlled copy of this data already exists in Firestore (`DATA_ACCESS_RULES.md` governs who can read it there); a second copy in a general-purpose analytics tool would be a materially weaker-protected duplicate of the same sensitive data, not a new capability.

**Cross-reference `docs/SECURITY/PRIVACY.md`** (parallel batch, does not yet exist at the time of this document) — this boundary should be re-verified against that document once written, and any consent/disclosure language it defines should match what's actually collected here.

## 6. Out of Scope

- Operational monitoring (Cloud Functions error rates, Gemini failure rates) — `docs/OPERATIONS/MONITORING.md`.
- Cost-specific dashboards and alert thresholds — `docs/OPERATIONS/COST_MONITORING.md`.
- The architectural cost-control levers for Gemini specifically — `docs/AI/AI_COST_CONTROL.md` (parallel batch).
- Full privacy/consent policy — `docs/SECURITY/PRIVACY.md` (parallel batch).

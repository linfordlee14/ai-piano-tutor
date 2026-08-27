# Cost Monitoring

## 1. Purpose

The **operational** view of AI Piano Tutor's cost exposure: what to watch in production, and a proposed starting alert threshold. This document is the dashboards-and-alerts complement to `docs/AI/AI_COST_CONTROL.md`'s (parallel batch) architectural view — the design levers (model tiering, caching, rate caps) that keep Gemini spend bounded in the first place.

Source of truth: `PRODUCT_SPEC.md` §31.

## 2. Scope

What to instrument and watch, and how to think about alert thresholds. Excludes the cost-control *design* itself (`docs/AI/AI_COST_CONTROL.md`), general operational health monitoring (`docs/OPERATIONS/MONITORING.md`), and current Firebase/Gemini pricing numbers (`PRODUCT_SPEC.md` §31 explicitly warns against hardcoding these — "check current free-tier limits at build time rather than trusting a number in this document" — a discipline this document follows too).

## 3. The Cost Picture

`PRODUCT_SPEC.md` §31: **Firebase (Auth/Firestore/Hosting/Functions)** free tiers are generous enough that, at a few hundred early users, cost is likely near-zero outside Gemini spend itself. **Gemini is the real variable cost.** The architectural levers that bound it are already documented (or will be, in the parallel batch) in `docs/AI/AI_COST_CONTROL.md`: calls gated to explicit user action only, cheapest model tier for high-volume templates, caching of static theory Q&A, hard per-user daily rate caps, and short structured-context prompts instead of raw transcripts. This document does not re-derive those levers — it covers watching whether they're actually holding, in production, over time.

## 4. Instrumentation — Day One, Not After a Bill Surprise

`PRODUCT_SPEC.md` §31's explicit recommendation: *"instrument Gemini spend from day one, even in MVP — a simple per-call log with token counts, not just a monthly bill review. Catching a cost regression in week one is cheap; catching it after a usage spike is not."*

Concretely, every Gemini call (both templates — Exercise Feedback and Tutor Q&A, per `PRODUCT_SPEC.md` §15) should log at minimum: which template/tier was used, input and output token counts, whether it was served from cache (`TutorResponse.cached`, `TYPES_AND_CONTRACTS.md` §4.10, already carries this exact signal for the tutor path) or was a live call, and the requesting user (for per-user rate-cap verification, not for any analytics purpose — this log is an operational/cost artifact, not a product-analytics event; keep it out of the general analytics stream per `docs/OPERATIONS/ANALYTICS.md` §5's privacy boundary).

**Dashboard/aggregation mechanism (PROPOSED — `PRODUCT_SPEC.md` specifies the instrumentation but not the aggregation/visualization layer):** at MVP scale, a scheduled Cloud Function that reads the per-call log and rolls it up daily (call count, total tokens, estimated spend by template/tier) is enough — either surfaced as a simple internal report or exported via BigQuery (the same upgrade path already named for product analytics, `docs/OPERATIONS/ANALYTICS.md` §4) if querying needs grow. A dedicated cost-observability dashboard tool is not warranted at this scale; a periodic query or a lightweight scheduled summary is proportionate.

## 5. What to Actually Watch For — Per PRODUCT_SPEC.md's Own Framing

`PRODUCT_SPEC.md` §31 is specific and worth carrying forward in intent: *"The thing to actually watch for isn't the per-call price — it's a bug that quietly removes one of the guardrails (e.g., a retry loop that fires Gemini calls in a tight loop, or a cache that silently stops hitting)."*

This translates into two concrete monitoring signatures, not just a raw dollar total:

- **Call-volume-to-user-activity ratio.** Gemini calls are gated to explicit user action only (§14) — call volume should track active-user/session counts roughly proportionally. A call-volume spike disproportionate to actual user activity is the signature of a retry loop, not organic growth.
- **Cache hit rate for theory Q&A.** §15 names this as "likely the single highest-leverage cost optimization in the whole AI layer." A dropping cache-hit rate on the `(concept, skillLevel)`-keyed theory cache is the signature of a silently broken cache, which would otherwise show up only as a slow, unexplained cost creep.

## 6. Alert Threshold — PROPOSED, Not Specified in PRODUCT_SPEC.md

**`PRODUCT_SPEC.md` gives no specific dollar figure or percentage for a cost alert.** §31's own "illustrative math, explicitly not a quote" caveat applies with equal force here — proposing a hardcoded dollar ceiling in this document would carry the same staleness risk the spec itself warns against for pricing numbers.

**Proposal (PROPOSED — needs founder sign-off, not extracted fact):**

- There is no real usage baseline before launch. Consistent with `PRODUCT_SPEC.md` §7's own reasoning for deferring AI-personalized practice plans ("personalization needs enough historical performance data to be accurate; a brand-new user has none"), the same logic applies to a spend baseline: don't invent a number before there's real traffic to measure. Once a trailing baseline exists (even a rough one, after the first week or two of real usage), set a **relative alert**: flag any day where Gemini spend exceeds roughly 3× the trailing 7-day average.
- In parallel, set a simple **absolute backstop** — a hard dollar ceiling the founder is comfortable naming as a "something is badly wrong, stop and look" number, active from day one (not waiting for a baseline), specifically to catch a bug in the very first days of usage before any baseline exists at all.
- Both thresholds are illustrative starting points, not calibrated numbers — the founder should set the actual figures based on their own risk tolerance and the (still-unverified-in-this-document) current Gemini pricing at implementation time.

## 7. Out of Scope

- The architectural cost-control levers (model tiering, caching design, rate-cap implementation) — `docs/AI/AI_COST_CONTROL.md` (parallel batch).
- Firestore/Functions free-tier limits and Gemini per-token pricing — deliberately not restated here; `PRODUCT_SPEC.md` §31 itself instructs checking these at build time rather than trusting a static document.
- General operational monitoring (Functions error rates, Gemini failure rates as a reliability signal rather than a cost signal) — `docs/OPERATIONS/MONITORING.md`.
- Product-usage analytics — `docs/OPERATIONS/ANALYTICS.md`.

# Monitoring

## 1. Purpose

What to monitor operationally once AI Piano Tutor is deployed: Cloud Functions health, the one external-dependency failure mode the whole architecture is built to tolerate (Gemini), and Firestore volume. A new document — `PRODUCT_SPEC.md` does not cover operational monitoring in detail — scoped to an MVP-appropriate baseline, not a third-party observability build-out.

## 2. Scope

What to watch and why, and the tooling to watch it with. Excludes cost-specific dollar thresholds and alerting (`docs/OPERATIONS/COST_MONITORING.md`), incident response procedure (`docs/OPERATIONS/INCIDENT_RESPONSE.md`), and the AI layer's own fallback design (`docs/AI/AI_ERROR_HANDLING.md`, parallel batch).

## 3. What to Monitor

### 3.1 Cloud Functions Error Rates and Latency

Every scored exercise, every tutor question, and both account-lifecycle operations (`deleteAccount`, `exportUserData`) go through a callable Cloud Function (`PRODUCT_SPEC.md` §20). A rising error rate or latency regression on `getExerciseFeedback` specifically is a direct hit on the core loop — it's the function that validates note events server-side *and* writes the session/progress documents (§20), so a failure there doesn't just degrade the tutor, it means a learner's completed exercise silently fails to save.

Watch, per Function: invocation count, error rate, and p50/p95 latency — available directly from Cloud Functions' built-in metrics (§4).

### 3.2 Gemini Call Failure Rate — The One External Dependency This Architecture Is Built to Tolerate

`PRODUCT_SPEC.md`'s design principle, stated repeatedly and specifically (§2, §10, §14, §29): if Gemini is down, the rest of the app keeps working. The tutor screen shows "a clear 'temporarily unavailable, here's what to practice instead' fallback card built from deterministic data" (§10), and §29's acceptance criteria require this tested explicitly: *"Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly, not assumed."*

**This is exactly why Gemini failure rate needs active monitoring, not just a one-time pre-launch test.** The architecture is specifically designed so a Gemini outage degrades *gracefully and quietly* — which means, without monitoring, a real outage could persist unnoticed by the team for a long time even while every affected learner is correctly seeing a fallback card. The graceful-degradation design is a UX guarantee, not an operational visibility guarantee; monitoring is what closes that gap.

Watch: `getTutorResponse` and `getExerciseFeedback`'s Gemini-call-specific error/timeout rate (distinct from the Function's overall error rate — a Function can fail purely on the Gemini leg while everything else in it works), and the rate at which the client renders the fallback card versus a real tutor response. Cross-reference `docs/AI/AI_ERROR_HANDLING.md` (parallel batch, not yet written) for the fallback mechanism's exact design; this document covers observing that it's firing correctly and how often, not designing it.

A useful secondary signal, directly from `PRODUCT_SPEC.md` §22's own stated expectation: Gemini calls are gated to explicit user actions and "1–3 seconds is acceptable there as long as the UI shows a clear 'thinking' state." A sustained latency rise beyond that window is itself worth watching, separate from outright failures — a slow-but-technically-working Gemini call degrades the experience in a way a hard failure (which triggers the designed fallback) does not.

### 3.3 Firestore Read/Write Volume

For cost tracking primarily (see `docs/OPERATIONS/COST_MONITORING.md` for the dollar angle), but also as an operational health signal in its own right: a sudden spike in `practiceSessions` writes or `users/{uid}` reads can indicate a retry-loop bug, a script-driven abuse pattern the rate-limiting in `PRODUCT_SPEC.md` §19 didn't fully catch, or simply real organic growth worth knowing about before the monthly bill reveals it. Watch aggregate read/write counts per collection, available via the Firebase console's Firestore usage tab.

## 4. Tooling — Firebase's Built-In Console Monitoring Is Sufficient for MVP Scale

**Do not recommend a third-party observability tool for MVP.** This follows the same discipline `PRODUCT_SPEC.md` applies elsewhere in the stack — §21's "choose only if needed" for state management, and §32's explicit "deliberately excluded" list (Redux, GraphQL, a separate REST API layer, a third-party analytics SDK — "each would add a dependency without a corresponding need at this stage"). Operational monitoring tooling deserves the same test.

Cloud Functions (2nd gen) run on Cloud Run under the hood and surface standard metrics directly in both the Firebase console and the underlying Google Cloud Monitoring — error rates, invocation counts, latency percentiles, and logs, with no separate integration required. Firestore's own usage dashboard in the Firebase console covers read/write volume. This is sufficient for a solo-founder-stage MVP; revisit only if genuine gaps appear (e.g. needing alerting sophistication or retention beyond what the console provides), not preemptively.

## 5. Signals Worth a Deliberate Look (PROPOSED — MVP-Appropriate Baseline, Not Extracted From Spec)

`PRODUCT_SPEC.md` does not prescribe specific monitoring thresholds; the following is a reasonable starting baseline, offered as a proposal:

- A Cloud Functions error-rate spike on `getExerciseFeedback` (the scoring/progress-write path) — any sustained rate above near-zero is worth a look, since a failure here silently loses a learner's completed work.
- A Gemini-call failure or timeout rate that departs meaningfully from its recent baseline — the specific signal that the fallback card (§3.2) is firing more than usual, which the team should know about even though users are, by design, not blocked by it.
- `getTutorResponse`/`getExerciseFeedback` latency exceeding the ~1–3 second Gemini-call expectation (§3.2) by a wide and sustained margin.
- A Firestore read/write volume trend that departs sharply from active-user growth — see `docs/OPERATIONS/COST_MONITORING.md` §5 for the specific abuse-pattern signatures (retry loops, cache misses) this should be able to catch.

## 6. Out of Scope

- Dollar-denominated cost alert thresholds — `docs/OPERATIONS/COST_MONITORING.md`.
- What happens once an incident is identified (escalation, response, rollback) — `docs/OPERATIONS/INCIDENT_RESPONSE.md`.
- The AI layer's fallback design itself (as opposed to observing it) — `docs/AI/AI_ERROR_HANDLING.md` (parallel batch).
- Product-usage analytics (DAU/WAU, funnel drop-off) — `docs/OPERATIONS/ANALYTICS.md`, a distinct concern from operational health monitoring.

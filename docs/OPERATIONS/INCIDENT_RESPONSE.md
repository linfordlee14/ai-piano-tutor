# Incident Response

## 1. Purpose

An MVP-appropriate, solo-founder-stage incident checklist — not a formal SRE runbook. Defines what counts as an incident for AI Piano Tutor, in three severity tiers grounded directly in the architecture's own stated design (`PRODUCT_SPEC.md` §2, §10, §14, §19, §23, §26, §29), and a short response checklist for each.

## 2. Scope

Three specific incident categories, matched to how seriously the architecture itself treats each dependency's failure. Excludes routine operational monitoring (`docs/OPERATIONS/MONITORING.md`), cost-overrun handling (`docs/OPERATIONS/COST_MONITORING.md` — related but distinct, see §6), and a formal postmortem process (deliberately out of scope at this stage, see §5).

## 3. Severity Tiers

### 3.1 Gemini Fully Down — Should Be a Non-Event

**This is the architecture's own explicit design intent, not an aspiration.** `PRODUCT_SPEC.md`'s design principle, stated repeatedly (§2: "if Gemini is down, the piano still works"; §14; §29's acceptance criterion: "Disabling the Gemini Function entirely leaves every other screen fully functional — tested explicitly, not assumed"): the piano, lessons, and progress tracking have zero dependency on Gemini. Only the tutor question itself is affected, and even that has a designed fallback — §10's "temporarily unavailable, here's what to practice instead" card, "built from deterministic data — never a dead end."

**What this means operationally:** a Gemini outage does not, by itself, warrant urgent action. The correct response is to confirm the designed fallback is actually rendering correctly (a check against `docs/OPERATIONS/MONITORING.md` §3.2's Gemini-failure-rate signal, and, ideally, an actual look at the tutor screen) — not to treat it as a production-down emergency.

**Checklist:**
- [ ] Confirm via monitoring (`MONITORING.md` §3.2) that this is a genuine Gemini-side outage, not a bug in the Cloud Function's own Gemini-calling code.
- [ ] Confirm the tutor screen is showing the designed fallback card, not a broken/blank screen — if it's broken, this is actually a bug in the fallback mechanism itself, which *is* worth treating with more urgency, since it breaks the "never a dead end" guarantee (§10) the whole design depends on.
- [ ] Confirm every other screen — piano, lessons, dashboard, progress — is unaffected, per the §29 acceptance criterion this should already have been tested against.
- [ ] No rollback, no urgent fix required if the above all hold. Note it, watch for Gemini's own status/recovery, move on.

### 3.2 Firestore or Auth Down — Genuinely Degrades the Product

Unlike Gemini, Firestore and Auth are load-bearing for the core loop, not optional. `PRODUCT_SPEC.md` §23's offline strategy gives a real, but partial, mitigation: Firestore's built-in offline persistence (`enableIndexedDbPersistence`) means a learner already mid-session, with already-cached content, keeps working locally and their results queue for sync on reconnect — the degraded-service banner from §10 ("Playing offline — your progress will sync later") is the designed, honest signal for this case.

**This mitigation has real limits, worth being honest about rather than assuming full coverage:** a first-time visitor, or any learner needing content that was never previously cached locally, is much more seriously affected — they cannot sign in, and cannot load lesson content that isn't already in the local cache. Auth being down specifically blocks anyone not already in an active, cached session. Both Firestore and Auth are Google-operated services outside this project's direct control — there is no self-inflicted fix to deploy; the response is monitoring status, communicating clearly, and waiting.

**Checklist:**
- [ ] Check Google Cloud/Firebase status page to confirm scope (is this a Google-side outage, or something specific to this project's configuration/quota?).
- [ ] Verify the degraded-service banner (§10) is actually appearing for affected users, not a silent broken state.
- [ ] If this is project-specific (e.g. a quota limit hit, a misconfigured rule blocking legitimate reads) rather than a genuine Google-side outage, that's a different, faster-fixable problem — check Firestore usage/quota in the console before assuming a full outage.
- [ ] No action is possible against a genuine Google-side outage beyond monitoring and communicating — this is expected to be rare and outside this project's control.

### 3.3 Security-Rules Regression Allowing Client Score-Tampering — Most Serious, Treat as Urgent

This is the one category where genuine urgent action is warranted, because it breaks the trust model the entire product is built on. `PRODUCT_SPEC.md` is explicit and repeated about why: §14/§17/§19's whole write-ownership design exists specifically so "a modified client cannot self-report a perfect score" — and this documentation batch's own non-negotiable framing states it plainly: *Gemini never determines correctness/scores/XP/streaks — all persisted scoring data is written exclusively by Cloud Functions after deterministic server-side validation, never by the client, never by Gemini.* `PRODUCT_SPEC.md` §26 singles this out precisely, describing the Firestore rules unit tests that assert "a client cannot write its own score and cannot write lesson content" as *"the two rules where a silent regression would become a real cheating or content-integrity bug"* — and §29's acceptance criteria restate the same guarantee as a launch-blocking check: *"A client cannot write its own `practiceSessions`/progress/AI-memory documents directly (verified by a rules test, not a manual check)."*

A Firestore Security Rules deploy that accidentally opens a client write path to `practiceSessions`, `progressSummary`, `aiMemory/**`, or `lessons` doesn't just risk one user's data — it lets any user (or bot) fabricate scores/XP/streaks or corrupt public lesson content, and every derived view built on top of that data (dashboard, AI tutor context, analytics aggregates in `docs/OPERATIONS/ANALYTICS.md`) inherits the corruption silently.

**Checklist:**
- [ ] **Roll back immediately** to the last known-good `firestore.rules` — this is a rollback, not a forward-fix; correctness of the fix matters more than speed of a patch attempt under pressure.
- [ ] Redeploy the corrected rules via the Firebase CLI (`docs/OPERATIONS/DEPLOYMENT.md` §3).
- [ ] Treat any data written during the exposure window as suspect — identify the window precisely (deploy timestamp of the bad rules to the rollback timestamp) and review `practiceSessions`/`progressSummary`/`xp`/`streak` writes in that window for anomalies (a spike in perfect scores, XP jumps inconsistent with plausible practice time). This is a real remediation step, not just closing the hole — affected users' data may need manual correction.
- [ ] Verify (or write, if missing) the specific rules-unit-test that should have caught this before it shipped — `PRODUCT_SPEC.md` §26 names the tool directly: `@firebase/rules-unit-testing`, asserting explicitly that a client cannot write its own score and cannot write lesson content. A regression reaching production is itself a signal that this test either doesn't exist, doesn't run in CI, or doesn't cover the specific path that broke — fix the test gap, not just the rule.
- [ ] Confirm App Check (§16/§19) wasn't also silently misconfigured in the same change — a rules regression and an App Check regression are different bugs, but worth checking together since both are "deploy-time security configuration."

## 4. Communication

For §3.1 and §3.2 specifically, the product already communicates degraded state to users directly — the tutor fallback card and the offline/degraded-service banner (§10) are user-facing by design. This meaningfully reduces the operational communication burden at this stage: there is no need for a separate public status page at MVP scale, since the in-product messaging is already doing that job for the two most likely failure modes. §3.3 (a security regression) has no equivalent user-facing signal — communication there, if any is warranted, is a founder judgment call scoped to whether affected users need to be told their data may have been exposed to tampering, not a routine status update.

## 5. Proportionality

This checklist is deliberately short and does not include: a formal severity-1/2/3 classification scheme beyond the three tiers above, a paging/on-call rotation, or a blameless-postmortem template. A solo-founder-stage product with the user base implied by `PRODUCT_SPEC.md`'s "a few hundred early users" (§31) does not need SRE-rotation-grade process, and building one prematurely would be the same category of mistake `PRODUCT_SPEC.md` warns against elsewhere — adding infrastructure before there's a reason (§21's "choose only if needed," §32's "deliberately excluded" dependency list).

## 6. Related but Distinct: Cost Incidents

A cost-alert threshold breach (`docs/OPERATIONS/COST_MONITORING.md` §6) is a related trigger worth a light mention here, but it is not one of the three severity tiers above — a cost spike doesn't degrade the product for any user the way §3.1–§3.3 do. Treat it as its own lightweight check-in against `COST_MONITORING.md` §5's specific signatures (retry loop, broken cache), not folded into this incident checklist.

## 7. Out of Scope

- Formal postmortem process and templates — disproportionate at this stage; revisit if/when the team and user base grow enough to warrant it.
- Routine monitoring mechanics (what's watched, how) — `docs/OPERATIONS/MONITORING.md`.
- Cost-specific alerting — `docs/OPERATIONS/COST_MONITORING.md`.
- Deployment/rollback mechanics in general (as opposed to the specific §3.3 rollback trigger) — `docs/OPERATIONS/DEPLOYMENT.md`.

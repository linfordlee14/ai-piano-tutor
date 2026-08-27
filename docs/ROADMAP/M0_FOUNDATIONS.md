# M0 — Foundations

## 1. Objective

Prove the toolchain before any product feature exists. M0 produces an empty, deployed, CI-wired shell with default-deny security posture from its first commit — nothing about M0 is user-facing product value; all of it is infrastructure a coding agent should never have to re-derive or debug while also trying to build a feature.

**Source of truth:** `docs/PRODUCT_SPEC.md` §16 (Firebase Architecture), §19 (Security Architecture), §27 (Project Folder Structure), §28 (M0 row), §35 (Build Instructions). Milestone index: `docs/ROADMAP/MILESTONES.md`.

## 2. Prerequisites

None. M0 is the first milestone — it depends on nothing already built, only on the scaffolding/provisioning work `docs/PRODUCT_SPEC.md` §35 describes as step 1 (AI Studio Build or Antigravity provisioning Firestore, Auth, Functions, App Hosting) having happened, with the resulting repository moved to GitHub as the system of record (§35 step 3) before any precision engineering work begins.

## 3. Exact Scope

- **Repository skeleton** matching `docs/PRODUCT_SPEC.md` §27's folder tree (see §4 below) — every top-level package/app directory exists, even where its contents are still stubs to be filled in by later milestones.
- **Next.js (App Router) + TypeScript app** (`apps/web`) deployed and reachable at a real URL, with a landing route and one placeholder authenticated-area route (the Dashboard shell).
- **Firebase project wired end-to-end:** Authentication (anonymous provider enabled and functional — see §6), Firestore (empty except for the rules file), Cloud Functions (2nd gen, TypeScript, deployed but with no product logic yet — a trivial health-check function is acceptable to prove the deploy path), Firebase App Hosting (the actual deployment target for `apps/web`), App Check (registered, even if not yet enforced on anything, since `docs/PRODUCT_SPEC.md` §16 calls it out as "cheap to add early, painful to retrofit").
- **CI/CD pipeline that deploys on push** to the main branch — this is a named, literal acceptance criterion (`docs/PRODUCT_SPEC.md` §28), not a nice-to-have.
- **`firestore.rules` establishing default-deny as the baseline posture from the first commit that contains a rules file at all** — every subsequent milestone adds explicit `allow` rules on top of this baseline as it introduces the collection those rules govern (M2 adds `lessons` public-read and the `practiceSessions`/`progressSummary` Function-only pattern once `getExerciseFeedback` exists; M3/M4 add nothing new to rules unless new paths are introduced; M5 hardens and verifies the full set). M0 itself does not need feature-specific `allow` rules — its job is to prove the *deny-by-default root* is in place, not to write every rule the product will ever need.
- **Empty Dashboard shell** matching the empty-state described in `docs/PRODUCT_SPEC.md` §10: *"first-ever visit shows 'Start your first lesson' instead of streak/stats."* Since no lesson exists yet (that's M2), this is a static empty-state render, not a functional CTA destination.

## 4. Files/Components Expected

Per `docs/PRODUCT_SPEC.md` §27 (cross-reference `docs/ENGINEERING/PROJECT_STRUCTURE.md`, parallel batch, for the ratified final layout — the paths below are §27's own tree, used directly since that document doesn't exist yet):

```
apps/web/
  app/(marketing)/            # landing route — pre-auth
  app/(app)/dashboard/        # empty dashboard shell route
  components/                 # minimal — shell layout only, no piano/lesson components yet
  hooks/                      # empty/stub
  stores/                     # empty/stub — Zustand wiring proven with a trivial store if useful,
                               # full engine/UI state (docs/PRODUCT_SPEC.md §21) is M1+ work
  lib/                        # Firebase client SDK initialization (apps/web's connection to
                               # Auth/Firestore) — this is real M0 work, not a stub
packages/
  music-engine/                # package skeleton (package.json, tsconfig) — NO chord/scale/
                               # scoring logic yet; that is entirely M1 scope
  shared-types/                # package skeleton — types from docs/ENGINEERING/TYPES_AND_CONTRACTS.md
                               # are added incrementally as each milestone needs them, not all at once
functions/
  src/                         # Cloud Functions skeleton — a trivial deploy-verification function
                               # is acceptable; getExerciseFeedback etc. are M2+/M4 scope
content/
  lessons/                     # empty directory — seed-script skeleton only; real lesson JSON is M2+
firestore.rules                # default-deny root rule
firestore.indexes.json         # empty/minimal
firebase.json                  # Firebase project config (Hosting, Functions, Firestore, emulator ports)
.github/workflows/              # (or equivalent CI config) — deploy-on-push pipeline
```

## 5. Technical Requirements

- Next.js App Router, TypeScript, deployed via Firebase App Hosting (`docs/PRODUCT_SPEC.md` §16, §32) — not a hand-rolled Cloud Run deployment.
- Cloud Functions are 2nd generation, TypeScript (§16).
- `packages/music-engine` exists as its **own workspace package**, not a folder inside `apps/web`, from the very first commit — `docs/PRODUCT_SPEC.md` §27 is explicit that this is what makes "independently testable" structurally true rather than a stated intention: the package "physically cannot import React or Firebase without that showing up as a visible, reviewable dependency violation." Getting the monorepo/workspace wiring right at M0 avoids a painful restructure once M1 starts writing real engine code into it.
- CI pipeline runs on every push to main (minimum), deploying `apps/web` and `functions/` together so they never drift out of sync with each other.

## 6. On Anonymous Auth Specifically

`docs/PRODUCT_SPEC.md` §28's M0 acceptance line reads "an anonymous user hits a deployed URL and sees an empty dashboard shell." This milestone spec reads that as requiring the **Firebase Auth anonymous provider to be functionally wired** (an anonymous session is created, `context.auth` is real and verifiable), not merely an unauthenticated visitor viewing a public page — because Firebase Auth is itself one of the toolchain pieces M0 exists to prove (§16), and because every later milestone's "try before signup" flow (§5, §10, §18) depends on anonymous auth already working reliably. This is a reasonable, spec-consistent elaboration of a one-line acceptance criterion, not a new product decision — flagged here so the reasoning is visible rather than assumed.

## 7. Tests Required

Per `docs/TESTING/TESTING_STRATEGY.md` §5 (Test Suite Map), only two categories are live at M0:

- **`docs/TESTING/SECURITY_TESTING.md`:** a rules-unit-test asserting the Firestore default-deny root actually holds — any read or write to any undeclared path, by any (or no) authenticated user, is rejected. This is the M0-specific slice of that document's full suite; the feature-specific rules (§5.1–5.6 of that document) light up as their collections are introduced in later milestones.
- **CI pipeline correctness itself:** verify a push to main actually triggers a deploy and that the deployed URL serves the expected shell — this is closer to an infrastructure smoke-check than a named suite in `docs/TESTING/`, but it is exactly what M0's acceptance criterion is asking to be proven.

No music-engine, integration, AI, E2E-journey, accessibility-enforcement, or performance-budget suite is meaningfully runnable yet — there is no feature for them to exercise. Do not write placeholder tests for these categories just to have coverage; add them for real starting at the milestone `docs/TESTING/TESTING_STRATEGY.md` §5 names for each.

## 8. Acceptance Criteria

Reproduced and expanded from `docs/PRODUCT_SPEC.md` §28's M0 row:

- [ ] An anonymous user (real Firebase Auth anonymous session, not just an unauthenticated page view) hits the deployed URL and sees the empty dashboard shell.
- [ ] CI deploys on push to main, without manual intervention.
- [ ] `firestore.rules` default-denies every path not explicitly listed, verified by a rules test, from the first commit that includes a rules file.
- [ ] The repository's top-level structure matches `docs/PRODUCT_SPEC.md` §27 (§4 above), including `packages/music-engine` as its own workspace package.
- [ ] Cloud Functions deploy successfully (even if trivial), proving the Functions half of the toolchain.

## 9. Definition of Done

Checked against `docs/ROADMAP/DEFINITION_OF_DONE.md` in full. Notes specific to M0: item 6 ("no secrets exposed") is already live even though no Gemini credential exists yet — any Firebase config exposed client-side must be the intentionally-public client config (API key etc. that Firebase's own model treats as non-secret when paired with Security Rules and App Check), never a service-account or Admin SDK credential. Item 3 ("no Phase-2+ feature leaked in") is easy to satisfy at M0 by construction — there's barely anything built yet — but worth stating so no one scaffolds, say, a `songs/` collection "while we're setting up Firestore anyway."

## 10. Out of Scope for This Milestone

- Any piano UI, audio, or input handling — that is M1.
- Any lesson content, scoring logic, or `music-engine` implementation — M1 (engine) and M2 (first lesson).
- Any Cloud Function with real product logic (`getExerciseFeedback`, `getTutorResponse`, `deleteAccount`, `exportUserData`) — M2, M4, M5 respectively.
- Feature-specific Firestore Security Rules beyond the default-deny root (`lessons` public-read, the `practiceSessions`/`progressSummary`/`aiMemory` Function-only pattern) — these are added milestone by milestone as their collections are introduced, starting at M2.
- Any Gemini integration whatsoever — M4. No Gemini credential, no Cloud Function calling Gemini, no `functions/src/ai/` content should exist at M0.
- MIDI input, additional scales/chords, song learning, admin CMS, offline PWA caching, microphone input, achievements beyond streak/XP — none of these are Phase 1 (MVP) at all (`docs/PRODUCT_SPEC.md` §7), and M0 in particular has no reason to touch any of them.

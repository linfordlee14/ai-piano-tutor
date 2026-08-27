# Project Structure

## 1. Purpose

The authoritative folder structure for AI Piano Tutor's repository — reproduced and expanded from `docs/PRODUCT_SPEC.md` §27 — and the explanation for the one structural decision in this repo that carries real architectural weight: why `packages/music-engine` is its own workspace package and not a folder inside `apps/web`.

`docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` and `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` both explicitly defer to this document as "the authority on exact paths" — this is that document. They own *why the application is shaped the way it is* at the architecture level; this document owns *where every piece of it physically lives*, and — for `packages/music-engine` specifically — why the physical location is itself part of the architecture, not just a filing convenience.

## 2. Scope

Covers: the top-level monorepo layout, what lives inside each workspace package and app, and the structural rationale for the two package boundaries that matter most (`packages/music-engine`, `packages/shared-types`).

Out of scope: the *internal* module/dependency rules of `packages/music-engine` (input adapter contract, the Phase-2 MIDI adapter design) — that's `docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md`'s territory; this document explains why the package boundary exists and what enforces it, not the algorithms inside. Also out of scope: exact lint/CI configuration syntax (`docs/ENGINEERING/CODING_STANDARDS.md` specifies the enforcement mechanism; this document explains what it's enforcing and why).

## 3. Architecture — the folder tree

Expanded from `docs/PRODUCT_SPEC.md` §27. Additions beyond the spec's own tree are marked `# +` and justified in the notes below the tree — nothing is added here that isn't a direct, traceable consequence of something PRODUCT_SPEC.md or `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` already requires.

```
piano-tutor/
├── apps/
│   └── web/                              # Next.js app (App Router)
│       ├── app/
│       │   ├── (marketing)/              # pre-auth landing — "Try the piano now" (§10)
│       │   ├── (app)/
│       │   │   ├── dashboard/            # Home tab (§9)
│       │   │   ├── learn/                # Learn tab — lesson path + Lesson Player (§9, §10)
│       │   │   ├── piano/                # Piano tab — free play, also the pre-auth try flow
│       │   │   ├── tutor/                # Tutor tab — AI Tutor chat
│       │   │   └── profile/              # Profile tab — settings + Progress/Stats (§9: lives here, not a top-level tab)
│       │   └── layout.tsx                # + root layout — this is where the degraded-service banner (§10) renders once, shell-level
│       ├── components/                   # UI layer — piano keys, lesson player shell, dashboard cards
│       │   └── piano/                    # + per-key components, memoized per-key/per-octave (§22)
│       ├── hooks/                        # Firestore subscription hooks, audio-engine hooks
│       │   ├── useUserProfile.ts         # + onSnapshot on users/{uid} (§4.2 below)
│       │   ├── useLessonPath.ts          # + onSnapshot on the lessons collection
│       │   └── useAudioEngine.ts         # + bridges Tone.js callbacks into the Zustand store
│       ├── stores/                       # Zustand stores (engine/UI state, §21) — one per concern, not one monolithic store
│       ├── lib/
│       │   ├── firebase/                 # + client SDK init (Auth, Firestore, Functions)
│       │   └── functions/                # + typed callable-function wrappers, one per function in §20 — see §5 below
│       └── public/
│           └── audio/samples/            # + Salamander samples served as static assets from the app's own CDN (§1 pushback, §12) — deliberately NOT Firebase Storage
├── packages/
│   ├── music-engine/                     # framework-agnostic: notes, chords, scales, scoring (§13) — see §4 below for why this is a package, not a folder
│   │   ├── package.json                  # + its dependency list is the enforcement surface — see §4
│   │   ├── src/
│   │   │   ├── notes/                    # + MIDI-number note ID, name derivation (§13)
│   │   │   ├── chords/                   # + pitch-class set matching, ChordDefinition table (§13, TYPES_AND_CONTRACTS §4.12)
│   │   │   ├── scales/                   # + ordered/unordered scale comparison, ScaleDefinition table (§13, TYPES_AND_CONTRACTS §4.13)
│   │   │   └── scoring/                  # + the composite score formula (§13, TYPES_AND_CONTRACTS §4.14)
│   │   └── src/__tests__/                # the highest-value test suite in the repo
│   └── shared-types/                     # types shared client ⇄ Functions (NoteEvent, Lesson, etc. — all 15 contracts in TYPES_AND_CONTRACTS.md)
│       ├── package.json                  # + zero runtime dependencies — types only, see §4.3 below
│       └── src/
├── functions/                            # Cloud Functions (2nd gen)
│   └── src/
│       ├── ai/                           # getTutorResponse, getExerciseFeedback's Gemini call, prompt templates (§15)
│       ├── scoring/                      # server-side validation of submitted note events against lesson defs, using packages/music-engine
│       ├── account/                      # deleteAccount, exportUserData (§18)
│       ├── lib/                          # + shared helpers: auth-context check, rate-limit counter (§19) — used by every callable, not duplicated per-function
│       └── index.ts                      # + exports the callable functions Firebase deploys
├── content/
│   └── lessons/                          # static seed JSON for lesson content (§17), loaded by a seed script
├── scripts/
│   └── seedLessons.ts                    # + loads content/lessons/*.json into Firestore; also where the ScoringWeights-sums-to-1.0 content validation (TYPES_AND_CONTRACTS §4.14) runs, at seed time not runtime
├── docs/                                 # this documentation set
├── firestore.rules
├── firestore.indexes.json
├── firebase.json
├── NOTICE                                # + Salamander Grand Piano CC BY 3.0 attribution — legally required, §12
├── package.json                          # + workspace root
├── tsconfig.base.json                    # + strict mode, extended by every package (docs/ENGINEERING/CODING_STANDARDS.md)
└── .gitignore                            # + .env.local excluded — no secret ever committed (§19)
```

**Notes on the additions marked `# +`:**

- **`apps/web/public/audio/samples/`** gives the §1 pushback ("Bundle piano samples as static assets served by the same CDN as the app") an actual home. Next.js's own `public/` directory, on Firebase App Hosting, is exactly that CDN-served static path — no separate asset pipeline needed.
- **`apps/web/lib/functions/`** is the client-side home for the typed callable-function wrappers `docs/ARCHITECTURE/APPLICATION_ARCHITECTURE.md` §6 describes: "thin typed wrappers around Firebase's callable-function SDK, one per function... each returning the `{ ok, data | error }` shape untouched so every call site handles errors the same way." `docs/ENGINEERING/ERROR_HANDLING.md` specifies exactly what "untouched" requires this layer to do even when the underlying SDK call itself rejects.
- **`functions/src/lib/`** exists because §19's auth-context check ("every callable Function checks `context.auth` before doing anything user-specific") and rate-limit counter apply identically to every callable — writing that once and importing it, instead of copy-pasting it into `ai/`, `scoring/`, and `account/` separately, is the difference between one place to fix a bug in that logic and four.
- **`scripts/seedLessons.ts`** is the script §17 and `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 both reference ("seeded via a script from static JSON," "loaded into Firestore by a seed script") but PRODUCT_SPEC.md §27's tree never gives it a place in the folder structure. It's also the natural place for the content-level validation `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14 calls for — "a Lesson/Exercise seed file that fails this check should fail content validation at build/seed time, not at runtime."
- **`NOTICE`, `.gitignore`, `tsconfig.base.json`** operationalize three explicit spec requirements: the Salamander attribution file (§12), never committing `.env.local` (§19), and strict-mode TypeScript applied uniformly (`docs/ENGINEERING/CODING_STANDARDS.md`).

⚠ **Open question (NON-BLOCKING):** PRODUCT_SPEC.md never names a monorepo build-orchestration tool. §27's `apps/`/`packages/` layout strongly implies a workspace-style monorepo, but doesn't say whether that's plain npm/pnpm/yarn workspaces or a build tool like Turborepo or Nx layered on top. **Recommendation, not extracted from the spec:** plain package-manager-native workspaces (no extra build-orchestration dependency) for MVP, consistent with the "don't add complexity without a reason" principle already applied elsewhere in the stack (§20, §32, `docs/ENGINEERING/DEPENDENCY_POLICY.md`). Revisit only if cross-package build/caching time becomes a real, measured problem — not preemptively.

## 4. Why `packages/music-engine` Is Its Own Workspace Package

This is the one structural decision in this repo that isn't just organizational taste — it's what makes a specific claim in PRODUCT_SPEC.md true by construction rather than by discipline. §27 states the claim directly: `packages/music-engine` as its own workspace package "is what makes 'independent from the UI, testable independently' structurally true instead of a stated intention. It physically cannot import React or Firebase without that showing up as a visible, reviewable dependency violation." Here is exactly why that's true, mechanism by mechanism:

**1. A workspace package's `package.json` is the enforcement surface.** Every workspace package declares its own `dependencies`. If a contributor writes `import { useState } from 'react'` inside `packages/music-engine/src`, that import fails to resolve at build time *unless* `react` is first added to `packages/music-engine/package.json` — and adding it is a one-line diff on a file whose entire purpose is "here is everything this package depends on." A reviewer sees `+ "react": "^..."` appear in a package.json that's supposed to have none of that, on a PR that's supposed to be about scoring logic, and the violation is immediately visible.

   Compare the counterfactual: if `music-engine` were `apps/web/lib/music-engine/` instead, it would transitively inherit *everything* already resolvable inside `apps/web` — React, Next, the Firebase client SDK, whatever Gemini SDK the client uses, all already sitting in `apps/web`'s `node_modules`. The same `import { useState } from 'react'` would resolve immediately, silently, with no diff anywhere flagging that anything changed. The violation would be just as real and completely invisible.

**2. It's the literal precondition for the rule stated in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5, Cross-Cutting Rule 2**: "`packages/music-engine`... must have **zero import** of `firebase/*`, `react`, `next`, or any Gemini/AI SDK. It is a pure function of `(NoteEvent[], Lesson | Exercise) → ExerciseVerdict`." A rule stated in a markdown file is a promise about intent. `docs/ENGINEERING/CODING_STANDARDS.md` specifies exactly how that promise gets checked mechanically (an import-restriction lint rule and/or a CI grep check) — but neither of those tools has a package boundary to check *against* unless the boundary exists first. The folder structure is the precondition the enforcement mechanism depends on, not decoration on top of an already-true fact.

**3. It's what makes "independent testability" (§11, §13) literally true, not aspirational.** `packages/music-engine`'s test suite runs with a plain test command inside that package alone — a `NoteEvent[]` fixture in, a verdict object out (§13's own worked example: "expected C-D-E-F-G, played C-D-E-G" asserts `{ missedNotes: ['F'], accuracy: 0.8 }` directly) — with no DOM, no React Testing Library, no Firebase emulator, and no Next.js dev server anywhere in the loop. If the same logic lived inside `apps/web`, running its tests would require standing up whatever the whole Next.js app's test environment needs, even though the logic under test needs none of it. This is the concrete reason `docs/ENGINEERING/TECH_STACK.md` can credibly target near-100% coverage on this package specifically — a fast, dependency-free suite is one people actually run on every save.

**4. It's what lets the exact same engine code run on both sides of the client/server trust boundary.** `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.4 requires that a submitted `ExerciseAttempt` be treated as untrusted, and that the Cloud Function "computes the actual `ExerciseVerdict` itself by re-running `packages/music-engine` server-side against the `Lesson`/`Exercise` definition it independently loads." `functions/` is a separate deployable from `apps/web` — it can cleanly declare a workspace dependency on `packages/music-engine` (and `apps/web` does the same, in reverse) because `music-engine` is a normal sibling package, not something buried inside another app's internals. If the engine lived inside `apps/web`, reusing it from `functions/` would mean either duplicating the logic — exactly the kind of drift `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` exists to prevent — or an awkward cross-app reach-through no monorepo tool treats as a first-class, declarable dependency.

**Enforcement mechanism:** specified in `docs/ENGINEERING/CODING_STANDARDS.md` (an import-restriction lint rule, e.g. `eslint-plugin-boundaries`, run in CI — with a simpler CI grep check as an explicitly-acknowledged MVP stopgap). This document explains *why the boundary exists and must hold*; that one explains *how CI proves it still does*, on every PR, not just at setup time.

### 4.1 `packages/shared-types` — the sibling boundary, for a different reason

`packages/shared-types` is also its own package, but for a narrower reason: it's the single physical location where all fifteen contracts in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` are declared as actual `.ts` exports (§3 of that document), imported by both `apps/web` and `functions/` so the two never define a slightly-different `NoteEvent` or `Lesson` shape independently. Unlike `music-engine`, it carries no computation and — per `docs/ENGINEERING/CODING_STANDARDS.md` — should carry no runtime dependencies of its own beyond perhaps a validation library; it exists to be imported everywhere cheaply, not to enforce a framework-independence boundary the way `music-engine` does.

⚠ **Open question (IMPORTANT — flagged, not silently resolved):** `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` is not fully self-consistent about *where the type declarations for `NoteEvent`, `ChordDefinition`, and `ScaleDefinition` physically live*. §3 states plainly that "`packages/shared-types` is the package that owns these as actual `.ts` exports" — referring to all fifteen contracts listed in §2, including those three. But §5, Cross-Cutting Rule 2, separately states that "`packages/music-engine` (which **owns** `NoteEvent`, `ChordDefinition`, `ScaleDefinition`, and the verdict-computation logic behind `ExerciseVerdict`)" must have zero framework imports — using the word "owns" for the same three types.

This document's working interpretation, used consistently above and in `docs/ENGINEERING/CODING_STANDARDS.md`: the `.ts` interface declarations physically live in `packages/shared-types` (per §3's plain statement, which is the more specific and more direct claim), and `packages/music-engine` *imports* them from there and is the domain owner of the *logic that operates on them* (chord/scale interval tables, scoring functions) — i.e., §5's "owns" is read as "is authoritative for the behavior involving," not "is the file location of the interface." This reading lets `music-engine` depend on `shared-types` (a permitted, types-only workspace dependency, not a framework-import violation) without contradicting either statement. This is a reasonable and workable interpretation, but it is an interpretation, not an unambiguous reading of the source document — flagging it so `docs/ENGINEERING/TYPES_AND_CONTRACTS.md`'s author can confirm or correct it, rather than two contributors independently guessing differently about which package a new type-level change belongs in.

## 5. Interfaces / Contracts

- Every callable Cloud Function listed in `docs/PRODUCT_SPEC.md` §20 / `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` §6 gets exactly one wrapper module under `apps/web/lib/functions/` — e.g. `getExerciseFeedback.ts`, `getTutorResponse.ts`, `deleteAccount.ts`, `exportUserData.ts` — so there is one, and only one, place in the client that calls each Function.
- `apps/web/hooks/` names should describe *what data*, not *how it's fetched* (`useUserProfile`, not `useUserProfileSnapshot`) — the underlying mechanism (`onSnapshot`) is an implementation detail of the hook, not part of its contract with the component tree that calls it.

## 6. Dependencies

Structural, not runtime: `apps/web` depends on `packages/music-engine` and `packages/shared-types`; `functions/` depends on `packages/music-engine` and `packages/shared-types`; `packages/music-engine` depends on `packages/shared-types` only. No other cross-package dependency exists in this repo, and a new one should be treated as a design question, not a routine addition — see `docs/ENGINEERING/DEPENDENCY_POLICY.md`.

## 7. Out of Scope

- Exact npm package versions — `docs/ENGINEERING/TECH_STACK.md`.
- `packages/music-engine`'s internal module boundaries, the input-adapter contract, and the Phase-2 MIDI adapter design — `docs/ARCHITECTURE/MUSIC_ENGINE_ARCHITECTURE.md` (not yet written; this document covers why the package boundary exists, not what's organized inside it beyond the top-level `notes/`/`chords/`/`scales/`/`scoring/` split).
- Firestore Security Rules syntax — `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.
- Lint rule / CI check configuration for the import-boundary enforcement — `docs/ENGINEERING/CODING_STANDARDS.md`.

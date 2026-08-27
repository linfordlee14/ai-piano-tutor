# Coding Standards

## 1. Purpose

Engineering defaults for writing code against `docs/PRODUCT_SPEC.md` and `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` in this specific stack. Where the spec makes an explicit requirement, this document states it as a rule and cites the source. Where the spec is silent and a decision is still needed to write code at all, this document makes a reasonable default and **marks it clearly as a recommendation, not something extracted from PRODUCT_SPEC.md** — per this task's own instruction not to silently invent scope, and per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §6's own explicit delegation of one such decision (the validation library) to this document.

## 2. Scope

Applies to every workspace package: `apps/web`, `packages/music-engine`, `packages/shared-types`, `functions/`. Covers TypeScript configuration, the `packages/music-engine` import boundary and its enforcement, naming conventions, the Cloud Functions error-return contract, and commit/PR conventions.

Out of scope: lesson-content authoring standards (`docs/CONTENT/`), Firestore Security Rules syntax (`docs/SECURITY/FIRESTORE_SECURITY_RULES.md`), the error-code taxonomy itself and its UI mapping (`docs/ENGINEERING/ERROR_HANDLING.md` — this document only requires that Functions use it consistently), and prompt-engineering conventions for Gemini templates (`docs/AI/AI_PROMPTS.md`).

## 3. Requirements

### 3.1 TypeScript strict mode — required, no exceptions

`"strict": true` in a shared `tsconfig.base.json`, extended by every workspace package's own `tsconfig.json`. This includes `strictNullChecks`, which is not optional here specifically: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` repeatedly relies on the distinction between an optional field (`?`, may be absent) and a required field typed `T | null` (always present, explicitly nullable — e.g. `UserProfile.displayName: string | null`, `Lesson.prerequisiteLessonId: string | null`). Without strict null checks, that distinction is documentation only; with it, the compiler enforces it.

No implicit `any`. An explicit `any` requires an inline comment explaining why (e.g. `// eslint-disable-next-line @typescript-eslint/no-explicit-any -- <reason>`) — it should be rare enough in this codebase that every instance is worth a sentence.

**Recommendation, not extracted from the spec:** use [Zod](https://zod.dev) as the runtime validation library at every trust boundary — server-side validation of an incoming `ExerciseAttempt` or `TutorRequest`'s `learnerQuestion` (§19: "input validation/length caps on the learner's free-text question"), and Firestore/content-seed validation (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.14's `ScoringWeights`-sums-to-1.0 check). `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §6 explicitly leaves this choice to this document ("Exact validation library/schema-enforcement mechanism (Zod, io-ts, etc.) — a choice for `docs/ENGINEERING/CODING_STANDARDS.md` to make, not this document"). Zod is recommended over alternatives (io-ts, manual guards) for three concrete reasons specific to this stack: its inferred types (`z.infer<typeof Schema>`) avoid hand-maintaining a duplicate runtime schema next to each `packages/shared-types` interface; it runs identically in Next.js (`apps/web`) and Cloud Functions (`functions/`) with no environment-specific code; and its schema objects can back Gemini's structured-output/JSON-schema mode (§15, used for the Phase 2 Daily Practice Plan template) without a second schema-definition step.

### 3.2 Naming conventions

Type names in code **must match `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` exactly** — `NoteEvent`, `ExerciseVerdict`, `PracticeSession`, `AIMemorySummary`, and so on, character-for-character, including field names and casing. This isn't a style preference: `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §1 states it's "the single canonical source for every shared data shape," and a type that quietly renames a field while implementing it (`missedNotes` vs. `missed_notes`, `midiNote` vs. `note`) reintroduces exactly the drift that document exists to prevent. If an implementation needs a field the canonical doc doesn't have, that's a gap — raise it in `docs/OPEN_QUESTIONS.md`, don't invent a divergent shape locally (`docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §1's own rule, restated here because it's a coding standard as much as a documentation one).

Concretely:
- **Types/interfaces:** `PascalCase` (`NoteEvent`, `ScoringWeights`), matching `packages/shared-types` exactly.
- **String-literal unions over enums** for the small closed sets throughout the contracts (`'noteOn' | 'noteOff'`, `'mobile' | 'tablet' | 'desktop'`) — matching how every one of the fifteen contracts is already written, and preferred over a TypeScript `enum` because literal unions serialize to/from Firestore and JSON with no separate mapping step.
- **Fields:** `camelCase`, matching `packages/shared-types` exactly — no local renaming for "readability."
- **Cloud Function names:** must match `docs/PRODUCT_SPEC.md` §20 verbatim — `getTutorResponse`, `getExerciseFeedback`, `deleteAccount`, `exportUserData`, and (Phase 2) `generateDailyPlan`. Other documents in this set (`docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` §6) already reference these by these exact names; renaming one during implementation breaks those references silently.
- **Filenames — recommendation, not extracted from the spec:** `kebab-case` for non-component modules (`note-event.ts`, `scoring-weights.ts`), `PascalCase` for React components matching the component name (`PianoKey.tsx`, `LessonPlayer.tsx`) — a common, low-controversy default, stated explicitly so it's a decision rather than an accident.
- **Exports:** named exports only, no default exports — makes renaming-safe imports and keeps `packages/music-engine`'s public surface (what `functions/` and `apps/web` are actually allowed to import) explicit rather than implicit.

### 3.3 `packages/music-engine` import boundary — zero tolerance, automated

The rule, restated from `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 and explained structurally in `docs/ENGINEERING/PROJECT_STRUCTURE.md` §4: no file under `packages/music-engine/src` may import `react`, `next`, `firebase`, `firebase-admin`, any package under the `firebase/*` subpath family, or the Gemini/AI SDK package (name TBD at implementation time — see `docs/ENGINEERING/TECH_STACK.md`'s note on why it's not pinned here). Its only permitted internal workspace dependency is `packages/shared-types`.

This must be enforced by tooling, not convention — "zero tolerance" that only lives in this paragraph is not zero tolerance. Two layers, from more to less robust:

1. **Preferred: an import-restriction lint rule, run in CI.** [`eslint-plugin-boundaries`](https://github.com/javierbrea/eslint-plugin-boundaries) (or the lighter-weight `import/no-restricted-paths` from `eslint-plugin-import`) configured so files under `packages/music-engine/**` may only import from within the package itself and from `packages/shared-types`. This should be configured as an **allowlist** (only these imports are permitted; everything else denied by default), not a denylist (only these imports are forbidden) — an allowlist doesn't require anticipating every possible bad import in advance, which a denylist does by construction. The rule runs as part of `lint` in CI and fails the build on violation; a violation should never be discoverable only by an editor squiggly someone can ignore.
2. **Acceptable MVP-simple stopgap, if standing up the lint rule is more setup than the team wants before M1: a CI grep check.** A short script (e.g. `scripts/check-engine-boundary.sh`) that greps `packages/music-engine/src` for import statements matching `firebase`, `from 'react'`, `from 'next`, or the Gemini SDK package name, and exits non-zero on any match — wired into the same CI job that runs tests. This is explicitly a stopgap with an acknowledged weakness: it's a denylist, so it only catches what someone thought to grep for, and it does not stop a `dependencies` entry from being added to `packages/music-engine/package.json` in the first place. Graduate to the lint-rule version once there's time to configure it properly — don't treat the grep check as the permanent solution.

Underneath both: `packages/music-engine/package.json`'s own `dependencies` (and `peerDependencies`) list must never contain `react`, `next`, `firebase*`, or a Gemini SDK package. This is the most fundamental layer (`docs/ENGINEERING/PROJECT_STRUCTURE.md` §4's "enforcement surface") — without the dependency even being installable in that package's own `node_modules`, most import paths won't resolve at all in a correctly configured workspace. The lint rule / CI grep check is the fast-feedback layer that catches the mistake at PR time, before someone gets far enough to add the dependency and have it start working.

This check runs on **every PR** as a required CI job, not as a pre-commit hook alone — a pre-commit hook can be skipped locally (`--no-verify`); a required CI check gating merge cannot.

### 3.4 Cloud Functions — consistent `{ ok, data }` / `{ ok, error }` return contract

Every callable Cloud Function returns the shape `docs/PRODUCT_SPEC.md` §20 defines: `{ ok: true, data: T }` on success, `{ ok: false, error: { code, message } }` on any anticipated failure. `docs/ENGINEERING/ERROR_HANDLING.md` owns the full error-code taxonomy and the client-facing state each code maps to; this section states the coding-standard requirement that every Function actually produce this shape, consistently, rather than each Function inventing its own variant.

```typescript
// Canonical type — formalized in TYPES_AND_CONTRACTS.md §4.16 (D-04).
type FunctionResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: FunctionErrorCode; message: string } };
```

**Rule:** a Function handler catches its own anticipated failure modes (auth check fails, input fails validation, Gemini is unreachable, the rate limit is hit) and *returns* — never throws — the `{ ok: false, error }` shape for each. This matches `docs/ARCHITECTURE/SYSTEM_ARCHITECTURE.md` §10's own example ("a failed [auth] check returns the `{ ok: false, error }` shape... not a raw 401 the client has to interpret") and is the only way "every function returns a typed... shape" (§20's own wording — "returns," not "throws or returns") is actually true across every code path, not just the happy one. Full implementation guidance on the throw-vs-return distinction, and what a genuinely unrecoverable platform-level failure (network failure, a crash before any handler code runs) still means for the client: `docs/ENGINEERING/ERROR_HANDLING.md`.

~~Previously an open question~~ — `FunctionResult<T>` and `FunctionErrorCode` are now formalized as canonical types in `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.16, using `ERROR_HANDLING.md` §4.3's naming as the canonical source. See `docs/DECISIONS.md` D-04.

### 3.5 Commit and PR conventions

⚠ **Recommendation, not extracted from PRODUCT_SPEC.md** — the spec is silent on commit/PR format, so this is a reasonable default stated explicitly as a default, not a spec requirement:

- **Commits:** [Conventional Commits](https://www.conventionalcommits.org/) — `<type>(<scope>): <description>`. Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`, `perf`. Scopes match workspace packages: `web`, `music-engine`, `functions`, `shared-types`, `content`. Example: `feat(music-engine): add major/minor triad inversion detection`.
- **PRs:** scoped to one milestone sub-task where practical, per `docs/PRODUCT_SPEC.md` §28's "build milestone by milestone, not the whole spec at once" instruction (§35 restates the same principle for the AI-assisted scaffolding phase specifically — "feed the relevant section(s)... for each milestone in order"). The PR description names which `docs/PRODUCT_SPEC.md` section(s) and/or milestone (`M0`–`M5`) it implements. CI (lint, typecheck, unit tests, and Firestore rules tests where relevant) must pass before merge — no exceptions for "it's just docs" or "it's just config," since a config change is exactly where a security-rule or dependency-boundary regression is most likely to slip in unreviewed.
- **New shared types:** any change to a type in `packages/shared-types` is added to `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` in the *same* PR, not as a follow-up — otherwise the canonical doc and the actual code drift from the moment the PR merges.

## 4. Interfaces / Contracts

This document does not redefine any data shape — `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` is canonical for all of them. What this document adds is *how* those shapes get validated at runtime (Zod, §3.1) and *how consistently* the one contract PRODUCT_SPEC.md defines at the API layer (`{ ok, data }` / `{ ok, error }`, §3.4) gets produced by every Function, every time.

## 5. Testing Requirements

- `packages/music-engine` tests live under `src/__tests__/` per `docs/ENGINEERING/PROJECT_STRUCTURE.md`, run with zero mocking (pure functions, plain data fixtures in and out per §13) — this is what makes the ≥90% coverage acceptance bar (`docs/PRODUCT_SPEC.md` §29) realistic rather than aspirational.
- The import-boundary check (§3.3) and a `tsc --noEmit --strict` pass across every workspace are both required, green CI checks — not advisory ones — before a PR can merge.
- Any new Cloud Function ships with at least one test asserting it returns the `{ ok, error }` shape (not a thrown exception the test framework has to catch) for at least one of its anticipated failure modes.

## 6. Acceptance Criteria

- [ ] `tsc --noEmit` with `strict: true` passes with zero errors across every workspace package.
- [ ] CI fails if any file under `packages/music-engine/src` imports `react`, `next`, `firebase*`, or the Gemini SDK package.
- [ ] CI fails if `packages/music-engine/package.json` lists `react`, `next`, `firebase*`, or a Gemini SDK package as a `dependency` or `peerDependency`.
- [ ] Every callable Cloud Function's return type is `FunctionResult<T>` (canonical type, `TYPES_AND_CONTRACTS.md` §4.16, D-04) — no handler returns a bare thrown exception to the client for an anticipated failure mode.
- [ ] Every exported type name in `apps/web`, `functions/`, and `packages/music-engine` that corresponds to a `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` contract matches that document's name and field names exactly.

## 7. Out of Scope

- The error-code taxonomy itself and its client-UI mapping — `docs/ENGINEERING/ERROR_HANDLING.md`.
- Firestore Security Rules syntax — `docs/SECURITY/FIRESTORE_SECURITY_RULES.md`.
- Performance budgets and how they're measured — `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md`, `docs/TESTING/PERFORMANCE_TESTING.md`.
- Dependency-addition policy beyond the music-engine-specific note in §3.3 — `docs/ENGINEERING/DEPENDENCY_POLICY.md`.
- Lesson-content JSON schema and its authoring conventions — `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`.

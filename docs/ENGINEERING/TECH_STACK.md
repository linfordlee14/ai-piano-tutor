# Tech Stack

## 1. Purpose

The canonical technology stack for AI Piano Tutor's MVP, and — just as importantly — the reasoning behind each choice. Extracted from `docs/PRODUCT_SPEC.md` §32 (Recommended Technology Stack), cross-checked against every section that actually justifies a row (§1, §12, §16, §20, §21, §26, §35).

The spec's own standing rule is that every dependency needs a reason (§1, §32: "the brief's own instruction... is that every dependency needs a reason, and none of these currently have one" — referring to the excluded set in §4 below). This document exists so that rule has one place to be checked against, instead of being re-litigated per PR.

**Division of labor:** this document answers *what do we build with, and why*. `docs/ENGINEERING/PROJECT_STRUCTURE.md` answers *where it lives*. `docs/ENGINEERING/DEPENDENCY_POLICY.md` is the standing policy that governs anything *added* to this list after MVP — this document is a snapshot the policy produced, not the policy itself.

## 2. Scope

Covers the MVP stack: frontend framework, styling, audio, client state, backend/AI proxy, database, auth, hosting, AI provider, testing tooling, and scaffolding/dev tooling — i.e., every row PRODUCT_SPEC.md §32 lists.

Out of scope: exact npm package versions and lockfile contents (a `package.json` concern, not a spec-level decision); CI/CD platform selection (PRODUCT_SPEC.md never names one — see the open question in §4); Phase 2+ additions (MIDI input uses the browser-native Web MIDI API via `navigator.requestMIDIAccess()`, not a new npm dependency, so it doesn't change this document when it ships, §11).

## 3. Stack

| Layer | Choice | Why | Source |
|---|---|---|---|
| Frontend framework | Next.js (App Router) + TypeScript | Firebase App Hosting is Next.js-native — SSR/framework-aware builds without a hand-rolled Cloud Run deployment; also matches the author's default stack elsewhere | §32, §16 |
| Styling | Tailwind CSS | Matches the author's default stack; fast to build a dense, custom piano UI without fighting a component library's opinions | §32 |
| Audio | Tone.js + `Tone.Sampler`, loaded with a trimmed **Salamander Grand Piano** sample set (CC BY 3.0 — attribution to Alexander Holm is legally required, see §7 below) | Real recorded piano beats synthesized realism for a product whose entire promise is "this is a piano." Tone.js's Transport/scheduling layer is needed regardless of sample source — demo playback, a future metronome, and timing scoring all depend on it | §12, §32 |
| Client state — engine/UI | Zustand | Readable and writable from *outside* React's render cycle. The audio engine needs to read and update state from Tone.js event callbacks, which fire independently of React's scheduler — this is the one thing plain `useState`/Context can't do cleanly, and it's why Zustand was picked specifically rather than "a state library" generically | §21, §32 |
| Client state — server data | Firestore SDK's own realtime cache (`onSnapshot`), wrapped in small custom hooks | Firestore's SDK already caches and syncs in realtime. Layering React Query or SWR on top would duplicate work it already does, for no benefit | §21 |
| Backend / AI proxy | Cloud Functions, 2nd gen, TypeScript, **callable** functions specifically (not raw HTTPS functions) | The only component allowed to hold Gemini credentials or write scored/progress data. Callables were chosen over raw HTTPS specifically because they handle auth-context passing and CORS automatically — less boilerplate, fewer footguns | §14, §16, §19, §20 |
| Database | Firestore | Primary datastore: user profiles, progress, lesson content, practice sessions, AI memory doc | §16, §17 |
| Auth | Firebase Authentication | Anonymous try-first, Google Sign-In, email/password; provider list is additive by default (e.g. Apple Sign-In later, no data-model change) | §18 |
| Hosting | Firebase App Hosting | Next.js SSR-native deployment target | §16 |
| Abuse prevention | Firebase App Check | Enabled from day one on Firestore *and* the Gemini-calling Functions — cheap to add early, expensive to retrofit after abuse shows up | §16, §19 |
| AI | Gemini API, tiered by prompt template — cheapest tier for high-volume/low-creative-demand templates (exercise feedback, motivation), a mid tier reserved for open-ended tutor chat where response quality is load-bearing | Section 15 assigns the tier per template. See the model-version note below — this row is deliberately *not* pinned to a model ID | §14, §15, §32 |
| Testing — unit | Vitest or Jest | Pure-function testing for `packages/music-engine` — near-100% target coverage on chord/scale/scoring logic, the highest-value suite in the repo because everything else's trust in the AI-boundary argument depends on it being right | §13, §26, §32 |
| Testing — E2E/UI | Playwright | Piano key-press → sound trigger → visual-state update, the computer-keyboard mapping, and the anonymous-try-the-piano flow, end to end. Also the canonical E2E test (§28 M2): open lesson → identify note → press key → get result → progress saved | §26, §28, §32 |
| Testing — integration/security | Firebase Local Emulator Suite (Firestore + Functions + Auth) + `@firebase/rules-unit-testing` | Cloud Functions run against emulated Firestore/Auth without touching production data; explicit rules tests assert a client cannot self-write scores or lesson content | §26, §32 |
| Testing — accessibility | axe-core, automated in CI, plus one manual screen-reader pass before launch | Automated tools catch roughly half of real accessibility issues — §24's honest caveat about the piano-playing interaction itself specifically needs a human check | §24, §26 |
| Testing — performance | Lighthouse CI budget on the Lesson Player route, plus a manual key-press-to-sound latency check on a real mid-range Android device | Lighthouse doesn't surface that specific latency number. Full methodology: `docs/TESTING/PERFORMANCE_TESTING.md` (targets: `docs/ENGINEERING/PERFORMANCE_REQUIREMENTS.md`) | §26 |
| Scaffolding tooling | Google AI Studio Build or Antigravity for initial scaffolding + Firebase provisioning; **Claude Code** for `packages/music-engine`, security rules, the Gemini proxy, and the full test suite | Neither AI Studio nor Antigravity is built for hand-tuning Web Audio scheduling or reviewing Firestore rules line by line, and both are under a year old and evolving fast (Firebase Studio's own sunset — new workspaces disabled June 2026, full shutdown March 2027 — is direct evidence of how quickly this layer moves). Their output is scaffolding to review and own in a real repo, never the system of record | §1, §35 |

**Rows that are process, not shipped dependencies:** "Scaffolding tooling" and the CI-oriented testing tools don't ship inside the production bundle, so `docs/ENGINEERING/DEPENDENCY_POLICY.md`'s justification requirement applies to them more loosely than it does to a runtime npm package. They're listed here because §32 lists them in the same table, and this document's job is to reproduce that table faithfully — not because they carry the same bundle-size/security-surface cost a shipped dependency does.

**Model version — deliberately not pinned.** §14 is explicit that hardcoding a specific Gemini model version in a spec document goes stale fast: three tiers of "Flash" alone launched between early and mid-2026. The *tiering strategy* above (cheapest tier for high-volume/low-creative-demand calls, mid tier for open-ended chat) is the stable decision; the specific model ID behind each tier should be re-checked against [ai.google.dev's pricing page](https://ai.google.dev/gemini-api/docs/pricing) at implementation time, not copied from this document.

**Gemini SDK package name — also deliberately not asserted here.** For the same reason, this document does not name the specific npm package for the Gemini SDK (e.g. whichever of Google's client libraries is current at build time). `docs/ENGINEERING/CODING_STANDARDS.md`'s music-engine import-boundary rule is written to apply to "the Gemini SDK package" generically rather than to one hardcoded name, so the rule doesn't silently stop working if the package name changes before implementation.

## 4. Deliberately Excluded

Per §32: "Redux, GraphQL, a separate REST API layer, a third-party analytics SDK, Firebase Storage for public assets, and any CMS for MVP. Each would add a dependency without a corresponding need at this stage." The specific reasoning behind each:

| Excluded | Why not |
|---|---|
| **Redux** | Zustand already covers the one specific need this stack has for state outside React's render cycle (§21) — the audio engine reading/writing state from Tone.js callbacks. Redux's action/reducer/middleware boilerplate has no offsetting benefit at this scale, and running it alongside Zustand would mean two state paradigms solving the same problem. |
| **GraphQL** | No public or multi-client API exists or is planned — "No public REST API in MVP" (§20). The Next.js client talks to Firestore directly for reads and to typed Cloud Functions callables for anything needing server logic; both already fit the client's exact, fixed shape of needs. GraphQL earns its keep solving over-/under-fetching across heterogeneous clients and schemas — there is one client and a small, fixed set of reads. |
| **A separate REST API layer** | §20 chose Cloud Functions *callable* functions specifically because they handle auth-context passing and CORS automatically. A hand-rolled REST layer would reintroduce exactly the boilerplate and footguns that decision exists to avoid. Revisit only if a public API or third-party integration appears (§20 explicitly leaves this door open); none is planned. |
| **A third-party analytics SDK** | Firebase's built-in Google Analytics integration, with BigQuery export as the natural upgrade path, already covers the MVP-appropriate metric set (§25: DAU/WAU, lesson completion rate, session count/duration, funnel drop-off, most-missed notes/chords, tutor call volume). No case yet for a second, redundant analytics pipeline before there's a reason to need one — and a third-party SDK is also a second, less-protected copy of data that should stay scoped to Firestore under the user's own record (§25's privacy boundary). |
| **Firebase Storage for public assets** | Piano samples are public, immutable files. Storage's auth-rule and latency layer adds complexity for zero benefit on something that should just be cached at the edge — static samples are served by the same CDN as the app instead (§1 pushback, §16). Storage stays reserved for genuinely user-generated/private assets if they exist later (e.g. a future avatar upload); none are needed in MVP. |
| **Any CMS for MVP** | Lesson content is ~12–15 hand-authored static JSON files, seeded into Firestore by a script (§17, `docs/CONTENT/LESSON_CONTENT_SCHEMA.md`). A CMS is explicitly Phase 3 scope, "only worth it once lesson volume makes hand-editing JSON painful" (§8) — building one now solves a volume problem that doesn't exist yet. |

⚠ **Open question (NON-BLOCKING):** PRODUCT_SPEC.md never names a CI/CD provider. §28's M0 milestone requires "CI deploys on push," and §35 says "move the repo to GitHub immediately," from which GitHub Actions is a reasonable but *unstated* inference. This document deliberately does not assert a provider as spec fact. Pick one at implementation time and record the justification per `docs/ENGINEERING/DEPENDENCY_POLICY.md` rather than treating GitHub Actions as already decided.

## 5. Dependencies

The general policy governing anything added to this stack after MVP — what justification is required, and how the excluded list in §4 is itself an example of the policy in action rather than a special case — is `docs/ENGINEERING/DEPENDENCY_POLICY.md`. Read that document before adding a new package to any `package.json` in this repo.

One stack-specific note worth stating here rather than in the general policy: `packages/music-engine` carries a stricter bar than the rest of the repo. Per `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §5 and `docs/ENGINEERING/CODING_STANDARDS.md`, it must have zero dependency on `react`, `next`, `firebase*`, or any Gemini/AI SDK — full stop, not "avoid unless justified." Its only permitted internal workspace dependency is `packages/shared-types` (types only). Any *other* dependency proposed for that specific package — even a small, well-justified one — should be treated as a design smell first and a normal dependency-policy decision second, because the entire reason that package exists is to stay small, pure, and framework-independent (`docs/ENGINEERING/PROJECT_STRUCTURE.md` §4).

## 6. Out of Scope

- Exact npm package versions / lockfile contents.
- CI/CD platform selection (see the open question in §4).
- Phase 2+ stack additions. Song licensing, an admin CMS, and microphone-based pitch detection (Phase 2/3, §8) will each need their own stack additions when scoped — `docs/ENGINEERING/DEPENDENCY_POLICY.md` governs how those get justified and added, and this document gets updated at that point, not before.

## 7. Compliance note

**Salamander Grand Piano attribution is not optional.** The sample set is CC BY 3.0 — free for commercial use, but attribution to Alexander Holm is legally required (§12, verified against the [GitHub source](https://github.com/sfzinstruments/SalamanderGrandPiano)). This must ship as a visible credit (Settings/About screen) and a `NOTICE` file in the repo (see `docs/ENGINEERING/PROJECT_STRUCTURE.md` for where the file lives) before launch, not as a follow-up task — a five-minute fix if done now, an avoidable compliance gap if forgotten.

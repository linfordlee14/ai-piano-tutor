# AI Coding Agent Instructions

## 1. Purpose

This is the primary operating manual for any AI coding agent (Claude Code, Google AI Studio Build, Antigravity, or otherwise) implementing AI Piano Tutor from this documentation set. If you are an agent about to write code for this project, read this document in full before touching anything else. It governs how you work, not what you build — the *what* lives in `PRODUCT_SPEC.md` and everything derived from it.

## 2. Before you write a single line of code

In this exact order:

1. **Read `docs/PRODUCT_SPEC.md` in full.** Not the section that looks relevant — all of it. It is the ground-truth product specification and the source of every requirement, and details in one section routinely constrain what looks like an unrelated section.
2. **Read `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` in full.** Every shared data shape you will touch is defined here. Use these exact types. If you need a field this file doesn't have, that's a gap — raise it (see §6) rather than inventing a divergent shape.
3. **Read the architecture documents relevant to what you're building** (`docs/ARCHITECTURE/`) — at minimum `SYSTEM_ARCHITECTURE.md`, plus whichever of `MUSIC_ENGINE_ARCHITECTURE.md`, `AI_ARCHITECTURE.md`, `FIREBASE_ARCHITECTURE.md`, `SECURITY_ARCHITECTURE.md` applies to your task.
4. **Read the current milestone's spec in full** — `docs/ROADMAP/M0_FOUNDATIONS.md` through `M5_HARDENING.md`, whichever one you're implementing. It names exact scope, exact files, exact tests, exact acceptance criteria, and — just as important — what's explicitly **out of scope** for this milestone. Read the out-of-scope section as carefully as the in-scope one.
5. **Inspect the actual repository state before assuming anything.** If code already exists (a prior milestone, a prior session's partial work), read it before writing more. Do not assume a clean slate just because this document describes one.
6. **Check `docs/OPEN_QUESTIONS.md`'s BLOCKING section** for anything that touches your milestone. A BLOCKING item with no ratified resolution means you cannot correctly implement the piece it names — see §6 below rather than guessing.

## 3. The fifteen rules

These are not suggestions ranked by importance — every one of them is load-bearing, and violating any single one is a defect even if the resulting code runs and passes a superficial check.

1. **Read `PRODUCT_SPEC.md`, the architecture docs, and the current milestone spec before writing code** — every time, not just once at the start of a long session. Context resets; the requirement to actually read first does not.
2. **Inspect the existing repository before assuming its state.** Never write code as though starting from nothing when something is already there.
3. **Never rewrite a working system unnecessarily.** A milestone that touches one feature does not license rewriting adjacent, already-working code "while you're in there." Follow existing conventions; make focused changes.
4. **Implement one milestone at a time, in order** (`docs/ROADMAP/MILESTONES.md`), respecting each milestone's stated prerequisites. Do not start M3 work while M2's acceptance criteria are unmet. Do not implement a later milestone's feature early because it seemed convenient — see rule 11.
5. **Run the tests the current milestone specifies** (`docs/ROADMAP/M*.md` §8, cross-referenced against `docs/TESTING/`) before considering any piece of work done. A milestone is not complete because the code compiles.
6. **Never weaken a Firestore Security Rule, an App Check requirement, or any other security control to make a test pass or a feature easier to build.** If a rule is blocking something you're building, the rule is either correct (and your implementation needs to change) or genuinely wrong (in which case: update `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` deliberately, explain why in the same change, and add a `docs/DECISIONS.md` entry — never loosen it silently to unblock yourself.)
7. **Never expose a secret client-side.** No Gemini API key, no service-account credential, no `NEXT_PUBLIC_*` environment variable carrying anything sensitive. `docs/SECURITY/SECRETS_MANAGEMENT.md` and `docs/SECURITY/GEMINI_SECURITY.md` are the authority here. The CI bundle-grep check (`docs/TESTING/SECURITY_TESTING.md` §7) exists specifically to catch a violation of this rule — it must pass, not be worked around.
8. **Never move deterministic logic into Gemini, in either direction.** Not "just this once" for a feature that seems hard to score deterministically, and not by having Gemini "double check" a verdict `packages/music-engine` already computed. §3 of `README.md` states why. This is the single rule this entire documentation set exists to protect.
9. **Never add a dependency without a stated reason.** `docs/ENGINEERING/DEPENDENCY_POLICY.md`'s principle — "every addition needs a stated reason" — applies to npm packages, Firebase products, and third-party services alike.
10. **Never implement a Phase 2+ feature while working on an MVP milestone**, even if it seems like a small addition or you have the context loaded to do it well. `docs/PRODUCT/FUTURE_ROADMAP.md` names what's deferred; `docs/PRODUCT/MVP_SCOPE.md` names what isn't. If a milestone spec's out-of-scope section names something, it stays out, no matter how easy it would be to include.
11. **Update the relevant documentation when implementation reveals an architecture document was wrong**, or when a genuine design decision gets made during implementation that isn't already recorded. Code silently diverging from the docs that describe it is exactly the failure mode Spec-Driven Development exists to prevent — see `docs/ROADMAP/DEVELOPMENT_METHODOLOGY.md`.
12. **Report a blocker; don't guess past it.** If you hit a genuine BLOCKING open question (`docs/OPEN_QUESTIONS.md`), a contradiction between two documents you can't resolve on your own authority, or a requirement that seems to conflict with something already built — stop and report it, with the specific documents/sections in tension, rather than picking an interpretation and proceeding silently. See §6 below for exactly how.
13. **Verify every acceptance criterion in the current milestone's spec before marking it complete** — actually check each checkbox against the running system or test suite, not from memory of having implemented "something like that."
14. **Never claim something was tested, deployed, or verified if it wasn't actually done.** If you ran out of time or context to verify a criterion, say so explicitly rather than reporting the milestone as done.
15. **Prefer the simplest architecture that reliably satisfies the current milestone's requirements.** Do not add speculative flexibility, configuration, or abstraction for a Phase 2 feature that isn't being built yet.

## 4. Milestone quick-reference

| Milestone | One-line objective | Full spec |
|---|---|---|
| M0 | Project foundations — repo, tooling, CI, empty deployable shell | `docs/ROADMAP/M0_FOUNDATIONS.md` |
| M1 | The piano itself: input adapters, audio, the deterministic music engine, playable UI | `docs/ROADMAP/M1_MUSIC_ENGINE.md` |
| M2 | One real, server-scored lesson end to end | `docs/ROADMAP/M2_FIRST_LESSON.md` |
| M3 | Full MVP curriculum + drills, progress, XP/streak | `docs/ROADMAP/M3_MVP_CURRICULUM.md` |
| M4 | The Gemini-powered AI tutor layer, strictly on top of M1–M3 | `docs/ROADMAP/M4_GEMINI_TUTOR.md` |
| M5 | Hardening — security, accessibility, performance, account deletion/export, launch readiness | `docs/ROADMAP/M5_HARDENING.md` |

M3 and M4 both depend only on M2, not on each other — they can proceed in parallel or in either order (`docs/ROADMAP/MILESTONES.md` §4). Every other milestone is strictly sequential.

## 5. Where the exact answer lives

| You need... | Go to |
|---|---|
| The exact shape/type of something | `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` |
| Whether Gemini is allowed to touch something | §3 of `README.md`, `docs/ARCHITECTURE/AI_ARCHITECTURE.md` §Boundary |
| The actual `firestore.rules` | `docs/SECURITY/FIRESTORE_SECURITY_RULES.md` |
| A prompt template's literal text | `docs/AI/AI_PROMPTS.md` |
| The scoring algorithm, with worked examples | `docs/MUSIC/SCORING_SYSTEM.md` |
| A screen's full UX spec | `docs/UX/SCREEN_SPECIFICATIONS.md` |
| What's actually in the MVP | `docs/PRODUCT/MVP_SCOPE.md` |
| What test verifies a given requirement | `docs/REQUIREMENTS_TRACEABILITY.md` |
| Whether something is already decided or still open | `docs/DECISIONS.md`, `docs/OPEN_QUESTIONS.md` |

## 6. What to do when you hit a genuine blocker

A genuine blocker is: a BLOCKING item in `docs/OPEN_QUESTIONS.md` that your current task actually depends on, a contradiction between two authoritative documents you cannot resolve using the source-of-truth hierarchy in `README.md` §2, or a requirement with no traceable specification anywhere in `docs/`.

Do **not**: pick a plausible-sounding resolution and proceed silently; weaken a test, a security rule, or an acceptance criterion to route around the blocker; or implement a different, easier feature instead without saying so.

Do: stop, state exactly what's blocking you and which documents/sections are in tension or silent, and — if you're operating with a human in the loop — surface it for a decision. If you're able to make forward progress on unblocked work while a blocker is pending resolution, do that instead of idling, but say explicitly what you deferred and why.

## 7. A worked example of following this correctly

Suppose you're implementing M2 and reach the point of computing `Lesson.passThreshold`. `docs/ENGINEERING/TYPES_AND_CONTRACTS.md` §4.2 tells you the field exists and its type; `docs/OPEN_QUESTIONS.md` OQ-01 tells you no default is ratified; `docs/DECISIONS.md` D-05 tells you 0.8 is proposed, not ratified. The correct action is to implement against 0.8 (it's the documented, reasoned proposal, not an invented one), while noting in your milestone completion report that M2's acceptance criteria pass using the *proposed* 0.8 threshold pending Linford's ratification — not to treat 0.8 as silently settled, and not to block M2 entirely waiting for a sign-off that a non-blocking default already lets you proceed past.

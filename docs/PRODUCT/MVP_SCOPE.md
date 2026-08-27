# MVP Scope

## 1. Purpose

This is the **authoritative MVP boundary document**. Restates `docs/PRODUCT_SPEC.md` §7 (MVP Scope) and §34 (Final MVP Definition) in full, unmodified in substance. When any other document — including `docs/PRODUCT/PRODUCT_REQUIREMENTS.md`, `docs/PRODUCT/USER_STORIES.md`, or `docs/UX/SCREEN_SPECIFICATIONS.md` — needs to answer "is X in scope for MVP," this is the document to check. If another document appears to disagree with this one, this document wins; file the discrepancy as a doc bug against the other document.

## 2. In Scope — MVP

| Area | In MVP |
|---|---|
| Auth | Firebase Auth: anonymous try-first session, Google Sign-In, email/password, account linking |
| Piano | Virtual piano: mouse/touch/keyboard input, ~2–3 octave visible range with pan, sustain, volume |
| Engine | Deterministic music engine: MIDI-number-based note ID, major/minor triad detection, major-scale validation, timing/accuracy scoring |
| Lessons | ~12–15 structured lessons covering keyboard orientation → note names → octaves → simple melodies → basic rhythm → intro to major triads (the original brief's Levels 1–6, roughly) |
| Drills | Note Identification game, one mode, adaptive difficulty |
| Scales | Guided scale lessons: C major plus 3–4 more common keys |
| Chords | Chord lessons: major triads, root position only |
| AI Tutor | Gemini tutor: free-form Q&A + structured exercise feedback (two prompt templates), rate-limited per user |
| Daily Practice | Deterministic "next lesson + review your last 3 misses" generator — **not** yet AI-personalized (see §5 pushback below) |
| Dashboard | Continue-lesson CTA, streak, XP, one weak-area callout |
| Progress | Firestore-backed progress tracking, resumable across devices |
| UI | Mobile-first responsive UI; baseline accessibility (keyboard nav, ARIA labels, non-color correctness indicators) |

## 3. Out of Scope — Deferred

Reproduced exactly as `PRODUCT_SPEC.md` §7's deferred table. An item appearing here means: **not built in MVP, under any circumstance, regardless of how small it looks in isolation.**

| Deferred to | Items |
|---|---|
| Phase 2 | MIDI input, minor/7th/diminished/augmented chords + inversions, more scales, song learning module, Gemini-personalized daily practice, achievements beyond streak/XP, PWA + offline lesson caching |
| Phase 3 | Admin CMS, microphone-based audio input, shareable progress reports, internal analytics dashboards |
| Phase 4 | Native mobile app, two-hand advanced lessons, sight-reading/sheet music, multiplayer, human-teacher integration, wearables |

Full detail, value, and complexity rationale for each of these lives in `docs/PRODUCT/FUTURE_ROADMAP.md` — this document only establishes the boundary, not the future plan.

## 4. In/Out at a Glance

| Feature | MVP? |
|---|---|
| Anonymous try-first session | Yes |
| Google Sign-In / email-password | Yes |
| Account linking (anonymous → real account) | Yes |
| Mouse / touch / computer-keyboard input | Yes |
| MIDI keyboard input | No — Phase 2 |
| Note/chord/scale detection (deterministic engine) | Yes |
| Timing/accuracy scoring | Yes |
| Major triad lessons, root position | Yes |
| Minor / 7th / diminished / augmented chord lessons, inversions | No — Phase 2 (note: engine-level *detection* of minor triads is required in MVP per §29/`TYPES_AND_CONTRACTS.md` §4.12 even though minor-triad *lesson content* is Phase 2 — see the open-questions note in `docs/PRODUCT/PRODUCT_REQUIREMENTS.md` §5.3) |
| C major + 3–4 common-key scale lessons | Yes |
| Full scale set | No — Phase 2 |
| Note Identification game (adaptive difficulty) | Yes |
| Song learning module | No — Phase 2 |
| Gemini free-form tutor Q&A | Yes |
| Gemini structured exercise feedback | Yes |
| Gemini-personalized daily practice plan | No — Phase 2 (deterministic rule ships in MVP instead) |
| Streak + XP tracking | Yes |
| Achievements/badges beyond streak/XP | No — Phase 2 |
| Leaderboards / multiplayer | No — Phase 4 |
| Firestore-backed cross-device progress sync | Yes |
| Data export / account deletion | Yes |
| Admin CMS for lesson content | No — Phase 3 |
| Microphone-based audio input (acoustic pitch detection) | No — Phase 3 |
| Shareable progress reports | No — Phase 3 |
| Full offline-first PWA / service-worker asset caching | No — Phase 2 (Firestore's built-in offline persistence for ordinary blips *is* in MVP — see §5.4 of `PRODUCT_SPEC.md` §23) |
| Native mobile app | No — Phase 4 |
| Sight-reading / sheet music | No — Phase 4 |
| Two-hand advanced technique | No — Phase 4 |

## 5. Pushback on Record: "AI-Personalized Daily Practice"

Reproduced from `PRODUCT_SPEC.md` §7, because it is the single most likely place someone re-litigates scope mid-build:

> This sounds like a v1 feature but it isn't one. Personalization needs enough historical performance data to be *accurate*; a brand-new user has none. Calling Gemini to generate a practice plan from zero signal is pure cost with no accuracy benefit over a simple rule ("next incomplete lesson + review your last 3 mistakes"). Ship the deterministic version first; graduate to Gemini-authored plans in Phase 2 once there's real usage data to personalize against.

## 6. The MVP Test

Reproduced verbatim from `PRODUCT_SPEC.md` §34 (Final MVP Definition). Any proposed feature — no matter how small — is checked against this single sentence. If it doesn't serve the sentence, it is Phase 2 or later, full stop:

> A first-time visitor can open the app with zero signup, play their first correctly-identified note within two minutes, progress through a structured beginner path that ends with a recognizable simple melody and an introduction to major triads, ask the AI tutor a real question and get a grounded answer, and have all of that survive a sign-up and a device switch — with every one of those steps still working if Gemini is unreachable, except the tutor question itself.

## 7. What This Test Implies, Made Explicit

Breaking the single sentence into its component obligations, since each clause maps to a concrete, testable system property:

| Clause | What it requires |
|---|---|
| "open the app with zero signup" | Anonymous try-first session (§2 above) |
| "play their first correctly-identified note within two minutes" | Fast piano load, local audio, local engine recognition — no network dependency in the critical path |
| "progress through a structured beginner path" | The ~12–15 lesson sequence, linear, prerequisite-gated |
| "ends with a recognizable simple melody and an introduction to major triads" | Lesson content actually reaches melody + major-triad content, not just note ID — this bounds how *shallow* the MVP curriculum is allowed to be, as much as it bounds scope creep upward |
| "ask the AI tutor a real question and get a grounded answer" | Tutor Q&A grounded in actual recent performance (`TutorRequest.recentExerciseContext`), not a generic chatbot |
| "survive a sign-up" | Anonymous-to-linked upgrade preserves progress |
| "survive... a device switch" | Firestore-backed cross-device sync |
| "every one of those steps still working if Gemini is unreachable, except the tutor question itself" | The non-negotiable engine/AI boundary (§14) — this is the clause that makes the whole sentence buildable as a testable acceptance criterion rather than a vibe |

## 8. Cross-References

- `docs/PRODUCT/FUTURE_ROADMAP.md` — what happens to deferred items, and why, phase by phase.
- `docs/PRODUCT/PRODUCT_REQUIREMENTS.md` — every deferred item also has a stable requirement ID here, tagged `[Future — Phase N]`, so it can still be traced even though it isn't built.
- `docs/PRODUCT_SPEC.md` §7, §34 — original source; this document is a restatement, not a reinterpretation.

# Philosophy — SOLID / KISS / DRY as RN conventions

Principles are only useful when they map to a decision you make in this codebase. Each one below is a rule you can apply while typing, not a slogan.

## Single Responsibility (SRP)

- One file, one job. A `model/` file computes; an `api/` file talks to the network; a `ui/` file renders. If a file does two of those, split it.
- A screen's single responsibility is **composition**: read from a hook, render, wire callbacks. The moment a screen formats a date, maps a status to a color, or decides a redirect, that logic belongs in `model/`.
- A Zustand store's responsibility is *client* state (who is logged in, which theme). Server data is not its job — that's TanStack Query.

**Test:** can you describe the file in one sentence without "and"? If not, split it.

## Open/Closed & Dependency Inversion (the injectable seam)

The `apiClient` is built from injected dependencies (`transport`, `refresh`, `getAccess`, …) rather than importing axios and expo-secure-store directly. This is DIP in practice:

- Core logic depends on interfaces, not concretions. The 401-refresh-retry logic is pure and testable with fake deps — no network, no native modules, no mocking framework gymnastics.
- Adding a new behavior (e.g. request logging) means composing a new dep, not editing the core. Open for extension, closed for modification.

Apply the same seam anywhere logic meets I/O: pass the effectful thing in, keep the decision pure.

## KISS

- Thin routes. A screen that's more than ~120 lines is usually holding logic that belongs in a hook or a `model/`.
- Prefer a pure function to a hook, a hook to a context, a context to a global. Reach for the simplest thing that works.
- Don't add a state manager, a form library, or an abstraction until a second real caller exists.
- The idempotency-key generator is pure JS (`Math.random`) instead of a native crypto module — because "unique enough" is the actual requirement, and avoiding a native dep keeps the build simpler. Solve the real problem, not the impressive one.

## DRY (via contract files, not premature abstraction)

DRY here means **one source of truth for shared literals**, not "extract every two-line similarity."

- URLs, feature strings → `src/lib/constants.ts`.
- Cache keys → `src/lib/queryKeys.ts` (a structured factory, never inline `['user', id]` at call sites).
- Colors, radii, spacing → theme tokens (`global.css` CSS vars mirrored in `src/theme/`).
- The backend contract → generated types (`openapi-typescript`), never hand-retyped.

Duplicated *logic* is a smell; duplicated *literals* are a bug waiting to drift. Fix the second aggressively; fix the first only when the duplication is real (rule of three).

## YAGNI

- Build the feature slice you need now. The architecture makes adding the next slice cheap, so you lose nothing by waiting.
- No speculative screens, no "we might need dark mode config per-user," no abstraction with one implementation.
- Delete dead code and unused exports as you go; a strict `tsc` and unused-export discipline keep the surface honest.

## The through-line

Every rule above serves one goal: **most of your code should be pure, small, and testable, with I/O pushed to the edges.** That's why this codebase reaches high test coverage without an E2E harness — the logic that matters doesn't need a device to run. When a change feels hard to test, that's the signal a responsibility is in the wrong place; move it, don't mock around it.

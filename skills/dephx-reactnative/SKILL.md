---
name: dephx-reactnative
description: Use when scaffolding a new React Native / Expo mobile app, or adding a screen or feature to one — starting an Expo project, choosing the stack, structuring features, wiring navigation, data-fetching, state, storage, theming or tests, or hitting Expo build/config trouble (New Architecture, NativeWind, Reanimated worklets, CNG/prebuild, peer-dep errors). Applies to any app domain, not a specific one.
---

# dephx-reactnative

## Overview

A production-grade foundation for Expo / React Native apps, distilled from a shipped app. It gives you an opinionated stack, a feature-sliced architecture, the exact (fragile) config wiring, and the gotchas that otherwise cost hours — so a new app starts correct and stays testable.

**Core idea:** screens stay thin; logic is pure and colocated with its tests; all I/O goes through small, injectable seams. SOLID / KISS / DRY expressed as concrete RN conventions, not slogans. See `references/philosophy.md`.

This is **domain-agnostic**. Auth/session appears as the example most apps need, never as a required feature. Swap the example domain for yours.

## When to use

- Starting a new Expo app and you want a structure that scales past the first few screens.
- Adding a feature to an app already built on this foundation (follow the same slice shape).
- Deciding the stack (navigation, data, state, storage, styling, tests) instead of re-litigating each choice.
- Debugging Expo-specific build/config failures (New Architecture crash on boot, styles not applying, "worklet" errors, `ERESOLVE` on install, edits to `android/ios` vanishing).

**Not for:** store submission / signing / release (out of scope here), Expo web-first apps, or bare (non-Expo) RN without CNG.

## The stack (baseline: Expo SDK 57, RN 0.86, React 19.2, New Architecture ON)

| Concern | Choice |
|---|---|
| Runtime / routing | Expo + **expo-router** (file-based) |
| Language | TypeScript, **strict** |
| Server state | **TanStack Query v5** (+ persist to MMKV) |
| Client state | **Zustand** (session, theme — small slices only) |
| HTTP | Axios behind an **injectable `apiClient`** with single-flight 401 refresh |
| Secure storage | **expo-secure-store** (tokens, PIN) |
| Fast KV / cache | **react-native-mmkv** (query persist, prefs) |
| Styling | **NativeWind v4** (Tailwind) + semantic CSS-var tokens; Gluestack primitives optional |
| Precise numbers | **decimal.js** (money, quantities) |
| Tests | **jest-expo** + Testing Library, colocated `*.test.ts` |

Pin these versions when you scaffold; they interlock (see `references/config-reference.md`).

## Workflow: scaffold a new app

1. `npx create-expo-app@latest <name> --template default` (or blank-typescript), then `cd <name>`.
2. Add the stack deps and apply the **config quartet** exactly — babel, metro, tailwind + `global.css`, `app.config.ts`, plus `.npmrc`, `tsconfig`. → `references/config-reference.md`.
3. Lay down the folder structure and the `src/lib` spine (apiClient, secureStore, queryClient, queryKeys). → `references/architecture.md`, `references/api-layer.md`, `references/state-storage.md`.
4. Wire the root layout providers (Query + persist, SafeArea, theme, auth-failure → login). → `references/architecture.md`.
5. Set up theming tokens. → `references/design-system.md`.
6. Build the first feature as a slice: `src/features/<domain>/{api,model,ui}` with colocated tests. → `references/architecture.md`, `references/testing.md`.
7. Run on a **development build** (`npx expo run:android` / `run:ios`), not Expo Go — native modules require it.

Before writing any feature, read `references/gotchas.md` once. It prevents the common time-sinks.

## Guardrails (always / never)

- **NEVER** put business or formatting logic in a screen. Screens call a hook and render. Logic → `model/` (pure, tested).
- **NEVER** hardcode a URL, route string, color, or magic number in a component. Central `constants.ts`, `queryKeys.ts`, theme tokens.
- **NEVER** hand-edit `android/` or `ios/` — they are generated (CNG). Change `app.config.ts` or write a config plugin. → `references/gotchas.md`.
- **NEVER** add a native dependency without confirming New-Architecture support, then rebuild the dev client (not just reload).
- **NEVER** leave a trailing slash on an API route string.
- **ALWAYS** keep server state in TanStack Query and only session/UI state in Zustand — don't duplicate server data into a store.
- **ALWAYS** access the network through `apiClient` (injectable deps) so logic stays testable without mocking axios/expo.
- **ALWAYS** colocate a `*.test.ts` with pure logic; keep `tsc --noEmit` and `jest` green before moving on.

## References

| Read | For |
|---|---|
| `references/philosophy.md` | SOLID / KISS / DRY / YAGNI as concrete RN conventions |
| `references/architecture.md` | Folder structure, feature-slice pattern, root layout wiring |
| `references/config-reference.md` | The exact config quartet + `.npmrc`/`tsconfig` + version matrix |
| `references/api-layer.md` | `apiClient`, query hooks, mutations, queryKeys, OpenAPI type-gen |
| `references/state-storage.md` | Zustand session, secure-store, MMKV, query persist, money |
| `references/design-system.md` | Semantic tokens, light/dark, component conventions |
| `references/testing.md` | jest-expo setup, mocks, colocation, what to test |
| `references/gotchas.md` | Mistakes-learned → DO-NOT / ALWAYS checklist |

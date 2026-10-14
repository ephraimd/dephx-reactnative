---
name: dephx-reactnative
description: Use when working on a React Native / Expo mobile app, including its native screens, navigation, data, storage, UX, performance or build configuration. Not for responsive websites, general frontend/backend work or a mobile-width browser review.
---

# dephx-reactnative

Keep routes thin, business decisions testable and I/O behind small injectable seams.
Reuse the consuming app's architecture and component system before adding another.
This skill supplies portable engineering guidance, not a product's brand or policy.

Load only for a React Native / Expo app task. A website viewed on a phone is not a
native mobile app. Installation exposes the skill for discovery; it adds no global
startup instructions. In a monorepo, scope guidance to the mobile package, not siblings.

## Bind to the app before work

Read repository instructions, canonical brand/design documents, approved copy, shared
components and the real backend contracts. Name their current versions and the rules
applied. Reload after compaction or a checkout change. Project rules and explicit human
decisions override examples here. Discover existing review/CI gates; extend them within
authorized scope rather than inventing a second enforcement system.

## Foundation, not a mandatory shopping list

| Concern | Starting point |
|---|---|
| Runtime/navigation | Expo and Expo Router, existing native navigation patterns |
| Language | Strict TypeScript |
| Server/local state | TanStack Query / small Zustand slices; do not mirror server records |
| HTTP/auth | Existing injectable client; refresh only if the backend supports it |
| Credentials | expo-secure-store, following actual session contracts |
| UI | One approved component library, themed shared primitives and compositions |
| Styling | NativeWind when selected by the app; match its major and the UI kit |
| Persistence | Existing storage first; optional, scoped, allowlisted query persistence |
| Tests | SDK-compatible jest-expo and Testing Library plus native runtime checks |

Gluestack is a candidate for a new app, not permission to replace an existing kit.
Add only needed components. Alpha means prerelease maturity, not price; verify the
selected component's status, license and compatibility. Versions and examples in
`references/config-reference.md` are version-specific, not a verified current bundle.
MMKV, SQLite, decimal.js and refresh-token support are additions only when justified.

## Workflow

1. Inspect project guidance, code, lockfile, SDK, native-folder ownership and API contracts.
2. Keep the project's package manager. Resolve the Expo/RN/UI-kit matrix before installing.
3. Reuse feature `api/model/ui` slices, shared client, query keys and theme. See
   `references/architecture.md` and `references/api-layer.md`.
4. Compose through the chosen UI foundation. Read `references/design-system.md` for
   bounded lists, keyboard-safe dialogs, draft protection and discoverable tasks.
5. Choose persistence by data sensitivity and lifecycle. Read `references/state-storage.md`.
6. Run focused logic/component checks and the required native/device evidence in
   `references/testing.md`. A build or simulator screenshot is not a performance verdict.

For build trouble read `references/gotchas.md`; for architecture decisions read
`references/philosophy.md`. Do not add store submission, signing or production release
work merely because this skill loaded.

## Invariants

- No parallel component families, inline replicas of controls or copied brand doctrine.
- Failed reads remain errors; zero is valid data. Preserve records and dirty drafts.
- Submitted snapshots and subsequent edits stay distinct through pending saves.
- Credentials never enter logs, query persistence or deep-link URLs. Private records
  still require protection when credentials are elsewhere.
- No automatic money/admission replay after uncertain outcomes. Idempotency requires
  server support, stable attempt identity and reconciliation, not just a request header.
- Native folders are regenerable only after ownership is verified. Preserve manual work.
- Report observed evidence and untested cases separately; never invent a device pass.

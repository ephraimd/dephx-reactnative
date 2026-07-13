# Architecture — structure, feature slices, root wiring

## Folder structure

```
app/                      expo-router routes. Folder = URL segment. Screens only.
  _layout.tsx             root providers (Query+persist, SafeArea, theme, auth wiring)
  index.tsx               boot/redirect entry
  (auth)/                 route group: login, signup, verify, set-pin …
  (tabs)/                 route group: the tab bar + tab screens
  <feature>/              stack screens for a feature (e.g. settings, profile)
src/
  features/<domain>/      the real code, grouped by domain
    api/                  one file per endpoint: a TanStack Query/mutation hook + .test.ts
    model/                pure logic: formatting, derivations, state machines + .test.ts
    ui/                   components specific to this feature
  lib/                    cross-cutting spine (see below)
    apiClient.ts          injectable HTTP client + single-flight 401 refresh
    secureStore.ts        token read/write/clear (expo-secure-store)
    queryClient.ts        QueryClient + MMKV persister + persistOptions
    queryKeys.ts          structured cache-key factory (one source of truth)
    constants.ts          shared literals (URLs, feature strings)
    money.ts              decimal.js helpers (only if you handle precise numbers)
    idempotency.ts        de-dupe key for money/mutation retries (only if needed)
    api/types.ts          GENERATED from the backend OpenAPI spec — never hand-edit
  theme/                  tokens + useTheme (light/dark) + theme.ts (JS mirror of CSS vars)
  components/             YOUR shared, domain-agnostic components (ScreenHeader, AppInput …)
components/ui/            OPTIONAL: thin wrappers around a 3rd-party primitive lib
                         (e.g. Gluestack). Skip this folder if you use plain
                         react-native View/Text — then import those directly.
assets/                  icons, splash, adaptive icon
```

**Route groups** `(auth)` / `(tabs)` are expo-router parentheses folders: they group screens under a shared layout without adding a URL segment. Keep screens in `app/` as thin as possible — they import a hook from `src/features/*/api` and render.

## The feature slice — the unit you repeat

Every feature is a self-contained slice with three layers. A "notes" feature, de-branded, is the canonical shape:

```
src/features/notes/
  api/
    useNotes.ts          list query (or infinite query) + .test.ts
    useCreateNote.ts     mutation + .test.ts
  model/
    noteDisplay.ts       pure: derive title, badge tone, sign, labels + .test.ts
  ui/
    NoteRow.tsx          presentational row, given a display object
```

Why this shape:

- **`model/` is pure and does the thinking.** It has no imports from expo/react-native/network. It's trivially unit-tested (see `references/testing.md`). This is where most of your tested logic lives.
- **`api/` is the only place that knows the backend.** Each hook wraps one endpoint, unwraps the response envelope, and returns typed data. Route strings live here as named consts.
- **`ui/` is presentational.** Components receive already-derived display objects from `model/`, so they contain no business logic.

A screen in `app/` then reads `useNotes()`, maps rows through the `model/` deriver, and renders `<NoteRow>`. Nothing in the screen needs a test because it holds no logic.

### Example: pure model + its hook (de-branded from a real list feature)

`model/noteDisplay.ts` — pure, safe to call in render, fully unit-testable:

```ts
export interface NoteDisplay {
  title: string;
  tone: 'success' | 'danger' | 'muted';
}

const DANGER = new Set(['ARCHIVED', 'DELETED']);

function titleCase(v: string): string {
  return v.toLowerCase().split(/[_\s]+/).filter(Boolean)
    .map((w) => w[0].toUpperCase() + w.slice(1)).join(' ');
}

export function toNoteDisplay(note: { title?: string; status: string }): NoteDisplay {
  const title = note.title?.trim() || 'Untitled';
  if (note.status === 'ACTIVE') return { title, tone: 'success' };
  if (DANGER.has(note.status)) return { title, tone: 'danger' };
  return { title, tone: 'muted' };
}
```

`api/useNotes.ts` — the only network-aware file; note the named route const and envelope unwrap:

```ts
import { useQuery, type UseQueryResult } from '@tanstack/react-query';
import { apiClient } from '@/src/lib/apiClient';
import { queryKeys } from '@/src/lib/queryKeys';
import type { ApiEnvelope, Note } from '@/src/lib/api/types';

export const NOTES_ROUTE = '/notes'; // no trailing slash — see gotchas.md

export async function fetchNotes(): Promise<Note[]> {
  const res = await apiClient.request<ApiEnvelope<Note[]>>({ url: NOTES_ROUTE });
  const data = res.data.data;
  return Array.isArray(data) ? data : []; // degrade to empty, never crash the list
}

export function useNotes(): UseQueryResult<Note[]> {
  return useQuery({ queryKey: queryKeys.notes.all(), queryFn: fetchNotes });
}
```

For paginated lists use `useInfiniteQuery` with a `buildParams(page)` helper and a pure `getNextPageParam` (a short page = the end) — both easily tested. See `references/api-layer.md`.

## Root layout wiring (`app/_layout.tsx`)

The root layout is where cross-cutting providers and one-time wiring live, in this order:

1. `GestureHandlerRootView` (outermost).
2. `PersistQueryClientProvider` with the shared `queryClient` + `persistOptions` (MMKV) — restores cache on cold start.
3. Theme provider / primitive provider (e.g. Gluestack) fed the resolved light/dark scheme.
4. `SafeAreaProvider` + `StatusBar`.
5. The `Stack` navigator with `headerShown: false` (screens render their own `ScreenHeader`).

Two effects wired once on mount:

- Sync the persisted theme mode into NativeWind's `colorScheme` so `dark:` classes and CSS-var themes flip immediately.
- Register the auth-failure handler: when `apiClient`'s refresh fails, clear the session and `router.replace('/(auth)/login')`. This keeps the "kicked to login on unrecoverable 401" behavior in one place.

Full file:

```tsx
// app/_layout.tsx
import '../global.css';                       // MUST be imported once, here
import { useEffect } from 'react';
import { Stack, router } from 'expo-router';
import { StatusBar } from 'expo-status-bar';
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { SafeAreaProvider } from 'react-native-safe-area-context';
import { PersistQueryClientProvider } from '@tanstack/react-query-persist-client';
import { colorScheme } from 'nativewind';

import { queryClient, persistOptions } from '@/src/lib/queryClient';
import { setAuthFailureHandler } from '@/src/lib/apiClient';
import { useSession } from '@/src/features/auth/model/session';
import { useTheme } from '@/src/theme/useTheme';

export default function RootLayout() {
  const mode = useTheme((s) => s.mode);
  const scheme = useTheme((s) => s.resolvedScheme);

  // Apply the restored theme preference on cold start (and on change).
  useEffect(() => { colorScheme.set(mode); }, [mode]);

  // Route to login when the session is unrecoverable (refresh failed).
  useEffect(() => {
    setAuthFailureHandler(() => {
      useSession.getState().clear();
      router.replace('/(auth)/login');
    });
  }, []);

  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <PersistQueryClientProvider client={queryClient} persistOptions={persistOptions}>
        <SafeAreaProvider>
          <StatusBar style={scheme === 'dark' ? 'light' : 'dark'} />
          <Stack screenOptions={{ headerShown: false }} />
        </SafeAreaProvider>
      </PersistQueryClientProvider>
    </GestureHandlerRootView>
  );
}
```

If you adopt a UI-primitive library (e.g. Gluestack), add its provider just inside `PersistQueryClientProvider`, fed the resolved `scheme`.

## `src/lib/constants.ts` — one home for shared literals

Keep app-wide strings here so they don't drift across files (DRY). Small and boring by design:

```ts
// src/lib/constants.ts
/** Public web origin (deep links, share invites). */
export const WEB_URL = 'https://app.example.com';

/** Support contact surfaced in error/empty states. */
export const SUPPORT_EMAIL = 'support@example.com';
```

Route strings stay next to their hook as named consts (`NOTES_ROUTE`), and cache keys live in `queryKeys.ts` — `constants.ts` is for the cross-cutting rest.

## Where things go — quick decisions

| You're adding… | It goes in… |
|---|---|
| A screen / route | `app/<...>/screen.tsx` (thin) |
| A network call | `src/features/<domain>/api/useX.ts` |
| Formatting / derivation / a small state machine | `src/features/<domain>/model/x.ts` (+ test) |
| A component used by one feature | `src/features/<domain>/ui/` |
| A component used app-wide | `src/components/` |
| A shared URL / string | `src/lib/constants.ts` |
| A cache key | `src/lib/queryKeys.ts` |
| A color / radius / spacing | theme tokens |

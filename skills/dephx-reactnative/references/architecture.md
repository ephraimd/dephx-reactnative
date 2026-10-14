# Architecture - structure, feature slices, root wiring

## Folder structure

```
app/                      expo-router routes. Folder = URL segment. Screens only.
  _layout.tsx             root providers (Query with optional persistence, SafeArea, theme, auth wiring)
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
    apiClient.ts          injectable HTTP client + supported auth lifecycle
    secureStore.ts        token read/write/clear (expo-secure-store)
    queryClient.ts        QueryClient + optional scoped/allowlisted persistence
    queryKeys.ts          structured cache-key factory (one source of truth)
    constants.ts          shared literals (URLs, feature strings)
    money.ts              existing money contract helpers (decimal.js only if needed)
    idempotency.ts        de-dupe key for money/mutation retries (only if needed)
    api/types.ts          backend contract types (generate when maintained OpenAPI exists)
  theme/                  one canonical token source + useTheme (light/dark)
  components/             YOUR shared, domain-agnostic components (ScreenHeader, AppInput …)
components/ui/            chosen UI-library primitives and shared adaptations
                         (use the existing location in a consuming app)
assets/                  icons, splash, adaptive icon
```

**Route groups** `(auth)` / `(tabs)` are expo-router parentheses folders: they group screens under a shared layout without adding a URL segment. Keep screens in `app/` as thin as possible - they import a hook from `src/features/*/api` and render.

## The feature slice - the unit you repeat

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

- **`model/` is pure and does the thinking.** It has no imports from expo/react-native/network. It's trivially unit-tested (see `testing.md`). This is where most of your tested logic lives.
- **`api/` is the only place that knows the backend.** Each hook wraps one endpoint, unwraps the response envelope, and returns typed data. Route strings live here as named consts.
- **`ui/` is presentational.** Components receive already-derived display objects from `model/`, so they contain no business logic.

A screen in `app/` then reads `useNotes()`, maps rows through the `model/` deriver, and renders `<NoteRow>`. Thin composition still needs runtime checks for navigation, accessibility, keyboard and draft behavior.

### Example: pure model + its hook (de-branded from a real list feature)

`model/noteDisplay.ts` - pure, safe to call in render, fully unit-testable:

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

`api/useNotes.ts` - the only network-aware file; note the named route const and envelope unwrap:

```ts
import { useQuery, type UseQueryResult } from '@tanstack/react-query';
import { apiClient } from '@/src/lib/apiClient';
import { queryKeys } from '@/src/lib/queryKeys';
import type { ApiEnvelope, Note } from '@/src/lib/api/types';

export const NOTES_ROUTE = '/notes'; // no trailing slash - see gotchas.md

export async function fetchNotes(signal: AbortSignal): Promise<Note[]> {
  const res = await apiClient.request<ApiEnvelope<Note[]>>({ url: NOTES_ROUTE, signal });
  const data = res.data.data;
  if (!Array.isArray(data)) throw new Error('Invalid notes response');
  return data; // successful empty remains distinct from a malformed response
}

export function useNotes(accountId: string): UseQueryResult<Note[]> {
  return useQuery({
    queryKey: queryKeys.notes.all(accountId),
    queryFn: ({ signal }) => fetchNotes(signal),
    enabled: !!accountId,
  });
}
```

The shared client must accept AbortSignal and forward it to transport. This hook alone
is not a session fence: bind the client's credential capture and late-response handling
to the initiating account/session, including before cache writes. See `api-layer.md`.

For paginated lists use `useInfiniteQuery` with a `buildParams(page)` helper and a pure `getNextPageParam` following server pagination metadata - both easily tested. See `api-layer.md`.

## Root layout lifecycle

Reuse the existing provider composition: QueryClientProvider, the chosen UI/theme
provider, SafeAreaProvider and native navigation. Gesture roots and persistence providers
are included only when used. Do not hide native stack headers by default.

Restore identity before private cache hydration. If persistence is approved, use the
account-scoped allowlist and logout lifecycle in `state-storage.md`; a single global
persister for every account is unsafe. Wire AppState/focus and connectivity once, avoid
multiple listeners and preserve drafts during foreground refetches. Auth failure follows
the actual backend session contract and clears private state before routing to login.

## Deep links and authentication handoffs

Map supported web links and native routes from the start, using Expo Router and the
existing URL parser. Configure owned HTTPS domains for Android App Links and iOS
Universal Links when selected; a custom scheme alone does not capture ordinary HTTPS
links. Verify domain-association files, signed builds and cold/warm starts. Preserve a
validated intended destination through login; reject malformed or untrusted targets.

Future OAuth/connector support is not permission to build speculative integrations.
For an authorized OAuth flow, inspect the server contract, use the supported system
browser/AuthSession flow and enforce state, PKCE and allowlisted callback/redirect rules
appropriate to the protocol. Never put long-lived credentials in link URLs. Opening or
prefetching a link is read-only; attendance or payment requires its explicit action.

## Camera-driven verification

Reuse the installed scanner/camera primitive before adding custom frame processors.
Keep the camera ready between scans while focused, stop it on background/unmount and
handle permission denial/recovery. Guard repeated frame callbacks while an attempt is
pending; re-arm deliberately after showing the result without preventing intentional
rescans supported by the backend. Separate scan acknowledgement from confirmed success.

For server-confirmed admission, only a confirmed server result permits admission.
Timeout/connection loss leaves an unknown outcome: reconcile before another write, never
show green from cached eligibility. Do not add offline admission or a retry queue unless
explicitly authorized with server conflict rules. Profile decoding, feedback, request
latency and concurrent gate handling separately; unit tests cannot certify camera speed.

## `src/lib/constants.ts` - one home for shared literals

Keep app-wide strings here so they don't drift across files (DRY). Small and boring by design:

```ts
// src/lib/constants.ts
/** Public web origin (deep links, share invites). */
export const WEB_URL = 'https://app.example.com';

/** Support contact surfaced in error/empty states. */
export const SUPPORT_EMAIL = 'support@example.com';
```

Route strings stay next to their hook as named consts (`NOTES_ROUTE`), and cache keys live in `queryKeys.ts` - `constants.ts` is for the cross-cutting rest.

## Where things go - quick decisions

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

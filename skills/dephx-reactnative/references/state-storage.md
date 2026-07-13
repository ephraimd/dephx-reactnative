# State & storage — the right home for each kind of data

The single most clarifying rule in this stack:

> **Server state → TanStack Query. Client state → Zustand. Secrets → secure-store. Everything else fast/local → MMKV.**

Duplicating server data into a Zustand store is the most common architecture mistake. Don't. The query cache *is* your store for anything the server owns.

## Zustand — small client slices only

Hold identity and local UI state; nothing the server owns.

```ts
// src/features/auth/model/session.ts
import { create } from 'zustand';
import type { User } from '@/src/lib/api/types';

interface SessionState {
  user: User | null;
  isAuthenticated: boolean;
  biometricUnlocked: boolean;
  setUser: (u: User) => void;
  setBiometricUnlocked: (v: boolean) => void;
  clear: () => void;
}

export const useSession = create<SessionState>((set) => ({
  user: null,
  isAuthenticated: false,
  biometricUnlocked: false,
  setUser: (user) => set({ user, isAuthenticated: true }),
  setBiometricUnlocked: (biometricUnlocked) => set({ biometricUnlocked }),
  clear: () => set({ user: null, isAuthenticated: false, biometricUnlocked: false }),
}));
```

Note `user` here is a convenience mirror set at login; the authoritative, refetchable profile still comes from `useProfile()` (a query). Read from the store for "is someone logged in / what's their name for the header"; read from the query for anything that changes server-side.

Access outside React (e.g. in the api layer) via `useSession.getState()`.

## secure-store — tokens and secrets only

Encrypted device keychain. Small values only (tokens, a PIN hash), never large blobs.

```ts
// src/lib/secureStore.ts
import * as SecureStore from 'expo-secure-store';

const ACCESS = 'app.accessToken';   // namespace your keys
const REFRESH = 'app.refreshToken';

export interface TokenPair { accessToken: string; refreshToken: string }

export async function saveTokens({ accessToken, refreshToken }: TokenPair) {
  await SecureStore.setItemAsync(ACCESS, accessToken);
  await SecureStore.setItemAsync(REFRESH, refreshToken);
}
export const getAccessToken = async () => (await SecureStore.getItemAsync(ACCESS)) ?? null;
export const getRefreshToken = async () => (await SecureStore.getItemAsync(REFRESH)) ?? null;
export async function clearTokens() {
  await SecureStore.deleteItemAsync(ACCESS);
  await SecureStore.deleteItemAsync(REFRESH);
}
```

Tokens live **only** here — they never enter the query cache or a Zustand store, so persisting the cache to disk is safe.

## MMKV — fast local KV (cache + preferences)

Synchronous, fast. Use dedicated instances per concern (an `id` each).

```ts
import { createMMKV } from 'react-native-mmkv';
const prefs = createMMKV({ id: 'app-prefs' });
prefs.set('theme', 'dark');
const theme = prefs.getString('theme');   // get*
prefs.remove('theme');                     // remove (NOT delete)
```

> **API note — this is MMKV v4 (Nitro), not the old v2/v3 API.** v4 exports a
> factory `createMMKV({ id })` and instances use `.remove(key)`. The pre-v4 API
> (`new MMKV({ id })` and `.delete(key)`) is **different** — do not "correct" the
> v4 calls to it. If you pin a different MMKV major, match its API. v4 also
> requires the `react-native-nitro-modules` peer and, being native, a dev build
> (see `references/gotchas.md`).

## Query persistence — warm cache across cold starts

Persist the dehydrated query cache to MMKV so the app opens with last-known data instead of spinners. No sensitive data is ever in the cache (tokens are in secure-store), so plain JSON is fine.

```ts
// src/lib/queryClient.ts
import { QueryClient } from '@tanstack/react-query';
import type { PersistedClient, Persister, PersistQueryClientProviderProps }
  from '@tanstack/react-query-persist-client';
import { createMMKV } from 'react-native-mmkv';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: { staleTime: 30_000, retry: 1, refetchOnWindowFocus: false },
  },
});

const cache = createMMKV({ id: 'app-query-cache' });
const KEY = 'tanstack-query-cache';

export const mmkvPersister: Persister = {
  persistClient: (c: PersistedClient) => cache.set(KEY, JSON.stringify(c)),
  restoreClient: () => {
    const raw = cache.getString(KEY);
    if (!raw) return undefined;
    try { return JSON.parse(raw) as PersistedClient; }
    catch { cache.remove(KEY); return undefined; } // corrupt → drop, start clean
  },
  removeClient: () => cache.remove(KEY),
};

export const persistOptions: PersistQueryClientProviderProps['persistOptions'] = {
  persister: mmkvPersister,
  maxAge: 1000 * 60 * 60 * 24, // trust a persisted cache for 24h
};
```

`refetchOnWindowFocus: false` is deliberate — irrelevant on native and avoids surprise refetches. `staleTime: 30s` keeps reads warm; `retry: 1` avoids hammering the backend on transient failures.

## money / precise numbers — never float

Any time correctness of a number matters (currency, quantities, rates), use `decimal.js`, not JS floats.

```ts
// src/lib/money.ts
import Decimal from 'decimal.js';

export const toDecimal = (v: string | number) => new Decimal(v);

export function formatMoney(amount: string | number, symbol = '$'): string {
  const fixed = new Decimal(amount).toFixed(2);
  const neg = fixed.startsWith('-');
  const [int, dec] = (neg ? fixed.slice(1) : fixed).split('.');
  const grouped = int.replace(/\B(?=(\d{3})+(?!\d))/g, ',');
  return `${neg ? '-' : ''}${symbol}${grouped}.${dec}`;
}
```

## idempotency — safe write retries

A network retry on a "create payment"/"create order" must not double-submit. Send a stable key per attempt; the backend de-dupes. Uniqueness (not crypto strength) is the requirement, so keep it pure-JS to avoid a native crypto dep:

```ts
// src/lib/idempotency.ts
export function newIdempotencyKey(): string {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
    const r = (Math.random() * 16) | 0;
    return (c === 'x' ? r : (r & 0x3) | 0x8).toString(16);
  });
}
```

## Decision table

| Data | Home | Why |
|---|---|---|
| Balances, lists, profile, anything from the server | TanStack Query | refetch, cache, invalidate for free |
| Who's logged in, theme, "biometric unlocked" | Zustand | tiny client state, sync access |
| Access/refresh tokens, PIN | secure-store | encrypted; never in cache |
| Theme choice, onboarding-seen flags, query cache | MMKV | fast, synchronous, survives restart |
| Currency / precise math | decimal.js | floats lose cents |

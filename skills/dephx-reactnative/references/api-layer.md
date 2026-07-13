# API layer — client, hooks, keys, types

## The apiClient — an injectable seam with single-flight 401 refresh

The whole point: the retry/refresh **logic is pure** (built from injected deps), and only the production wiring at the bottom touches axios and secure-store. This makes the tricky part — refresh-once-then-retry under concurrent 401s — unit-testable with fakes.

```ts
// src/lib/apiClient.ts
import axios, { type AxiosInstance } from 'axios';
import Constants from 'expo-constants';
import type { ApiEnvelope, RefreshResponse } from './api/types';
import { clearTokens, getAccessToken, getRefreshToken, saveTokens } from './secureStore';

export interface RequestConfig {
  url: string;
  method?: 'GET' | 'POST' | 'PUT' | 'PATCH' | 'DELETE';
  data?: unknown;
  params?: Record<string, unknown>;
  headers?: Record<string, string>;
}
export interface TransportResponse<T = unknown> { data: T }
export interface Tokens { accessToken: string; refreshToken: string }

/** External dependencies, injected so the factory itself is pure. */
export interface ApiClientDeps {
  transport: <T = unknown>(c: RequestConfig) => Promise<TransportResponse<T>>;
  refresh: () => Promise<Tokens>;
  getAccess: () => Promise<string | null>;
  onRefreshed: (t: Tokens) => Promise<void>;
  onAuthFailure: () => Promise<void>;
}
export interface ApiClient {
  request: <T = unknown>(c: RequestConfig) => Promise<TransportResponse<T>>;
}

function statusOf(err: unknown): number | undefined {
  const e = err as { status?: number; response?: { status?: number } } | null;
  return e?.status ?? e?.response?.status;
}

/** Prefer the backend's own error message over axios's opaque string. */
function backendMessage(err: unknown): string | undefined {
  const m = (err as { response?: { data?: { message?: unknown } } })?.response?.data?.message;
  if (typeof m === 'string' && m.trim()) return m.trim();
  if (Array.isArray(m) && typeof m[0] === 'string') return m[0].trim(); // class-validator array
  return undefined;
}
function normalizeError(err: unknown): unknown {
  const msg = backendMessage(err);
  if (!msg) return err;
  const out = new Error(msg) as Error & { status?: number; cause?: unknown };
  out.status = statusOf(err);
  out.cause = err;
  return out;
}

export function createApiClient(deps: ApiClientDeps): ApiClient {
  const { transport, refresh, getAccess, onRefreshed, onAuthFailure } = deps;
  let inFlight: Promise<Tokens> | null = null; // single-flight: concurrent 401s share one refresh

  function refreshOnce(): Promise<Tokens> {
    if (!inFlight) {
      inFlight = refresh()
        .then(async (t) => { await onRefreshed(t); return t; })
        .catch(async (e) => { await onAuthFailure(); throw e; })
        .finally(() => { inFlight = null; });
    }
    return inFlight;
  }
  async function withAuth(c: RequestConfig, token: string | null): Promise<RequestConfig> {
    return token ? { ...c, headers: { ...c.headers, Authorization: `Bearer ${token}` } } : c;
  }
  async function request<T>(c: RequestConfig): Promise<TransportResponse<T>> {
    const token = await getAccess();
    try {
      return await transport<T>(await withAuth(c, token));
    } catch (err) {
      if (statusOf(err) !== 401) throw normalizeError(err);
      const t = await refreshOnce();                        // refresh once (shared)
      try {
        return await transport<T>(await withAuth(c, t.accessToken)); // retry ONCE
      } catch (retryErr) { throw normalizeError(retryErr); }
    }
  }
  return { request };
}

// --- production wiring (the only part that imports axios / expo) ---
const apiBaseUrl = (Constants.expoConfig?.extra?.apiBaseUrl as string) ?? '';
const axiosInstance: AxiosInstance = axios.create({
  baseURL: apiBaseUrl,
  headers: { 'Content-Type': 'application/json' },
});
const prodTransport: ApiClientDeps['transport'] = async (c) => {
  const res = await axiosInstance.request({
    url: c.url, method: c.method ?? 'GET', data: c.data, params: c.params, headers: c.headers,
  });
  return { data: res.data };
};

let authFailureHandler: () => void | Promise<void> = () => {};
export function setAuthFailureHandler(h: () => void | Promise<void>) { authFailureHandler = h; }

const prodRefresh: ApiClientDeps['refresh'] = async () => {
  const refreshToken = await getRefreshToken();
  const res = await axiosInstance.post<ApiEnvelope<RefreshResponse>>(
    '/auth/refresh', { refreshToken },
  );
  const { accessToken, refreshToken: next } = res.data.data;
  return { accessToken, refreshToken: next };
};

export const apiClient: ApiClient = createApiClient({
  transport: prodTransport,
  refresh: prodRefresh,
  getAccess: getAccessToken,
  onRefreshed: saveTokens,
  onAuthFailure: async () => { await clearTokens(); await authFailureHandler(); },
});
```

Key design points to preserve when you adapt:

- **Single-flight refresh.** Ten screens firing at once, all getting 401, trigger exactly one refresh; the rest await it. No refresh storm.
- **Retry at most once.** A second failure propagates; no infinite loops.
- **Error normalization.** The backend's real message replaces `"Request failed with status code 400"`, with the original on `.cause` and the code on `.status`. UIs show something useful.
- **The app wires `setAuthFailureHandler`** in the root layout to route to login (keeps navigation out of the client).

## Query hooks — one file per endpoint

A read is a thin wrapper: named route const, envelope unwrap, typed return, key from the factory.

```ts
// src/features/profile/api/useProfile.ts
import { useQuery, type UseQueryResult } from '@tanstack/react-query';
import { apiClient } from '@/src/lib/apiClient';
import { queryKeys } from '@/src/lib/queryKeys';
import type { ApiEnvelope, User } from '@/src/lib/api/types';

export const PROFILE_ROUTE = '/profile';

export async function fetchProfile(): Promise<User> {
  const res = await apiClient.request<ApiEnvelope<User>>({ url: PROFILE_ROUTE });
  return res.data.data;
}
export function useProfile(): UseQueryResult<User> {
  return useQuery({ queryKey: queryKeys.profile(), queryFn: fetchProfile });
}
```

## Mutations — with cache invalidation and idempotency

```ts
// src/features/notes/api/useCreateNote.ts
import { useMutation, useQueryClient, type UseMutationResult } from '@tanstack/react-query';
import { apiClient } from '@/src/lib/apiClient';
import { queryKeys } from '@/src/lib/queryKeys';
import { newIdempotencyKey } from '@/src/lib/idempotency';
import type { ApiEnvelope, Note, CreateNoteRequest } from '@/src/lib/api/types';

export const CREATE_NOTE_ROUTE = '/notes';

export async function createNote(body: CreateNoteRequest): Promise<Note> {
  const res = await apiClient.request<ApiEnvelope<Note>>({
    url: CREATE_NOTE_ROUTE,
    method: 'POST',
    data: body,
    headers: { 'Idempotency-Key': newIdempotencyKey() }, // safe retries for writes
  });
  return res.data.data;
}
export function useCreateNote(): UseMutationResult<Note, Error, CreateNoteRequest> {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: createNote,
    onSuccess: () => { qc.invalidateQueries({ queryKey: queryKeys.notes.all() }); },
  });
}
```

## Infinite lists — keep the paging pure

Split the paging math into pure helpers so they can be tested without a component:

```ts
export const PAGE_SIZE = 20;
export function buildParams(page: number) {
  return { skip: page * PAGE_SIZE, take: PAGE_SIZE };
}
// full page ⇒ maybe more; short/empty page ⇒ done
export function getNextPageParam(lastPage: unknown[], allPages: unknown[][]) {
  return lastPage.length < PAGE_SIZE ? undefined : allPages.length;
}
```

Then `useInfiniteQuery({ queryKey, queryFn: ({pageParam}) => fetchPage(pageParam), initialPageParam: 0, getNextPageParam })`, and a `flatten(data) => data?.pages.flat() ?? []` helper for the list.

## queryKeys — one structured factory

Never write `['notes', id]` at a call site. Centralize:

```ts
// src/lib/queryKeys.ts
export const queryKeys = {
  profile: () => ['profile'] as const,
  notes: {
    all: () => ['notes'] as const,
    detail: (id: string) => ['notes', 'detail', id] as const,
    list: (params: { filter?: string }) => ['notes', 'list', params] as const,
  },
} as const;
```

This makes invalidation exact (`invalidateQueries({ queryKey: queryKeys.notes.all() })` clears every notes query) and prevents key drift.

## Types — generate, don't hand-write

The backend contract is the source of truth. Generate types from its OpenAPI spec:

```jsonc
// package.json
"scripts": {
  "gen:api": "openapi-typescript https://api.example.com/openapi.json -o src/lib/api/types.ts"
}
```

Run `npm run gen:api` after any backend DTO change, then update the affected hooks. Define your own `ApiEnvelope<T>` wrapper if the backend wraps payloads (a common `{ status, message, data }` shape):

```ts
export interface ApiEnvelope<T> { status: string; message: string; data: T }
```

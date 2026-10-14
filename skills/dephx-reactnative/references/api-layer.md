# API contracts, authentication and safe writes

## Trace the backend first

Read existing routes, DTOs, callers and error behavior before writing a client. Generate
types from the maintained OpenAPI source when available; otherwise use the repository's
existing contract strategy. Do not invent an envelope, refresh endpoint, pagination
termination rule or idempotency guarantee from an example. A successful malformed list
response is an error, not an empty array; test valid empty and invalid payloads separately.

## Injectable shared client

Reuse the app's client. Keep transport, credential reads and auth lifecycle behind
small seams so response handling can be tested without real secrets or native modules.
Normalize errors without losing status, request identity or unknown-outcome information.
Credentials are attached only to allowed API origins, not arbitrary URLs passed by links.

Authentication behavior is conditional on the backend contract:

- No refresh support: expired/rejected credentials follow the existing reauthentication
  path. Do not add `/auth/refresh` or persist a fictitious refresh token.
- Refresh support: reuse its token rotation and single-flight strategy. Bound replay,
  fence completion to the initiating session and honor invalidation/account switching.
- A 401 is not universal proof that a state-changing operation can be replayed safely.
  Use documented server semantics and existing retry safeguards. Money, sends and
  admission require reconciliation when outcome is uncertain.

## Account-scoped query keys

Use one existing key factory. Authenticated keys include account/tenant scope and every
filter that changes the result. Invalidation stays within that scope.

```ts
export const queryKeys = {
  notes: {
    all: (accountId: string) => ['account', accountId, 'notes'] as const,
    list: (accountId: string, filter: string) =>
      ['account', accountId, 'notes', 'list', filter] as const,
    detail: (accountId: string, id: string) =>
      ['account', accountId, 'notes', 'detail', id] as const,
  },
};
```

Disable account queries until identity is ready. Cancellation must reach the transport;
also prevent late responses from an old session from updating the next session. Follow
`state-storage.md` for persistence, foreground refetch and logout cleanup.

## Reads and records

Hooks return data plus cold loading/error, retained refresh failure and pagination state.
Do not discard retained records on a failed refresh. Query functions do not own editor
draft hydration. Use backend-provided pagination metadata/cursors; a short page is an
end condition only if the API guarantees it. Preserve filters through pagination and
cancel stale search requests without applying a text-search delay to scan processing.

## Writes

Capture resource identity and submitted payload before dispatch. On success update or
invalidate the correct resource/account; preserve edits made after that snapshot. Guard
duplicate presses while pending. A request error is not proof the server rolled back.

For supported idempotency, generate the attempt key once before invoking a mutation and
carry it as mutation input with the original payload. Keep `retry: false` for money and
admission by default. Verify duplicates, timeouts after commit, rejected reuse with a
changed payload and status reconciliation against the real backend. Never make a client
header stand in for a server implementation.

Native links and OAuth handoffs follow `architecture.md`. A GET detail lookup and a
POST that records attendance have different effects; links/prefetch cannot silently
perform writes.

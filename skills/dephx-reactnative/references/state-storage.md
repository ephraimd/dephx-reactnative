# State, privacy and storage

Server records belong in TanStack Query. Small local UI/session decisions belong in
Zustand. Credentials belong in expo-secure-store. Persistence is a separate decision,
not a consequence of adopting Query or Zustand.

## Local state

Use narrow selectors for the fields a component needs. Avoid subscribing the root or a
camera screen to whole stores. Keep authoritative profiles, permissions, balances and
lists in queries; session identity is not permission authority. Keep editor drafts local
to their resource and preserve the submitted snapshot separately from subsequent edits.

## Credentials

Store only credentials supported by the actual backend. A single access token does not
imply refresh-token support. Keep tokens out of query data, URLs, logs and ordinary KV.
SecureStore protects small secrets; it is not a database for invoice/contact blobs or
proof that a cached business record is safe. Check platform backup/uninstall behavior
for the chosen SDK and session policy. Never treat a local biometric flag as server auth.

## Optional persistence

Start with in-memory Query and existing preference storage. Add MMKV when synchronous
KV latency is needed; add SQLite when indexed/durable data needs it. Avoid two persistence
systems for the same job. Match MMKV's installed major and native peers if selected.

Private contacts, messages, invoices and customer information remain sensitive even
when tokens are elsewhere. Before persistence:

1. Classify and allowlist the queries that may be stored; exclude credentials, sensitive
   payloads and mutation queues by default. Use Query's dehydration filters explicitly.
2. Scope authenticated keys, persisters and restoration to the account/tenant and schema
   version. Apply retention bounds and the project's at-rest protection requirements.
3. On logout/account switch, cancel in-flight reads, detach persistence writers, clear
   private memory/disk state and credentials before exposing the next account. Fence old
   responses and refreshes by session identity so they cannot repopulate cleared state.
4. Test switching accounts during slow reads, writes, refresh and cold-cache restore.
   Never label cached balances, permissions or ticket validity authoritative or current.

A 24-hour cache expiry is a retention choice, not permission to trust data for 24 hours.
Wire TanStack focus to AppState and online state to the app's existing connectivity
source. Refetch appropriate stale reads on foreground/reconnect without overwriting
drafts or replaying mutations. Background execution and push delivery are not guaranteed.

## Money and attempt identity

Reuse server-computed totals and the existing currency contract, commonly decimal
strings or integer minor units. Zero is valid; missing fees show recovery, not fabricated
defaults. Add decimal.js only when necessary decimal arithmetic has no existing solution.

Do not generate a fresh idempotency key inside each network retry. Create one identity
for a logical user attempt through the existing UUID facility, then reuse it only with
the same payload under a verified backend idempotency contract. A header alone does not
de-duplicate writes. Keep money/admission mutation retries off by default; reconcile
uncertain results before allowing another attempt. Do not persist offline admission or
payment queues unless separately authorized and implemented with server conflict rules.

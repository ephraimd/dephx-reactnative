# Testing — colocated, fast, mostly pure

The architecture pays off here: because logic lives in pure `model/` functions and network access is behind an injectable seam, you get high confidence from fast unit tests and rarely need a device or an E2E harness.

## Setup

`package.json`:

```jsonc
"scripts": { "test": "jest" },
"devDependencies": {
  "jest": "~29.7.0",
  "jest-expo": "^57.0.1",
  "@testing-library/react-native": "^14.0.1",
  "@types/jest": "^29.5.0"
}
```

> `@testing-library/react-native` v12.4+ bundles the jest-native matchers
> (`toBeVisible`, `toHaveTextContent`, …), so you do **not** need the separate
> `@testing-library/jest-native` package. If you inherit an older setup that
> still imports `@testing-library/jest-native/extend-expect`, it's harmless but
> redundant — drop it and the matchers still work.

`jest.config.js` — the `transformIgnorePatterns` allowlist is required so Jest transpiles the RN/Expo/nativewind ESM packages:

```js
/** @type {import('jest').Config} */
module.exports = {
  preset: 'jest-expo',
  setupFilesAfterEnv: ['<rootDir>/jest.setup.ts'],
  transformIgnorePatterns: [
    'node_modules/(?!((jest-)?react-native|@react-native(-community)?|expo(nent)?|@expo(nent)?/.*|@expo-google-fonts/.*|react-navigation|@react-navigation/.*|@unimodules/.*|unimodules|sentry-expo|native-base|react-native-svg|nativewind|react-native-css-interop|@gluestack-ui/.*|@legendapp/motion))',
  ],
};
```

`jest.setup.ts` — mock native modules with in-memory fakes so unit tests never touch the device:

```ts
// jest.setup.ts — native-module fakes so unit tests never touch the device.
// (RNTL v12.4+ bundles the jest-native matchers; no extra import needed.)

jest.mock('expo-secure-store', () => {
  const store: Record<string, string> = {};
  return {
    __esModule: true,
    getItemAsync: jest.fn(async (k: string) => (k in store ? store[k] : null)),
    setItemAsync: jest.fn(async (k: string, v: string) => { store[k] = v; }),
    deleteItemAsync: jest.fn(async (k: string) => { delete store[k]; }),
  };
});

jest.mock('react-native-mmkv', () => {
  const mem = new Map<string, string>();
  return {
    createMMKV: () => ({
      set: (k: string, v: string) => mem.set(k, String(v)),
      getString: (k: string) => mem.get(k),
      remove: (k: string) => mem.delete(k),
    }),
  };
});
```

## What to test (in priority order)

1. **`model/` functions — always.** They're pure; there's no excuse. Formatting, derivations, filter/param builders, pagination math, state-machine transitions.
2. **`api/` helpers — the pure parts.** `buildParams`, `getNextPageParam`, envelope-unwrap fallbacks (empty payload → empty list). Test the request/refresh logic of `apiClient` by constructing it with **fake deps** — no axios, no network.
3. **Components — only where there's branching worth pinning** (an empty state vs list, a disabled button). Presentational rows given a display object rarely need a test.

You typically do **not** write E2E tests. If a flow is critical enough to need device-level verification, do it manually on a dev build or add Maestro later — don't block feature work on it.

## Example: testing the pure model

```ts
// src/features/notes/model/noteDisplay.test.ts
import { toNoteDisplay } from './noteDisplay';

describe('toNoteDisplay', () => {
  it('active → success tone', () => {
    expect(toNoteDisplay({ title: 'Hi', status: 'ACTIVE' }))
      .toEqual({ title: 'Hi', tone: 'success' });
  });
  it('archived → danger tone', () => {
    expect(toNoteDisplay({ title: 'x', status: 'ARCHIVED' }).tone).toBe('danger');
  });
  it('falls back to Untitled', () => {
    expect(toNoteDisplay({ status: 'ACTIVE' }).title).toBe('Untitled');
  });
});
```

## Example: testing apiClient logic with fake deps (no network)

This is the payoff of the injectable seam — the tricky refresh-once-retry logic is a pure unit test:

```ts
// src/lib/apiClient.test.ts
import { createApiClient } from './apiClient';

it('refreshes once on 401, then retries the request', async () => {
  let calls = 0;
  const transport = jest.fn(async () => {
    calls += 1;
    if (calls === 1) throw { status: 401 };   // first call unauthorized
    return { data: 'ok' };                      // retry succeeds
  });
  const refresh = jest.fn(async () => ({ accessToken: 'new', refreshToken: 'r2' }));

  const client = createApiClient({
    transport,
    refresh,
    getAccess: async () => 'old',
    onRefreshed: async () => {},
    onAuthFailure: async () => {},
  });

  const res = await client.request({ url: '/x' });
  expect(res.data).toBe('ok');
  expect(refresh).toHaveBeenCalledTimes(1);
  expect(transport).toHaveBeenCalledTimes(2);
});
```

## Discipline

- Keep `npx tsc --noEmit` and `npm test` green before moving to the next task. Strict TS catches a whole class of bugs the tests won't.
- Colocate: `x.ts` next to `x.test.ts`. A feature folder shows its own coverage at a glance.
- Write the test with the code (or first). If a piece is hard to test, a responsibility is misplaced — move it to `model/`, don't reach for a heavier mock.

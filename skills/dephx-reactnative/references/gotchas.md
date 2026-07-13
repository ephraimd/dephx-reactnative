# Gotchas — the time-sinks, and how to avoid them

Read this once before building. Each item cost real hours to learn. Symptom → cause → fix.

## Quick table

| Symptom | Cause | Fix |
|---|---|---|
| App crashes on boot after adding a native lib | Lib doesn't support the New Architecture | Verify New-Arch support before adding; rebuild the dev client |
| `npm install` fails with `ERESOLVE` | React 19 / RN 0.86 peer ranges lag | `.npmrc` → `legacy-peer-deps=true` |
| "worklet" runtime error / animations silently dead | Worklets babel plugin not last | Make `react-native-worklets/plugin` the LAST babel plugin |
| Styles don't apply / `className` ignored | NativeWind wiring or stale cache | Check babel+metro+global.css trio; `npx expo start -c` |
| Native module is `undefined` at runtime | Running in Expo Go / cache | Use a dev build (`expo run:*`); rebuild after adding native deps |
| Edits to `android/` or `ios/` disappear | CNG regenerates them | Edit `app.config.ts` or write a config plugin |
| Endpoint 404s / misroutes intermittently | Trailing slash on the route string | Remove it: `'/notes'` not `'/notes/'` |
| Gradle build fails cryptically | Wrong JDK | Use JDK 21 |
| MMKV native link error | Missing Nitro peer | Install `react-native-nitro-modules`; rebuild |
| `createMMKV is not a function` / `.remove is not a function` | Mixed MMKV v4 (Nitro) with the old v2/v3 API | v4 = `createMMKV({id})` + `.remove()`; v2/v3 = `new MMKV()` + `.delete()`. Match your pinned major — don't cross them |
| Config change has no effect | Metro cache | `npx expo start -c` |

## The big ones, explained

### CNG: `android/` and `ios/` are generated — never hand-edit
This project uses Continuous Native Generation. The native folders are (or should be) git-ignored and rebuilt from `app.config.ts` + config plugins by `expo prebuild`. Anything you type directly into them is wiped on the next clean prebuild. All native config — app name, bundle IDs, version, icons, splash, permissions, plugins — is set in `app.config.ts`. If you need a native change no plugin exposes, write a config plugin so it survives. `.gitignore` should contain `/android` and `/ios`.

### New Architecture is ON — every native dep must support it
`newArchEnabled=true` (RN 0.86 default). A dependency that hasn't migrated to Fabric/TurboModules will crash the app on launch, often with an opaque native error that looks unrelated. **Before adding any native library, confirm New-Arch support.** This is the #1 "works in another RN app but not here" trap.

### Worklets plugin ordering (Reanimated v4)
Reanimated v4 moved its babel plugin into `react-native-worklets`. In `babel.config.js`, `'react-native-worklets/plugin'` must be the **last** entry in `plugins`. Wrong (or missing) → cryptic worklet errors or animations that just don't run. Don't reorder it when adding other plugins; keep it last.

### `legacy-peer-deps` is intentional
`.npmrc` sets `legacy-peer-deps=true` because React 19 + RN 0.86 outpace some libraries' declared peer ranges. Without it, install fails. Don't "fix" it by deleting the flag or blindly `--force`; the flag already handles the common case. Investigate only a specific genuinely-conflicting package.

### Native modules require a dev build, not Expo Go
MMKV, secure-store, local-authentication, reanimated worklets, etc. don't exist in the stock Expo Go client. Run a **development build** (`npx expo run:android` / `run:ios`, or an EAS `development` profile). After adding or upgrading any native module, a Metro reload is not enough — rebuild the dev client.

### NativeWind is a three-file contract
Styling depends on all three cooperating: `global.css` imported at the top of `app/_layout.tsx`; `metro.config.js` wrapping with `withNativeWind(config, { input: './global.css' })`; `babel.config.js` with `jsxImportSource: 'nativewind'` + the `nativewind/babel` preset. If classes stop applying after a bump, verify all three, then clear cache (`npx expo start -c`).

### No trailing slash on route strings
A trailing `/` on an API path can cause the backend to 404 or misroute. Keep route consts clean (`'/auth/login'`, not `'/auth/login/'`). Harden the backend to ignore trailing slashes if you can, but don't rely on it.

### Base URL comes from `extra`, never hardcoded
Set it in `app.config.ts` `extra.apiBaseUrl` (defaulting from `process.env.API_BASE_URL`) and read it via `Constants.expoConfig.extra.apiBaseUrl`. This lets a build target dev/staging/prod without a code change. Never inline a URL in `apiClient.ts`.

### JDK 21 for Android
Android builds expect JDK 21. A different JDK (17, 23) produces Gradle failures that masquerade as dependency problems.

## The reliable reset

When something native misbehaves after an upgrade and you can't pin it down, nuke the generated + cached layers (safe, because native folders are generated):

```bash
rm -rf node_modules android ios
npm install
npx expo prebuild --clean
npx expo start -c
```

## ALWAYS / NEVER checklist

**ALWAYS**
- Keep server state in TanStack Query; only session/UI in Zustand.
- Route network calls through `apiClient`.
- Put logic in pure `model/` with a colocated test.
- Central `constants.ts` / `queryKeys.ts` / theme tokens for shared literals.
- Rebuild the dev client after touching native deps.
- Keep `tsc --noEmit` and `jest` green before moving on.

**NEVER**
- Edit `android/` or `ios/` by hand.
- Add a native dep without checking New-Arch support.
- Hardcode a URL, route, color, or magic number in a component.
- Put a trailing slash on a route string.
- Duplicate server data into a Zustand store.
- Store tokens anywhere but secure-store.

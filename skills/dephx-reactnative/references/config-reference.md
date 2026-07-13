# Config reference — the wiring that interlocks

These files are version-sensitive and easy to get subtly wrong. Copy them as a set. Symptoms of a mistake are listed in `references/gotchas.md`.

## Version matrix (baseline)

Verified together. If you bump one, re-check the others against the Expo SDK upgrade notes.

| Package | Version | Note |
|---|---|---|
| expo | ~57.0 | SDK baseline |
| react-native | 0.86 | New Architecture on by default |
| react / react-dom | 19.2 | |
| expo-router | ~57.0 | file-based routing |
| nativewind | ^4.2 | Tailwind for RN |
| tailwindcss | ^3.4 | v3 (NativeWind v4 targets Tailwind 3) |
| react-native-reanimated | 4.5 | v4 → plugin moved into worklets |
| react-native-worklets | 0.10 | provides the babel plugin |
| react-native-mmkv | ^4.3 | needs react-native-nitro-modules |
| react-native-nitro-modules | ^0.36 | MMKV v4 peer |
| @tanstack/react-query | ^5.101 | + react-query-persist-client |
| zustand | ^5.0 | |
| axios | ^1.18 | |
| decimal.js | ^10.6 | precise numbers |
| expo-secure-store | ~57.0 | tokens |
| typescript | ~6.0 | strict |
| jest-expo | ^57.0 | test preset |

## `.npmrc` — required

```
legacy-peer-deps=true
```

React 19 + RN 0.86 are new enough that some libraries' `peerDependencies` ranges lag. Without this, `npm install` fails with `ERESOLVE`. This is intentional; keep it. (If you prefer yarn/pnpm, set the equivalent.)

## `app.config.ts` — the source of truth (CNG)

`android/` and `ios/` are generated from this file (Continuous Native Generation). This is where identifiers, version, icons, splash, permissions, and plugins live. Never edit the native folders directly.

```ts
import type { ExpoConfig, ConfigContext } from 'expo/config';

const DEFAULT_API_BASE_URL = 'https://api.example.com';

export default ({ config }: ConfigContext): ExpoConfig => ({
  ...config,
  name: 'MyApp',
  slug: 'myapp',
  scheme: 'myapp',                 // deep-link scheme
  version: '1.0.0',                // user-facing marketing version
  orientation: 'portrait',
  icon: './assets/icon.png',
  userInterfaceStyle: 'light',
  ios: {
    supportsTablet: true,
    bundleIdentifier: 'com.yourorg.myapp',
  },
  android: {
    package: 'com.yourorg.myapp',
    versionCode: 1,                // integer, must increase per store upload
    adaptiveIcon: {
      backgroundColor: '#7D30D6',
      foregroundImage: './assets/adaptive-icon.png',
    },
    predictiveBackGestureEnabled: false,
  },
  plugins: [
    'expo-router',
    'expo-secure-store',
    ['expo-splash-screen', {
      image: './assets/splash.png',
      imageWidth: 280,
      resizeMode: 'contain',
      backgroundColor: '#7D30D6',
    }],
    // add expo-local-authentication, expo-web-browser, etc. as features need them
  ],
  extra: {
    // Read at runtime via Constants.expoConfig.extra.apiBaseUrl.
    // Override per environment with the API_BASE_URL env var (or eas.json profiles).
    apiBaseUrl: process.env.API_BASE_URL ?? DEFAULT_API_BASE_URL,
  },
});
```

Runtime access (in `apiClient.ts`):

```ts
import Constants from 'expo-constants';
const apiBaseUrl = (Constants.expoConfig?.extra?.apiBaseUrl as string) ?? '';
```

Never hardcode the base URL in source — thread it through `extra` so builds can retarget without a code change.

## `babel.config.js` — worklets plugin MUST be last

```js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ['babel-preset-expo', { jsxImportSource: 'nativewind' }],
      'nativewind/babel',
    ],
    plugins: [
      // Reanimated v4 moved its babel plugin into react-native-worklets.
      // It MUST be the LAST plugin. Wrong order → "worklet" runtime errors
      // or animations silently not running.
      'react-native-worklets/plugin',
    ],
  };
};
```

## `metro.config.js` — NativeWind transform

```js
const { getDefaultConfig } = require('expo/metro-config');
const { withNativeWind } = require('nativewind/metro');

const config = getDefaultConfig(__dirname);
module.exports = withNativeWind(config, { input: './global.css' });
```

## `tailwind.config.js` + `global.css` — semantic tokens

Colors are CSS variables (RGB channels so Tailwind opacity modifiers like `bg-primary/50` work), defined light in `:root` and dark in `.dark`, and mirrored in `src/theme/` for JS access.

`tailwind.config.js`:

```js
/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    './app/**/*.{js,jsx,ts,tsx}',
    './src/**/*.{js,jsx,ts,tsx}',
    './components/**/*.{js,jsx,ts,tsx}',
  ],
  presets: [require('nativewind/preset')],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        background: 'rgb(var(--color-background) / <alpha-value>)',
        foreground: 'rgb(var(--color-foreground) / <alpha-value>)',
        card: {
          DEFAULT: 'rgb(var(--color-card) / <alpha-value>)',
          foreground: 'rgb(var(--color-card-foreground) / <alpha-value>)',
        },
        muted: {
          DEFAULT: 'rgb(var(--color-muted) / <alpha-value>)',
          foreground: 'rgb(var(--color-muted-foreground) / <alpha-value>)',
        },
        border: 'rgb(var(--color-border) / <alpha-value>)',
        primary: {
          DEFAULT: 'rgb(var(--color-primary) / <alpha-value>)',
          foreground: 'rgb(var(--color-primary-foreground) / <alpha-value>)',
        },
        success: 'rgb(var(--color-success) / <alpha-value>)',
        danger: 'rgb(var(--color-danger) / <alpha-value>)',
      },
    },
  },
  plugins: [],
};
```

`global.css` (imported once at the top of `app/_layout.tsx`):

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  :root {
    --color-background: 255 255 255;
    --color-foreground: 29 35 41;
    --color-card: 243 245 246;
    --color-card-foreground: 29 35 41;
    --color-muted: 238 240 242;
    --color-muted-foreground: 88 98 131;
    --color-border: 226 229 233;
    --color-primary: 125 48 214;
    --color-primary-foreground: 255 255 255;
    --color-success: 22 163 74;
    --color-danger: 220 38 38;
  }
  .dark:root, .dark {
    --color-background: 29 35 41;
    --color-foreground: 243 245 246;
    --color-card: 37 44 51;
    --color-card-foreground: 243 245 246;
    --color-muted: 44 52 60;
    --color-muted-foreground: 147 160 180;
    --color-border: 51 59 68;
    --color-primary: 125 48 214;
    --color-primary-foreground: 255 255 255;
    --color-success: 34 197 94;
    --color-danger: 239 68 68;
  }
}
```

`nativewind-env.d.ts` (so TS accepts `className` and `*.css` imports):

```ts
/// <reference types="nativewind/types" />
declare module '*.css';
```

## `tsconfig.json`

```json
{
  "extends": "expo/tsconfig.base",
  "compilerOptions": {
    "strict": true,
    "types": ["jest", "node"],
    "paths": { "@/*": ["./*"] }
  },
  "include": ["**/*.ts", "**/*.tsx", "nativewind-env.d.ts"]
}
```

`@/*` maps to the project root, so imports read `@/src/lib/apiClient`, `@/src/components/ScreenHeader`.

## `eas.json` — build profiles (optional; needed for cloud builds)

```json
{
  "cli": { "version": ">= 12.0.0", "appVersionSource": "remote" },
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal",
      "env": { "API_BASE_URL": "https://api.example.com" }
    },
    "preview": {
      "distribution": "internal",
      "env": { "API_BASE_URL": "https://api.example.com" }
    },
    "production": { "autoIncrement": true }
  },
  "submit": { "production": {} }
}
```

Per-profile `env.API_BASE_URL` is how you point QA/preview at a different backend. Store submission itself is out of scope for this skill.

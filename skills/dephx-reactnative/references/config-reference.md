# Configuration: version-specific examples

## Resolve a supported matrix first

The original v0.1.0 recipe named Expo 57 / RN 0.86 / React 19.2 / NativeWind v4.
That historical example is not a verified current compatibility matrix. Do not pin it
as a bundle or copy it over an existing app. Inspect the SDK, lockfile and UI kit;
use Expo-compatible versions and validate them with the project's package manager.
Run the selected SDK's install check/Expo Doctor and the relevant native runtime tests.
Record the resolved versions, sources, verification date and remaining limitations.

The Babel/Metro/Tailwind snippets below illustrate NativeWind v4 with Tailwind v3.
They do not apply unchanged to NativeWind v5/Tailwind v4. Confirm the installed
Reanimated/worklets configuration rather than adding a second transform blindly.

Primary sources: [Expo SDK reference](https://docs.expo.dev/versions/latest/),
[NativeWind installation](https://www.nativewind.dev/docs/getting-started/installation),
[Gluestack installation](https://gluestack.io/ui/docs/home/getting-started/installation).

Keep the existing package manager. Resolve peer conflicts; no blanket `.npmrc`
`legacy-peer-deps` requirement. See `gotchas.md` for recovery ownership.

## `app.config.ts` - the source of truth (CNG)

For a confirmed CNG project, app configuration and config plugins own generated native settings. Verify native-folder ownership before regeneration. The following identifiers and colors are illustrative; replace them with the project's canonical values. Build configuration is public, not secret storage.

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

Do not hardcode origins in request/client code. Use the existing public build configuration; example defaults above are illustrative, not production destinations. Validate required origins before sending requests.

## `babel.config.js`: NativeWind v4 / worklets example

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

## `metro.config.js` - NativeWind transform

```js
const { getDefaultConfig } = require('expo/metro-config');
const { withNativeWind } = require('nativewind/metro');

const config = getDefaultConfig(__dirname);
module.exports = withNativeWind(config, { input: './global.css' });
```

## `tailwind.config.js` + `global.css` - semantic tokens

Colors are CSS variables (RGB channels so Tailwind opacity modifiers like `bg-primary/50` work), defined light in `:root` and dark in `.dark`, with native values derived or validated against the same canonical source. These illustrative snippets do not establish complete native provider wiring; follow the selected major's official recipe and test actual devices.

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

## `eas.json` - build profiles (optional; needed for cloud builds)

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

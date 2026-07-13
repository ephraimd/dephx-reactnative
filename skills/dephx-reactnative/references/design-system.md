# Design system — tokens, theming, components

## Semantic tokens, not raw colors

Components reference **roles** (`bg-background`, `text-foreground`, `border-border`, `text-muted-foreground`, `bg-primary`), never raw hex. The role → value mapping lives once, in CSS variables (see `references/config-reference.md` for the full `global.css`). This is what makes light/dark a single switch and a rebrand a five-line diff.

```tsx
// Good — role-based, themes automatically
<View className="bg-card border border-border rounded-2xl p-4">
  <Text className="text-foreground text-lg font-semibold">Title</Text>
  <Text className="text-muted-foreground">Subtitle</Text>
</View>

// Bad — raw color, breaks in dark mode, can't rebrand
<View style={{ backgroundColor: '#F3F5F6' }}>…</View>
```

Semantic roles to standardize on: `background`, `foreground`, `card(+foreground)`, `muted(+foreground)`, `border`, `primary(+foreground)`, `success`, `danger`. Add more only when a real, repeated need appears (YAGNI).

## Two ways to read the theme

1. **In JSX:** Tailwind classes (`text-foreground`) — resolved by NativeWind, flips with the scheme. Use this by default.
2. **In JS (rare):** when you must pass a concrete color to a non-className prop (an icon color, an SVG fill), read from a JS mirror of the tokens keyed by scheme.

Keep a JS mirror (`src/theme/theme.ts`) in sync with the CSS vars — the same hex values, keyed by scheme — so JS-side code reads the exact color the CSS uses:

```ts
// src/theme/theme.ts
export type ColorScheme = 'light' | 'dark';

export const themes = {
  light: {
    background: '#FFFFFF', foreground: '#1D2329',
    card: '#F3F5F6', muted: '#EEF0F2', mutedForeground: '#586283',
    border: '#E2E5E9', primary: '#7D30D6',
    success: '#16A34A', danger: '#DC2626',
  },
  dark: {
    background: '#1D2329', foreground: '#F3F5F6',
    card: '#252C33', muted: '#2C343C', mutedForeground: '#93A0B4',
    border: '#333B44', primary: '#7D30D6',
    success: '#22C55E', danger: '#EF4444',
  },
} as const;
```

Usage: `const theme = themes[scheme]; <Icon color={theme.foreground} />`. Then the mode store:

```ts
// src/theme/useTheme.ts
import type { ColorScheme } from './theme';
import { Appearance } from 'react-native';
import { colorScheme } from 'nativewind';
import { createMMKV } from 'react-native-mmkv';
import { create } from 'zustand';

export type ThemeMode = 'light' | 'dark' | 'system';

const storage = createMMKV({ id: 'app-theme' });
const KEY = 'app.theme';
const isMode = (v: unknown): v is ThemeMode => v === 'light' || v === 'dark' || v === 'system';
const load = (): ThemeMode => { const r = storage.getString(KEY); return isMode(r) ? r : 'light'; };
const resolve = (m: ThemeMode): ColorScheme =>
  m === 'system' ? (Appearance.getColorScheme() === 'dark' ? 'dark' : 'light') : m;

interface ThemeState { mode: ThemeMode; resolvedScheme: ColorScheme; setMode: (m: ThemeMode) => void }
const initial = load();

export const useTheme = create<ThemeState>((set) => ({
  mode: initial,
  resolvedScheme: resolve(initial),
  setMode: (mode) => {
    storage.set(KEY, mode);
    colorScheme.set(mode);              // flips `dark:` classes + CSS-var themes immediately
    set({ mode, resolvedScheme: resolve(mode) });
  },
}));
```

The root layout also runs `useEffect(() => colorScheme.set(mode), [mode])` so the restored preference is applied on cold start (see `references/architecture.md`).

## Component conventions

- **Presentational components take data, not queries.** A `NoteRow` receives an already-derived display object; it never calls a hook or formats anything. Logic lives in `model/`.
- **One shared header.** A single `ScreenHeader` (back affordance + centered title + optional right action) used on every stack screen gives predictable navigation. Screens render it inside a `SafeAreaView` top edge rather than using the native nav header (`headerShown: false`).
- **Accessibility is not optional:** `accessibilityRole`, `accessibilityLabel`, and generous `hitSlop` on every touchable. Pressables show an `active:` state.
- **Shared vs feature components:** app-wide, domain-agnostic pieces go in `src/components/` (`ScreenHeader`, `AppInput`, `PinConfirm`, `PaginatedList`); anything specific to one feature stays in that feature's `ui/`.

Example of the "given a display object, just render" rule:

```tsx
// src/features/notes/ui/NoteRow.tsx
import { View, Text } from 'react-native'; // or your primitive wrapper (see below)
import type { NoteDisplay } from '../model/noteDisplay';

const TONE: Record<NoteDisplay['tone'], string> = {
  success: 'text-success',
  danger: 'text-danger',
  muted: 'text-muted-foreground',
};

export function NoteRow({ note }: { note: NoteDisplay }) {
  return (
    <View className="flex-row items-center justify-between border-b border-border py-3">
      <Text className="text-foreground" numberOfLines={1}>{note.title}</Text>
      <Text className={TONE[note.tone]}>●</Text>
    </View>
  );
}
```

## Optional: Gluestack primitives

Gluestack UI primitives (`Box`, `Text`, `HStack`, `Button`, …) pair well with NativeWind for accessible, styleable base components. They're optional — plain `View`/`Text` with `className` is enough for most apps. If you adopt them, keep them under `components/ui/` and still drive colors through the same semantic tokens.

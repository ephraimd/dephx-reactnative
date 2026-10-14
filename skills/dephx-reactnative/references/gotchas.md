# Expo configuration and recovery

## Diagnose before resetting

Read the installed SDK, lockfile, Babel/Metro configuration, native-module versions
and the actual error. Use the project's package manager; preserve lockfiles and local
native work. Clear a targeted cache only after identifying it as disposable.

| Symptom | Check before changing |
|---|---|
| Native crash after dependency change | SDK/architecture support, native peers and rebuilt development client |
| Peer dependency conflict | Actual dependency ranges and SDK-compatible versions; do not globally bypass peers |
| Worklet error | Installed Reanimated/worklets major and its documented Babel wiring |
| NativeWind styles ignored | Matching NativeWind/Tailwind major and that major's Metro/Babel/CSS setup |
| Native module absent | Whether this SDK's Expo Go includes it and whether config needs a development build |
| Native edits disappear | Whether CNG owns these folders; use config plugins for generated projects |
| Endpoint mismatch | Real backend route/redirect semantics, not a blanket slash-rewrite rule |
| Android toolchain error | Project Gradle/AGP/JDK matrix, not a universal JDK version |
| Storage API mismatch | Installed MMKV major, peer modules and native rebuild |

## Peer dependencies

Do not add `legacy-peer-deps=true`, `--force` or another global bypass as the default
resolution. Find the conflicting package and use a supported matrix. A narrowly scoped
exception needs the project's approval policy and actual runtime verification. A Yarn
project stays Yarn; do not add an npm lockfile to make installation pass.

## Expo Go versus development builds

Support is module- and SDK-specific. SecureStore and Reanimated cannot be declared
absent from every Expo Go version. Check the installed SDK's supported-module docs.
Use a development build for native modules/configuration not included in Expo Go,
and rebuild after native dependencies/config change. Metro reload alone is insufficient.

## NativeWind and worklets

NativeWind v4/Tailwind v3 examples are not NativeWind v5/Tailwind v4 instructions.
Use the selected major's official installation recipe; do not combine them. Worklets
plugin requirements/order also depend on the installed major and Expo preset. Match
that configuration instead of adding duplicate transforms.

## CNG and recovery ownership

For confirmed CNG projects, app configuration/config plugins own generated native
settings. Existing bare or manually maintained native projects have different ownership.
Inspect tracked files and local changes before regeneration. Preserve manual work first.

There is no blanket `rm -rf node_modules android ios` recovery recipe. Choose the
smallest justified reset, regenerate only confirmed disposable outputs and reinstall
using the existing manager/lockfile. A clean prebuild can erase native changes; its
consequences must be explained before executing it. Never remove signing material,
credentials, databases or user-created assets as build-cache cleanup.

## Public build configuration

API origins and app identifiers come from the project's existing build configuration.
Values shipped through Expo `extra` are public, not a safe home for server secrets.
Check allowed origins before attaching credentials. SDK upgrade success requires
native runtime and relevant performance checks, not only an install/build pass.

# Verification: logic, native UX and performance

Pure models and injectable I/O make fast tests useful. They do not exercise keyboard
insets, camera permissions, native links, real accessibility focus or release performance.
Match evidence to the changed behavior and the project's acceptance requirements.

## Automated checks

Reuse the app's package manager, SDK-compatible jest-expo setup and existing suite.
Colocate focused model tests. Use Testing Library for meaningful component branches,
accessible names and draft/state transitions; avoid snapshots that only mirror markup.
Native-module fakes test logic, not actual SecureStore, camera or device behavior.

Exercise the failures a change can cause:

- Valid zero fee versus missing response; valid empty list versus malformed/error response.
- Refetch while dirty, edit during Save, save failure and resource switching.
- Logout/account switching while reads, cache restoration or refresh are pending.
- Refresh-less authentication, supported refresh concurrency and bounded replay.
- Duplicate presses/scans, timeout after server commit and stable attempt identity.

Keep types and the project's required checks green. Generating API types or building
successfully does not prove compatibility or the running user flow.

## Native runtime acceptance

Run actual components and routes. Cover zero/one/many records, long names/URLs, missing
imagery, slow and failed network, stale data, permissions and preserved drafts. Inspect
the smallest supported phone, a short viewport and tablet when supported, both themes,
large text, VoiceOver/TalkBack, Android back and reduced motion.

For forms/overlays verify keyboard-open final fields, reachable Save/close, independent
scrolling, safe areas and bottom navigation collisions. Test last-row menus and return
to the correct opener. For links test cold/warm start, signed-out restoration, malformed
paths and native/browser fallback. For camera workflows test real hardware, denied
permission, repeated frames, background/foreground, wrong/refunded codes and an uncertain
server result. Simulators cannot certify those hardware paths.

Use the existing E2E tool where it supports the required flow; manual device checks are
valid when recorded. No new harness is required just for ceremony. Implementation may
continue without a device, but the affected release acceptance remains unverified until
required checks are completed or a human explicitly approves a recorded exception.

## Release performance gate

Measure release builds on representative physical devices, including a modest Android
phone and an older supported iPhone when both platforms ship. Record hardware, OS, SDK,
library versions, build mode, dataset and observed measurements. Dev-mode simulator
success is not a performance verdict.

Define budgets appropriate to the product before broad rollout. Measure cold/warm launch,
touch response, navigation and list frame times, memory and sustained camera behavior;
include battery/thermal effects where a long-running feature requires them. Separate
scan decoding, UI feedback, API latency and server contention. Instant acknowledgement
must not masquerade as confirmed admission.

UI libraries and NativeWind have runtime costs; do not claim zero overhead. Compare
representative shared controls if profiling identifies a bottleneck. First address
unbounded rows, repeated layout, broad store subscriptions, heavy imagery and unnecessary
rerenders. Improve the shared foundation rather than fork another component family.
Vendor benchmarks are scoped evidence, not certification of the selected app/version.

## Evidence record

Follow existing repository evidence gates. Record loaded canonical sources/versions,
changed paths, automated results, exercised route/state/device coverage, inspected
captures and limitations. Source review, fixtures, simulator runs, physical-device UX,
release measurements and production behavior are distinct evidence. A passing guidance
scenario test proves instruction application only; it does not prove an app works.

Primary references to check for the selected versions:
- [React Native performance](https://reactnative.dev/docs/performance)
- [Expo testing](https://docs.expo.dev/develop/unit-testing/)
- [TanStack Query React Native integration](https://tanstack.com/query/latest/docs/framework/react/react-native)

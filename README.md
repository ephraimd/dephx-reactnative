# dephx-reactnative

A portable **skill** that gives coding agents (Claude Code, OpenCode, and any tool that reads the [Agent Skills](https://agentskills.io/specification) format) a production-grade foundation for building **Expo / React Native** apps.

It's the distilled, de-branded playbook from a shipped app: the stack, the architecture, the exact (fragile) config wiring, the data/state/storage/theming patterns, the testing doctrine, and the hard-won Expo build gotchas. Add it to a project and the agent scaffolds and extends a mobile app that is structured, testable, and correct from the first screen — for any app domain, not a specific one.

**Baseline:** Expo SDK 57 · React Native 0.86 · React 19.2 · New Architecture ON · TypeScript strict.

---

## What's inside

```
.claude-plugin/
  marketplace.json             marketplace "dephx" (enables one-command install)
  plugin.json                  plugin manifest
skills/dephx-reactnative/
  SKILL.md                     entry point: stack, workflow, guardrails, routing
  references/
    philosophy.md              SOLID / KISS / DRY / YAGNI as concrete RN conventions
    architecture.md            folder structure, feature-slice pattern, root wiring
    config-reference.md        the exact config quartet + version matrix
    api-layer.md               injectable apiClient, query hooks, mutations, keys, type-gen
    state-storage.md           Zustand / secure-store / MMKV / query-persist / money
    design-system.md           semantic tokens, light/dark, component conventions
    testing.md                 jest-expo setup, mocks, what to test
    gotchas.md                 mistakes-learned → DO-NOT / ALWAYS
```

The skill is pure knowledge — no dependencies, nothing to build. The agent reads it and applies it to your project.

---

## Install

### Claude Code — one-command (recommended)

```
/plugin marketplace add ephraimd/dephx-reactnative
/plugin install dephx-reactnative@dephx
```

The skill activates automatically when you start or extend an Expo / React Native app. To update later, run `/plugin marketplace update dephx` and reinstall if prompted.

### Claude Code — manual copy

```bash
git clone https://github.com/ephraimd/dephx-reactnative.git
# personal (all projects):
cp -r dephx-reactnative/skills/dephx-reactnative ~/.claude/skills/
# or project-local:
cp -r dephx-reactnative/skills/dephx-reactnative <your-project>/.claude/skills/
```

### OpenCode

```bash
git clone https://github.com/ephraimd/dephx-reactnative.git
# project-local:
cp -r dephx-reactnative/skills/dephx-reactnative <your-project>/.opencode/skills/
# or global:
cp -r dephx-reactnative/skills/dephx-reactnative ~/.config/opencode/skills/
```

Any other Agent-Skills-compatible tool: point it at the `skills/dephx-reactnative/` folder.

---

## Usage

Once installed, just describe the work:

- *"Scaffold a new Expo app for tracking workouts."*
- *"Add a settings screen with a dark-mode toggle."*
- *"Wire up authenticated data fetching for the feed."*
- *"My animations stopped working after upgrading Reanimated."*

The agent loads the skill, follows the architecture and config, and applies the guardrails. You can also nudge it explicitly: *"use the dephx-reactnative skill."*

---

## Scope

**Covers:** project scaffolding, stack selection, folder structure, navigation (expo-router), data fetching (TanStack Query), state (Zustand), storage (secure-store / MMKV), styling & theming (NativeWind + semantic tokens), testing (jest-expo), and Expo build/config troubleshooting.

**Does not cover:** app-store signing, submission, and release (keystores, EAS Submit, Play Console / App Store Connect). That's deliberately out of scope — this skill is about building the app well, not shipping it to the stores.

---

## Maintenance

The stack is pinned to a baseline (see above), dated **2026-07** at `v0.1.0`. React Native and Expo move fast; when a new SDK lands, refresh the version matrix in `references/config-reference.md` and re-verify the config quartet against the SDK's upgrade notes. The **architecture, patterns, and gotchas age far more slowly** than the version numbers — they're the durable part.

Issues and PRs welcome.

## License

MIT © 2026 Adedamola Daniel ([@ephraimd](https://github.com/ephraimd)) — see [LICENSE](LICENSE).

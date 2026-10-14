# dephx-reactnative

A portable **skill** that gives coding agents (Claude Code, OpenCode, and any tool that reads the [Agent Skills](https://agentskills.io/specification) format) a production-grade foundation for building **Expo / React Native** apps.

It's the distilled, de-branded playbook from a shipped app: the stack, the architecture, version-specific config examples, the data/state/storage/theming patterns, the testing doctrine, and the hard-won Expo build gotchas. Add it to a project and the agent scaffolds and extends a mobile app that is structured, testable, and guided by verified project contracts - for any app domain, not a specific one.

**Baseline:** Expo / React Native, strict TypeScript and one project-approved UI foundation. Resolve SDK/library versions against the consuming app; the original version recipe is historical, not a current verified bundle.

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
    config-reference.md        version selection + labelled config examples
    api-layer.md               injectable apiClient, query hooks, mutations, keys, type-gen
    state-storage.md           Zustand / secure-store / optional scoped persistence / query-persist / money
    design-system.md           semantic tokens, light/dark, component conventions
    testing.md                 logic, native UX and release-performance evidence
    gotchas.md                 mistakes-learned → DO-NOT / ALWAYS
```

The skill is pure knowledge - no dependencies, nothing to build. The agent reads it and applies it to your project.

---

## Install

### Claude Code - one-command (recommended)

```
/plugin marketplace add ephraimd/dephx-reactnative
/plugin install dephx-reactnative@dephx
```

The skill is discoverable for React Native / Expo app work only. It adds no global startup hook and excludes ordinary web/backend work, including websites viewed on phones. To update later, run `/plugin marketplace update dephx` and reinstall if prompted.

### Claude Code - manual copy

```bash
git clone https://github.com/ephraimd/dephx-reactnative.git
# personal (all projects):
cp -r dephx-reactnative/skills/dephx-reactnative ~/.claude/skills/
# or project-local:
cp -r dephx-reactnative/skills/dephx-reactnative <your-project>/.claude/skills/
```

### Codex

```sh
codex plugin marketplace add ephraimd/dephx-reactnative --ref main
codex plugin add dephx-reactnative@dephx
```

For updates, run `codex plugin marketplace upgrade dephx` then install again.
The description is discoverable globally; the instructions load for native app tasks.
It has no global startup hook.

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

**Covers:** project scaffolding, stack selection, folder structure, navigation (expo-router), data fetching (TanStack Query), state (Zustand), storage (secure-store / optional scoped persistence), styling & theming (NativeWind + semantic tokens), testing (jest-expo), and Expo build/config troubleshooting.

**Does not cover:** app-store signing, submission, and release (keystores, EAS Submit, Play Console / App Store Connect). That's deliberately out of scope - this skill is about building the app well, not shipping it to the stores.

---

## Maintenance

Guidance updated **2026-10-14** at `v0.2.0`. Version-sensitive examples are labelled and require validation against the selected SDK. React Native and Expo move fast; when a new SDK lands, resolve supported versions and re-verify configuration against the SDK's upgrade notes. The **architecture, patterns, and gotchas age far more slowly** than the version numbers - they're the durable part.

Issues and PRs welcome.

## License

MIT © 2026 Adedamola Daniel ([@ephraimd](https://github.com/ephraimd)) - see [LICENSE](LICENSE).

# Fivemanage AI Skills

This repository contains structured skill documents for AI coding assistants. Each skill defines rules, workflows, and reference material the AI should follow when helping FiveM server developers work with the [Fivemanage](https://fivemanage.com) SDK (`fmsdk`).

## Installation

These skills use the standard Agent Skills layout: `skills/<skill-name>/SKILL.md`. They can be installed with the [`skills`](https://www.npmjs.com/package/skills) CLI using `npx`.

List the available skills:

```bash
npx skills add fivemanage/skills --list
```

Install all skills into the current project:

```bash
npx skills add fivemanage/skills --skill "*"
```

Install one skill:

```bash
npx skills add fivemanage/skills --skill add-fmsdk-logging
```

Install globally for VS Code / GitHub Copilot:

```bash
npx skills add fivemanage/skills --skill "*" --agent github-copilot --global
```

Install globally for OpenCode:

```bash
npx skills add fivemanage/skills --skill "*" --agent opencode --global
```

Install globally for Claude Code:

```bash
npx skills add fivemanage/skills --skill "*" --agent claude-code --global
```

After installing, restart your AI coding assistant or open a new session so it can discover the new skills.

## Skills

### `fivem-resource-logging`
Teaches the AI how to choose the right FiveM logging strategy before editing code.

- Separates passive event listeners from direct instrumentation and patch-based logging
- Defines what can and cannot be observed from outside a resource
- Covers server/client boundaries, `fxmanifest.lua` dependencies, and safe metadata design

### `qbox-fmsdk-logging`
Teaches the AI how to add Fivemanage logging to Qbox/QBX resources, including resources under `[qbx]/`.

- Covers known Qbox patch points: `qbx_core`, `qbx_police`, `qbx_management`, `qbx_bankrobbery`
- Defines safe patching patterns for `lib.addCommand`, `lib.callback.register`, exports, and client-only admin actions
- Includes event maps, patch templates, and dashboard-oriented metadata guidance

### `add-fmsdk-logging`
Teaches the AI how to instrument a FiveM resource with Fivemanage cloud logging from scratch.

- Covers the full `fmsdk` export API (`Log`, `Info`, `Warn`, `Error`)
- Defines rules for structured logging, dataset selection, player identifiers, and manifest setup
- Includes a multi-step workflow: explore resource → plan datasets → confirm with user → write code → update manifest

### `migrate-to-fivemanage`
Teaches the AI how to migrate an existing FiveM resource from legacy logging to `fmsdk`.

Supported migration sources:
- **`ox_lib` logger** (`lib.logger`) — maps source/event/varargs to structured metadata
- **Discord webhook logging** — decomposes string-formatted messages into structured fmsdk calls

## Structure

```
skills/
├── fivem-resource-logging/
│   ├── SKILL.md
│   └── references/
│       └── passive-vs-patched.md
│
├── qbox-fmsdk-logging/
│   ├── SKILL.md
│   └── references/
│       ├── qbox-event-map.md
│       └── patching-qbox-resources.md
│
├── add-fmsdk-logging/
│   ├── SKILL.md              # Workflow, rules, and examples
│   └── references/
│       ├── sdk-api.md        # fmsdk export API reference
│       ├── best-practices.md # Structured logging best practices
│       └── config-examples.md
│
└── migrate-to-fivemanage/
    ├── SKILL.md              # Workflow, rules, and migration examples
    └── references/
        ├── best-practices.md
        └── config-examples.md
```

## Related

- [Fivemanage SDK](https://github.com/fivemanage/sdk)
- [Fivemanage Docs](https://docs.fivemanage.com)

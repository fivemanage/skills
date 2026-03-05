# Fivemanage AI Skills

This repository contains structured skill documents for AI coding assistants. Each skill defines rules, workflows, and reference material the AI should follow when helping FiveM server developers work with the [Fivemanage](https://fivemanage.com) SDK (`fmsdk`).

## Skills

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

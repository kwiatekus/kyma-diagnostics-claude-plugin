# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A Claude Code **plugin** for diagnosing SAP BTP Kyma instances. It is distributed through the `kwiatekus/claude-plugins` marketplace (registered in `~/.claude/settings.json` as `kyma-claude-plugins`).

It exposes two entry points:
- **`kyma-diagnoser` agent** — triggers automatically from natural-language prompts describing Kyma problems.
- **`/kyma-diagnose` skill** — user-invoked slash command for explicit diagnostic runs.

## Plugin structure

Claude Code plugins follow this layout:

```
plugin-root/
├── .claude-plugin/
│   └── plugin.json          # Plugin manifest (name, version, description, author, keywords)
├── agents/
│   └── <agent-name>.md      # One file per agent; YAML frontmatter + system prompt
├── skills/
│   └── <skill-name>/
│       └── SKILL.md         # One file per skill; YAML frontmatter + markdown instructions
└── README.md
```

- **`.claude-plugin/plugin.json`** — required manifest; must use kebab-case `name`.
- **`agents/<name>.md`** — agent definition. Claude spawns this automatically when the `description` matches the user's prompt.
- **`skills/<name>/SKILL.md`** — skill definition. Invoked explicitly by the user as a slash command or called by an agent.

### Agent frontmatter fields

| Field | Required | Purpose |
|---|---|---|
| `name` | yes | Agent identifier |
| `description` | yes | Trigger conditions — use `<example>` blocks for reliable matching |
| `model` | no | Override model (`haiku`, `sonnet`, `opus`) |
| `color` | no | Color shown in the UI |
| `tools` | no | List of tools the agent may use |

The `description` field drives automatic invocation. Use `<example>` blocks with `user:` / `assistant:` / `<commentary>` to show Claude exactly when to trigger the agent.

### SKILL.md frontmatter fields

| Field | Required | Purpose |
|---|---|---|
| `name` | yes | Skill identifier (matches directory name) |
| `description` | yes | Trigger phrases / conditions |
| `argument-hint` | no | Help text shown to users in `/help` |
| `allowed-tools` | no | Pre-approved tools; reduces permission prompts |

## How plugins are distributed

The marketplace is a GitHub repo (`kwiatekus/claude-plugins`) whose root contains one subdirectory per plugin. Each plugin subdirectory must contain `.claude-plugin/plugin.json` and the plugin's components.

Users install via:
```
/plugin marketplace add kwiatekus/claude-plugins
/plugin install kyma-diagnose
```

Claude Code fetches the plugin from GitHub and caches it under `~/.claude/plugins/cache/`.

## Developing and testing locally

Load the plugin directly without publishing:
```bash
claude --plugin-dir /path/to/this/repo
```

To exercise the skill interactively, open a Claude Code session with the plugin loaded and invoke:
```
/kyma-diagnose [optional module or area]
```

There are no build steps, test runners, or linters — the artefacts are Markdown and JSON files.

## Adding or updating a component

1. For a new agent: create `agents/<name>.md` with YAML frontmatter + system prompt.
2. For a new or updated skill: create or edit `skills/<skill-name>/SKILL.md`.
3. Bump `version` in `.claude-plugin/plugin.json` (semver).
4. Update `README.md` with usage instructions.
5. Push to the `main` branch of the GitHub repo that backs the marketplace — Claude Code installs from that branch.

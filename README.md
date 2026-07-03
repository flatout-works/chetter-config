# Chetter Config

Git-backed runtime configuration for Chetter. The MCP server syncs from this repository when `DEFINITIONS_REPO` points here.

**NOTE: This config repo backs the Chetter instance that works "on itself" used by the team developing Chetter**

## Structure

```
├── model-catalog.yaml      # AI model/provider registry
├── agents/                 # Agent definitions (*.md)
├── skills/                 # Skill definitions (SKILL.md under skill name directory)
├── triggers/               # Trigger definitions (*.yaml or *.yml)
└── task-templates/         # Reusable task prompt templates (*.md)
```

## How definitions are used

| Definition type | Synced to DB | Used at runtime |
|---|---|---|
| `model-catalog.yaml` | ✅ | Model/provider selection for tasks |
| `agents/*.md` | ✅ `definitions` table | Injected into runner container per task |
| `skills/*/SKILL.md` | ✅ `definitions` table | Injected into runner container per task |
| `triggers/*.yaml` | ✅ `chetter_triggers` table | Activated in the cron/webhook scheduler |
| `task-templates/*.md` | ✅ `definitions` table | *(stored, runtime usage pending)* |

## Security

Do not store secret values in this repository. Use environment variable names such as `api_key_env: ANTHROPIC_API_KEY`. The `api_key_env` value in `model-catalog.yaml` tells Chetter which environment variable to read from — the actual key stays in your deployment environment.

## Validation

Chetter validates synced definition files before materializing them. Schema references in this repository point to the adjacent Chetter checkout:

| File | Schema |
|---|---|
| `model-catalog.yaml` | `../chetter/schemas/model-catalog.schema.json` |
| `triggers/*.yaml` | `../../chetter/schemas/trigger.schema.json` |
| Agent YAML frontmatter in `agents/*.md` | `../../chetter/schemas/agent-frontmatter.schema.json` |

Validation failures reject the sync, leaving the previously active definitions in place.

## Example repo

See `examples/config-repo/` in the main Chetter repository for a starter template.

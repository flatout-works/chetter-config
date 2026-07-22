# Chetter Config

Git-backed runtime configuration for Chetter. The MCP server syncs from this repository when `DEFINITIONS_REPO` points here.

**Note:** This repository backs the Chetter instance used by the team developing Chetter itself.

## Structure

```
├── model-catalog.yaml            # AI model/provider registry
├── global/
│   ├── agents/                   # Global agent definitions (*.md)
│   ├── skills/                   # Global skill definitions (SKILL.md under skill name directory)
│   ├── triggers/                 # Global trigger definitions (*.yaml)
│   ├── mcp-endpoints/            # Global MCP endpoint definitions (*.yaml)
│   ├── task-templates/           # Global reusable task prompt templates
│   └── images/                   # Global agent dev container Dockerfiles
│       ├── golang/Dockerfile
│       ├── python/Dockerfile
│       ├── node/Dockerfile
│       ├── rust/Dockerfile
│       ├── minimal/Dockerfile
│       └── java-spring/Dockerfile
├── groups/
│   └── <team-name>/
│       ├── agents/               # Team-scoped agent definitions
│       ├── skills/               # Team-scoped skill definitions
│       ├── triggers/             # Team-scoped trigger definitions
│       ├── mcp-endpoints/        # Team-scoped MCP endpoint definitions
│       └── task-templates/       # Team-scoped task prompt templates
├── repos/
│   └── <owner>/<repo>/
│       ├── agents/               # Repo-scoped agent definitions
│       ├── skills/               # Repo-scoped skill definitions
│       ├── triggers/             # Repo-scoped trigger definitions
│       └── task-templates/       # Repo-scoped task prompt templates
```

## How definitions are used

| Definition type | Synced to DB | Used at runtime |
|---|---|---|
| `model-catalog.yaml` | ✅ | Model/provider selection for tasks |
| `global/agents/*.md` | ✅ `definitions` table (scope=global) | Injected into runner container per task |
| `global/skills/*/SKILL.md` | ✅ `definitions` table (scope=global) | Injected into runner container per task |
| `global/triggers/*.yaml` | ✅ `chetter_triggers` table (no team) | Activated in the cron/webhook scheduler |
| `global/mcp-endpoints/*.yaml` | ✅ `definitions` table (scope=global) | Mounted into tasks through agent frontmatter or task options |
| `groups/<team>/triggers/*.yaml` | ✅ `chetter_triggers` table (team-owned) | Activated with team_id set for scoped access |
| `repos/<owner>/<repo>/triggers/*.yaml` | ✅ `chetter_triggers` table (repo-scoped) | Activated with repo metadata for filtering |
| Scoped `task-templates/*.md` | `definitions` table | Reusable prompt definitions exposed through the definitions API |

Team-scoped triggers (`groups/<team>/triggers/*.yaml`) are materialized with the matching team's `team_id`, so only members of that team see them in their filtered views during task submission.

## Agent dev container images

The `global/images/` directory holds Dockerfiles for stack-specific agent runtime images.
Each one inherits from `ghcr.io/flatout-works/chetter-agent-base:main` which provides
the shared harness CLIs (opencode, claude-code, codewhale, pi) and common tooling.

Teammates pick an image via the `agent_image` field when submitting a task or in
a trigger definition. To add a new variant, create a new directory with a `Dockerfile`
that starts with `FROM ghcr.io/flatout-works/chetter-agent-base:main` and adds
the language/toolchain packages you need.

### Available images

| Tag | Contents |
|---|---|
| `golang` | Go 1.26, buf, sqlc, goose, govulncheck, osv-scanner, hcloud, MySQL client |
| `python` | Python 3, pip, venv, ruff, mypy, pytest, black, httpx |
| `node` | Node 22, pnpm, TypeScript, ts-node, eslint, prettier |
| `rust` | rustup, cargo, clippy, rustfmt, cargo-audit, build-essential, libssl |
| `minimal` | Base harnesses only — no language toolchain |
| `java-spring` | JDK 21, Maven, Gradle, Liquibase, PostgreSQL client |

### CI

A GitHub Actions workflow (`.github/workflows/build-agent-images.yml`) builds and
pushes all variant images to `ghcr.io/flatout-works/chetter-agent:$variant` on
every push to `main` that changes `global/images/**`. Each build also gets a
`:$variant-$sha` tag for rollbacks.

Use these agent images for tasks. Do not use `chetter-runner`, which is the
tight fleet daemon image and does not contain task harnesses.

## Instance prerequisites

- Create the `Chetter Core` team before syncing because the PR review trigger is team-scoped under `groups/Chetter Core/`.
- Create a global or team-applicable managed Git identity named `primary-bot`; all agent definitions reference it.
- Provide the credential environment variables named by `model-catalog.yaml` to applicable runners.
- Configure the GitHub App before enabling issue and PR triggers.
- Configure Arcane if image vulnerability scanning is required. The vulnerability workflow continues with dependency scanning when Arcane tools are unavailable.
- Provide any `auth.token_env` used by MCP endpoint definitions to applicable runners.

## Security

Do not store secret values in this repository. Use environment variable names such as `api_key_env: ANTHROPIC_API_KEY`. The `api_key_env` value in `model-catalog.yaml` tells Chetter which environment variable to read from — the actual key stays in your deployment environment.

Every agent definition must declare an `identity` in its YAML frontmatter. Identity credentials are managed server-side and must not be added to this repository.

## Validation

Chetter validates all synced definition files before materializing them. Most schema references in this repository point to an adjacent Chetter checkout for local editor validation:

| File | Schema |
|---|---|
| `model-catalog.yaml` | `../chetter/schemas/model-catalog.schema.json` |
| `global/triggers/*.yaml` | `../../../chetter/schemas/trigger.schema.json` |
| `global/mcp-endpoints/*.yaml` | `../../../chetter/schemas/mcp-endpoint.schema.json` |
| `groups/<team>/triggers/*.yaml` | `../../../../chetter/schemas/trigger.schema.json` |
| `repos/<owner>/<repo>/triggers/*.yaml` | `../../../../../chetter/schemas/trigger.schema.json` |
| Agent YAML frontmatter in `global/agents/*.md` | `../../../chetter/schemas/agent-frontmatter.schema.json` |

Validation failures reject the sync, leaving the previously active definitions in place.

Trigger names are instance-wide identifiers, not scope-qualified identifiers. Do not define the same trigger name in global, team, and repository scopes; a later materialized definition would replace the earlier one.

Referenced agents, skills, MCP endpoints, teams, and managed Git identities must exist in an applicable scope. Missing skills are not installed automatically.

## Example repo

See `examples/config-repo/` in the main Chetter repository for a starter template.

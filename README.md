# Chetter Config

Git-backed runtime configuration for Chetter. The MCP server syncs from this repository when `DEFINITIONS_REPO` points here.

**NOTE: This config repo backs the Chetter instance that works "on itself" used by the team developing Chetter**

## Structure

```
├── model-catalog.yaml      # AI model/provider registry
├── agents/                 # Agent definitions (*.md)
├── skills/                 # Skill definitions (SKILL.md under skill name directory)
├── triggers/               # Trigger definitions (*.yaml or *.yml)
├── task-templates/         # Reusable task prompt templates (*.md)
└── images/                 # Agent dev container images (see below)
    ├── golang/Dockerfile
    ├── python/Dockerfile
    ├── node/Dockerfile
    ├── rust/Dockerfile
    ├── minimal/Dockerfile
    └── java-spring/Dockerfile
```

## How definitions are used

| Definition type | Synced to DB | Used at runtime |
|---|---|---|
| `model-catalog.yaml` | ✅ | Model/provider selection for tasks |
| `agents/*.md` | ✅ `definitions` table | Injected into runner container per task |
| `skills/*/SKILL.md` | ✅ `definitions` table | Injected into runner container per task |
| `triggers/*.yaml` | ✅ `chetter_triggers` table | Activated in the cron/webhook scheduler |
| `task-templates/*.md` | ✅ `definitions` table | *(stored, runtime usage pending)* |

## Agent dev container images

The `images/` directory holds Dockerfiles for stack-specific agent runtime images.
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
every push to `main` that changes `images/**`. Each push gets both a `:main` tag
and a `:$variant-$sha` digest pin for reproducible rollbacks.

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

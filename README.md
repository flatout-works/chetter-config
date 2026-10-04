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
│       ├── nim/Dockerfile
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
Each variant inherits from `ghcr.io/flatout-works/chetter-agent-base:main`, which provides
all harnesses (opencode, claude-code, codewhale, pi, codex, niffler) and common tooling.

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
| `nim` | Nim 2.2.8, Nimble, Testament, Go 1.26.4, GCC/G++, Clang/libclang, GTK4, SQLite, libsodium, LZ4, MariaDB client headers, OpenSSL |
| `minimal` | All harnesses and their runtime/self-extension toolchains; no additional stack-specific tooling |
| `java-spring` | JDK 21, Maven, Gradle, Liquibase, PostgreSQL client |

### Niffler tasks

Select `harness: niffler` with any agent image. The shared base includes the
complete pinned Niffler distribution and `/usr/local/bin/niffler-serve-proxy`,
alongside the other harnesses; no dedicated variant is required. Chetter uses
this revision's native `cli run` driver for turns, MCP bootstrap, cancellation
and canonical export, and deduplicates its authoritative usage by turn ID.
Rebuild the base and downstream variants before selecting Niffler in a deployed fleet.

The base Dockerfile in the Chetter repository pins Niffler to commit
`3f7684bd0dab0b29f59318ad7fe281ea51578bd9`. `/opt/niffler/REVISION` records
that pin. The read-only `/opt/niffler` template includes prebuilt core,
session runner, components, NATS server, bootstrap manifest, SDKs, sources,
skills and documentation. It is assembled from `git archive` plus freshly
built `var/bin`, never from a developer's working directory: no `.env`,
store database, runtime bus address, or plugin state is included. The serve
proxy must create a writable per-task `.niffler` runtime root and symlink
immutable assets/binaries from the template. The image ships Nim packages at
`/opt/niffler/nimble-pkgs2`; runtime setup must link `$HOME/.nimble/pkgs2`
there so the pinned `config.nims` can resolve packages even when Chetter
sets `HOME=/workspace`. Never boot a shared store directly under `/opt/niffler`.
Keep component
binaries off `PATH` (their names include `git`, `grep` and `bash`).

The catalog default is `deepseek/deepseek-v4-flash`, using `DEEPSEEK_API_KEY`
from the runner environment. No API key is required to build or smoke-test
the image. Update the Dockerfile's `NIF_REV` and this README together when
changing the pin; a build fails if the fetched commit does not match it.

The base supplies Nim 2.2.12 and Go for Niffler's self-extension tools. Stack
variants may select their own development toolchain versions.

### CI

A GitHub Actions workflow (`.github/workflows/build-agent-images.yml`) builds and
pushes all variant images to `ghcr.io/flatout-works/chetter-agent:$variant` on
every push to `main` that changes `global/images/**`. Each build also gets a
`:$variant-$sha` tag for rollbacks.

Use these agent images for tasks. Do not use `chetter-runner`, which is the
tight fleet daemon image and does not contain task harnesses.

## Documentation self-merge

`chetter-nightly-docs-update` authorizes the docs-maintainer to merge only the
prose-only PR it creates in that activation, using the audited runner-bridge
`chetter_merge_pr {repo, pr_number, merge_method}`. No new MCP tool is needed:
Chetter injects the task/execution/claim and checks GitHub App authorization.
This is a prompt-scoped policy, not a general server-side docs permission gate.
The GitHub App must have merge permissions and branch protection still applies.

The agent reviews the complete diff, local checks, CI/review state and head/base
before merging. It never bypasses protections, merges other authors' PRs, or
changes code/config/workflows/security policy under this authorization. Pending
or failed checks, conflicts or a refused merge leave the PR open for a human.
Other docs/changelog/website triggers remain PR-only unless explicitly enabled.

## Instance prerequisites

- Create the `Chetter Core` team before syncing because the PR review trigger is team-scoped under `groups/Chetter Core/`.
- Create a global or team-applicable managed Git identity named `primary-bot`; all agent definitions reference it.
- Provide the credential environment variables named by `model-catalog.yaml` to applicable runners.
- Configure one GitHub App and install that same App on every organization or
  user account whose repositories use issue or PR triggers. Chetter selects the
  signed webhook installation and brokers short-lived repository-scoped task
  credentials; definitions must not promise or store `GITHUB_TOKEN`.
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

## Configured repositories

| Repository | Scope | Automation |
|---|---|---|
| `flatout-works/chetter` | global + `repos/` + `groups/Chetter Core/` | issue triage, implementers (approved / bug / six-hour), PR review, nightlies (changelog, docs, website, stale check, vulnerability scan), feature creator, weekly task improvers |
| `flatout-works/chetter-config` | `repos/` | PR review |
| `gokr/buddydrive` | `repos/` | issue triage, implementer, PR review, nightly changelog |
| `gokr/buddydrive-relay` | `repos/` | issue triage, implementer, PR review |
| `gokr/niffler` | `repos/` | issue triage, implementers (approved / bug / six-hour), PR review, nightlies (changelog, docs, website, stale check), feature creator |

# Changelog

All notable changes to this project will be documented in this file.

## [0.1.0] - 2026-08-06

First tagged release of Chetter: a self-hosted MCP server and runner fleet for
autonomous AI development tasks.

### Added

- **Tasks & sessions**: submit, track, and resume AI development tasks through MCP tools
  (`chetter_submit_task`, `chetter_recover_task`) or the web UI; resumable agent
  sessions (harness sessions or gVisor checkpoints) with task recovery from session exports.
- **Runner fleet**: Docker and Kubernetes execution with gVisor/runsc sandboxing,
  enforced isolation admission, per-task memory/CPU limits, heartbeat-based fleet
  health, draining, and self-test profiles (`chetter_run_self_test`).
- **Harnesses**: OpenCode, Claude Code, Pi, CodeWhale, and Codex execution with an
  in-session runner bridge exposing task artifacts and GitHub tools.
- **Triggers**: GitHub webhook triggers (push, PR review, issues, comments), cron
  triggers, and manual test runs for external-event triggers.
- **Web UI**: task dashboard, fleet view, trigger management, audit log, diagnostics,
  settings, and OIDC/OAuth SSO (Okta) with admin/team-group mapping.
- **Databases**: MySQL, TiDB, and PostgreSQL support via goose migrations.
- **Operations**: `/healthz`, `/readyz`, and `/api/server-info` endpoints; server
  version (`x.y.z`), git hash, and uptime shown in the UI footer.

### Changed

- Database tables renamed from `chetter_*` prefixes to unprefixed names (migration 052).
- Claiming optimized with an in-process notifier, cutting idle DB polling ~20x.

### Fixed

- Runner claiming on TiDB serverless (`SKIP LOCKED` planner bug) via a `NOT EXISTS`
  rewrite.
- Session timezone bug: TiDB session time zone must be UTC or fleet presence expires.

Detailed per-day history of everything that went into this release is below.

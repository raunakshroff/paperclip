# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Paperclip is an open-source orchestration control plane for AI agent companies. It lets operators define company goals, hire agents (Claude Code, Codex, Cursor, OpenClaw, Gemini, Grok, OpenCode, Pi, etc.), assign tasks, and track costs — all from one dashboard. The server orchestrates agents; it does not run them directly.

Key reference docs (read before major changes):
- `doc/GOAL.md` — product vision
- `doc/SPEC-implementation.md` — V1 build contract (takes precedence over `doc/SPEC.md` for V1)
- `doc/DEVELOPING.md` — full development guide (worktrees, secrets, Docker, CLI)
- `doc/DATABASE.md` — database modes and schema workflow
- `doc/CLI.md` — CLI client command reference
- `adapter-plugin.md` — external adapter plugin system

## Commands

```sh
pnpm install          # Install dependencies
pnpm dev              # Start API + UI in watch mode (port 3100)
pnpm dev:once         # Start without file watching (auto-applies pending migrations)
pnpm dev:list         # List managed dev runners
pnpm dev:stop         # Stop managed dev runner
pnpm storybook        # UI Storybook on port 6006

pnpm test             # Vitest suite only (default, cheap)
pnpm test:watch       # Vitest interactive watch mode
pnpm test:e2e         # Playwright browser suite (opt-in only)
pnpm test:release-smoke  # Release smoke tests (opt-in)

pnpm -r typecheck     # Full workspace typecheck
pnpm build            # Build all packages

pnpm db:generate      # Compile DB package and generate Drizzle migration
pnpm db:migrate       # Apply pending migrations
pnpm db:backup        # Manual DB backup

pnpm paperclipai issue list --company-id <id>   # CLI client example
pnpm paperclipai context set --api-base http://localhost:3100 --company-id <id>

docs:dev              # Mintlify docs dev server (cd docs && npx mintlify dev)
```

**PR-ready handoff check** (run before claiming work done on a broad change):
```sh
pnpm -r typecheck && pnpm test:run && pnpm build
```

**Reset local dev DB:**
```sh
rm -rf ~/.paperclip/instances/default/db
pnpm dev
```

**One-command local install:**
```sh
pnpm paperclipai run   # auto-onboard + doctor + start server
```

## Architecture

### Monorepo Layout

| Path | Purpose |
|---|---|
| `server/` | Express REST API + all orchestration services |
| `ui/` | React + Vite board UI |
| `packages/db/` | Drizzle ORM schema (`src/schema/*.ts`), migrations, DB clients |
| `packages/shared/` | Shared types, constants, validators, API path constants |
| `packages/adapters/` | Built-in agent adapter implementations (10 adapters) |
| `packages/adapter-utils/` | Shared adapter utilities |
| `packages/mcp-server/` | MCP server for control-plane tool access |
| `packages/plugins/` | Plugin system (SDK, runtime, examples) |
| `cli/` | `paperclipai` CLI (setup + client-side control-plane commands) |
| `skills/` | Built-in skills delivered to agents (markdown skill files) |
| `evals/` | Evaluation framework (promptfoo) |
| `tests/` | E2E (Playwright) and release-smoke suites |
| `docs/` | Mintlify documentation site |
| `doc/` | Operational and product docs |
| `doc/plans/` | Implementation plan docs (`YYYY-MM-DD-slug.md`) |

### Server

`server/src/` contains:
- `routes/` — Express route handlers (36 files). Key domains: `agents.ts`, `issues.ts`, `heartbeat-runs.ts`, `projects.ts`, `goals.ts`, `routines.ts`, `environments.ts`, `execution-workspaces.ts`, `secrets.ts`, `plugins.ts`, `adapters.ts`, `costs.ts`, `approvals.ts`
- `services/` — Business logic services (110 files). Key ones: `heartbeat.ts`, `issues.ts`, `budgets.ts`, `workspace-runtime.ts`, `plugin-lifecycle.ts`, `environments.ts`, `routines.ts`, `projects.ts`, `agents.ts`, `secrets.ts`, `company-portability.ts`
- `adapters/` — Server-side adapter registry (mutable); adapters can be built-in or dynamically loaded via the plugin system
- `middleware/` — Auth, board-mutation guard, private-hostname guard, error handler, logger
- `realtime/` — Live event streaming (SSE/WebSocket)
- `secrets/` — Secret provider implementations (`local-encrypted`, `aws-secrets-manager`, external stubs)
- `storage/` — Storage providers (`local-disk`, `s3`)

### UI

`ui/src/` contains:
- `pages/` — Route-level page components (60+ files)
- `components/` — Shared UI components (150+ files including `transcript/` for run views)
- `api/` — Typed API clients per domain
- `adapters/` — UI-side adapter registry (parallel to server); includes `registry.ts`, `metadata.ts`, `dynamic-loader.ts`
- `context/` — React context (`CompanyContext`, `LiveUpdatesProvider`, `ThemeContext`, etc.)
- `hooks/` — Custom React hooks
- `i18n/` — Internationalization (i18next)
- `plugins/` — UI plugin support

### Database

Drizzle ORM on PostgreSQL. Schema lives in `packages/db/src/schema/*.ts` — one file per table group. Tables are exported from `packages/db/src/schema/index.ts`.

**DB change workflow:**
1. Edit `packages/db/src/schema/*.ts`
2. Export new tables from `packages/db/src/schema/index.ts`
3. `pnpm db:generate` (compiles the DB package first, then generates the migration)
4. `pnpm -r typecheck`

In dev, leave `DATABASE_URL` unset — the server auto-starts embedded PostgreSQL at `~/.paperclip/instances/default/db/`. Set `DATABASE_URL` to use external Postgres. `DATABASE_MIGRATION_URL` can be set separately for migration-only connections (useful with connection poolers).

### Adapters

Built-in adapters live in `packages/adapters/<name>/`:

| Adapter | Runtime |
|---|---|
| `claude-local` | Claude Code CLI (local) |
| `codex-local` | Codex CLI (local) |
| `gemini-local` | Gemini CLI (local) |
| `grok-local` | Grok (local) |
| `opencode-local` | OpenCode CLI (local) |
| `cursor-local` | Cursor (local) |
| `cursor-cloud` | Cursor (cloud) |
| `openclaw-gateway` | OpenClaw remote gateway |
| `acpx-local` | ACPX (local) |
| `pi-local` | Pi (local) |

External adapters are loaded dynamically through the adapter plugin system (`adapter-plugin.md`). No hardcoded imports of external adapters in `server/` or `ui/`.

### Plugin System

Plugins extend Paperclip with new adapters, UI pages, tools, jobs, and database namespaces. The SDK lives in `packages/plugins/sdk/`. Plugin examples are in `packages/plugins/examples/`. The plugin runtime loads, sandboxes, and manages plugin lifecycle in `server/src/services/plugin-*.ts`.

### Skills

`skills/` contains built-in skill markdown files delivered to agents via `GET /api/skills/`:
- `paperclip` — Paperclip heartbeat protocol
- `paperclip-dev` — Development workflow
- `paperclip-create-agent`, `paperclip-create-plugin` — Scaffolding helpers
- `paperclip-converting-plans-to-tasks` — Task decomposition
- `diagnose-why-work-stopped` — Debugging stopped work
- `para-memory-files`, `terminal-bench-loop` — Supporting skills

### MCP Server

`packages/mcp-server/` exposes control-plane tools (issues, tasks, agents, companies) over the MCP protocol for use in compatible AI environments.

## Core Engineering Rules

**Company-scoped entities.** Every domain entity is scoped to a company. All routes/services must enforce company boundaries.

**Contract synchronization.** When changing schema or API behavior, update all four layers in lockstep: `packages/db` → `packages/shared` → `server` → `ui`.

**Control-plane invariants** — never break these:
- Single-assignee task model
- Atomic issue checkout (execution lock on `in_progress` transition)
- Approval gates for governed actions (hires, strategy)
- Budget hard-stop auto-pause
- Activity log entries for all mutating actions

**API conventions.** Base path: `/api`. Board users get full-control operator access. Agents use bearer API keys from `agent_api_keys` (hashed at rest). Every endpoint must: apply company access checks, enforce board-vs-agent permissions, write activity log entries for mutations, and return consistent HTTP error codes (`400/401/403/404/409/422/500`).

**Adapter isolation.** External adapters must never be imported directly in `server/` or `ui/`. Use the plugin/dynamic-loader path.

**Secrets.** Sensitive env values must use secret references, not inline plaintext, in authenticated deployments. `PAPERCLIP_SECRETS_STRICT_MODE=true` enforces this.

## Test Strategy

- `pnpm test` — default; runs Vitest suite across all packages. Use for normal issue work.
- `pnpm test:run:general` / `pnpm test:run:serialized` — targeted vitest modes for specific test classes.
- Browser suites (`test:e2e`, `test:release-smoke`) — opt-in; only run when touching those flows.
- Start with the narrowest check that proves the change. Reserve full typecheck+test+build for PR-ready handoff.

Vitest projects: `packages/shared`, `packages/db`, `packages/adapter-utils`, all adapter packages, `packages/plugins/sdk`, `server`, `ui`, `cli`.

## Pull Request Requirements

Every PR must use `.github/PULL_REQUEST_TEMPLATE.md`. Required sections:
- **Thinking Path** — trace from project context down to this change (5–8 steps, see `CONTRIBUTING.md` for examples)
- **What Changed** — concrete bullet list
- **Verification** — how a reviewer confirms it works; include screenshots for UI changes
- **Risks** — what could go wrong
- **Model Used** — provider, exact model ID, context window, capabilities (or "None — human-authored")
- **Checklist** — all items checked (including Greptile 5/5 score)

Do not commit `pnpm-lock.yaml` in pull requests (CI owns it).

## Worktree Development

For multi-worktree development, each worktree needs its own isolated Paperclip instance:

```sh
pnpm paperclipai worktree:make <name>   # create worktree + isolated instance
pnpm paperclipai worktree init          # init isolation in existing worktree
pnpm paperclipai worktree repair        # repair metadata or reseed DB
pnpm paperclipai worktree reseed        # reseed DB while preserving config
pnpm paperclipai worktree env           # print shell exports for the worktree
```

`pnpm dev` fails fast in a linked worktree if `.paperclip/.env` is missing — run `worktree init` first. Worktrees auto-get a colored banner, scoped favicon, and isolated auth cookies. Seeded worktrees quarantine copied live execution (running agents/in-progress issues) by default.

## Fork-Specific Notes (This Repo)

This is a fork of `paperclipai/paperclip` with QoL patches and an externalized Hermes adapter story.

**NTFS / Windows quirks:**
- `npx vite build` hangs on NTFS — use `node node_modules/vite/bin/vite.js build` instead
- Server startup from NTFS can take 30–60s — do not assume failure immediately
- Vite cache survives `rm -rf dist` — delete both `ui/dist` and `ui/node_modules/.vite` to fully clear

**Process hygiene:** Kill all Paperclip processes before starting a new instance. The fork auto-detects port conflicts and runs on 3101+ when 3100 is taken by an upstream instance.

**Fork QoL patches in `ui/`** (must be re-applied if re-copying source):
1. `stderr_group` — amber accordion for MCP init noise in `RunTranscriptView.tsx`
2. `tool_group` — accordion for consecutive non-terminal tools
3. `Dashboard excerpt` — `LatestRunCard` strips markdown, shows first 3 lines/280 chars

**Hermes adapter** — registered through Board → Adapter manager (type `hermes_local`). No imports of Hermes in `server/` or `ui/` source — pure dynamic loading via the adapter plugin system.

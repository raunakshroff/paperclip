# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

Paperclip is an open-source orchestration control plane for AI agent companies. It lets operators define company goals, hire agents (Claude Code, Codex, Cursor, OpenClaw, etc.), assign tasks, and track costs — all from one dashboard. The server orchestrates agents; it does not run them directly.

Key reference docs (read before major changes):
- `doc/GOAL.md` — product vision
- `doc/SPEC-implementation.md` — V1 build contract (takes precedence over `doc/SPEC.md` for V1)
- `doc/DEVELOPING.md` — full development guide
- `doc/DATABASE.md` — database modes and schema workflow

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

pnpm -r typecheck     # Full workspace typecheck
pnpm build            # Build all packages

pnpm db:generate      # Compile DB package and generate Drizzle migration
pnpm db:migrate       # Apply pending migrations
pnpm db:backup        # Manual DB backup
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

## Architecture

### Monorepo Layout

| Path | Purpose |
|---|---|
| `server/` | Express REST API + all orchestration services |
| `ui/` | React + Vite board UI |
| `packages/db/` | Drizzle ORM schema (`src/schema/*.ts`), migrations, DB clients |
| `packages/shared/` | Shared types, constants, validators, API path constants |
| `packages/adapters/` | Agent adapter implementations (`claude-local`, `codex-local`, `grok-local`, `openclaw-gateway`, etc.) |
| `packages/adapter-utils/` | Shared adapter utilities |
| `packages/plugins/` | Plugin system (SDK, runtime, examples) |
| `cli/` | `paperclipai` CLI |
| `doc/` | Operational and product docs |
| `doc/plans/` | Implementation plan docs (`YYYY-MM-DD-slug.md`) |

### Server

`server/src/` contains:
- `routes/` — Express route handlers (one file per domain: `agents.ts`, `issues.ts`, `heartbeat-runs.ts`, etc.)
- `services/` — Business logic services. Key ones: `heartbeat.ts`, `issues.ts`, `budgets.ts`, `workspace-runtime.ts`, `plugin-lifecycle.ts`
- `adapters/` — Server-side adapter registry; adapters can be built-in or dynamically loaded via the plugin system
- `middleware/` — Auth middleware, company access checks
- `realtime/` — Live event streaming

### UI

`ui/src/` contains:
- `pages/` — Route-level page components
- `components/` — Shared UI components
- `api/` — Typed API clients
- `adapters/` — UI-side adapter registry (parallel to server registry)
- `context/` — React context (company selection, auth, etc.)

### Database

Drizzle ORM on PostgreSQL. Schema lives in `packages/db/src/schema/*.ts` — one file per table group. Tables are exported from `packages/db/src/schema/index.ts`.

**DB change workflow:**
1. Edit `packages/db/src/schema/*.ts`
2. Export new tables from `packages/db/src/schema/index.ts`
3. `pnpm db:generate` (compiles the DB package first, then generates the migration)
4. `pnpm -r typecheck`

In dev, leave `DATABASE_URL` unset — the server auto-starts embedded PostgreSQL at `~/.paperclip/instances/default/db/`. Set `DATABASE_URL` to use external Postgres.

### Adapters

Each adapter (`packages/adapters/<name>/`) defines how Paperclip invokes, monitors, and cancels a heartbeat for a specific agent runtime (Claude Code, Codex, Cursor, Gemini, Grok, OpenClaw, etc.). External adapters can be loaded dynamically through the adapter plugin flow without hardcoded imports in `server/` or `ui/`.

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

## Test Strategy

- `pnpm test` — default; runs Vitest suite only. Use for normal issue work.
- Browser suites (`test:e2e`, `test:release-smoke`) — opt-in; only run when touching those flows.
- Start with the narrowest check that proves the change. Reserve full typecheck+test+build for PR-ready handoff.

## Pull Request Requirements

Every PR must use `.github/PULL_REQUEST_TEMPLATE.md`. Required sections:
- **Thinking Path** — trace from project context down to this change (5–8 steps, see `CONTRIBUTING.md` for examples)
- **What Changed** — concrete bullet list
- **Verification** — how a reviewer confirms it works; include screenshots for UI changes
- **Risks** — what could go wrong
- **Model Used** — provider, exact model ID, context window, capabilities (or "None — human-authored")
- **Checklist** — all items checked

Do not commit `pnpm-lock.yaml` in pull requests (CI owns it).

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

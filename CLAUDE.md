# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

OpenClaw is a self-hosted personal AI assistant. Multi-channel gateway (WhatsApp, Telegram, Slack, Discord, 20+ more), extensible plugin system, runs on macOS/Linux/Windows(WSL2). TypeScript ESM monorepo managed with pnpm.

Detailed project policies live in `AGENTS.md` (root) and scoped `AGENTS.md` files under subdirectories. Read the relevant scoped guide before working in a subtree.

## Commands

| Task | Command |
|------|---------|
| Install | `pnpm install` |
| Build | `pnpm build` |
| Dev server | `pnpm dev` or `pnpm openclaw ...` |
| Test all | `pnpm test` |
| Test specific file | `pnpm test <path-or-filter> [vitest args...]` |
| Test changed files | `pnpm test:changed` |
| Test extensions | `pnpm test:extensions` or `pnpm test extensions/<id>` |
| Serial tests | `pnpm test:serial` |
| Coverage | `pnpm test:coverage` |
| Live model tests | `OPENCLAW_LIVE_TEST=1 pnpm test:live` |
| Full checks | `pnpm check` |
| Changed checks | `pnpm check:changed` |
| Staged checks | `pnpm check:changed --staged` |
| Typecheck | `pnpm tsgo` (core), `pnpm tsgo:prod` (core+ext), `pnpm check:test-types` (tests) |
| Lint | `pnpm lint` (oxlint via shards) |
| Format | `pnpm format` (oxfmt, NOT Prettier) |
| Import cycle check | `pnpm check:import-cycles` |
| Commit | `scripts/committer "<msg>" <file...>` |

**Important**: Never use raw `vitest` or `tsc --noEmit`. Typechecking uses `tsgo` only.

### Codex/sparse worktrees

Avoid direct `pnpm test*`/`pnpm check*` (triggers dependency reconciliation). Use:
- Tests: `node scripts/run-vitest.mjs <path-or-filter>`
- Checks: `node scripts/crabbox-wrapper.mjs run ... --shell -- "pnpm check:changed"`

## Architecture

```
src/                    Core TypeScript (67 subsystems)
  cli/                  Commander-based CLI framework
  commands/             CLI command implementations
  gateway/              Control plane, protocol versioning
  channels/             Channel abstraction layer
  agents/               AI agent system
  plugin-sdk/           Public plugin API surface
  plugins/              Plugin loader/registry
  config/               Configuration management
  secrets/              Credential storage
  sessions/             Session management
  tools/                Tool definitions
  mcp/                  Model Context Protocol
  acp/                  Agent Client Protocol
  infra/                Shared infrastructure (env, errors, process)
  bootstrap/            Startup/initialization
  entry.ts              CLI entry point
  library.ts            Programmatic API exports
  extensionAPI.ts       Extension/plugin API boundary
extensions/            128 plugin packages (channels, providers, skills)
  telegram/ slack/ discord/ whatsapp/ ...   Channel plugins
  anthropic/ openai/ google/ ...            Provider plugins
packages/              Shared libraries
  sdk/                  @openclaw/sdk
  plugin-sdk/           Plugin SDK types and runtime
  plugin-package-contract/  Plugin packaging contract
  memory-host-sdk/     Memory host SDK
ui/                    Web UI (Vite + Lit + Vitest)
apps/                  Native apps (android, ios, macos)
docs/                  Documentation
test/                  Test infrastructure, vitest configs
scripts/               Build/CI/dev scripts
```

### Key boundaries

- **Core is plugin-agnostic**: no bundled ids/defaults/policy in core. Manifests, registries, and capability contracts drive behavior.
- **Plugin isolation**: plugins import from `openclaw/plugin-sdk/*` only. No direct imports into `src/**`, `src/plugin-sdk-internal/**`, or other plugin source.
- **Core/tests**: no deep plugin internals (`extensions/*/src/**`). Use public barrels, SDK facade, generic contracts.
- **Owner boundary**: owner-specific repair/detection/onboarding/auth/defaults lives in the owner plugin. Core gets generic seams only.
- **Dependency ownership**: plugin-only deps stay plugin-local. Root deps only for core imports or intentionally internalized bundled plugin runtime.
- **Channels** (`src/channels/**`) are implementation. Plugin authors get SDK seams only.
- **Lazy boundaries**: use `*.runtime.ts` files for dynamic imports. No static+dynamic import for the same prod module.

### Hot paths

Carry prepared facts forward (provider id, model ref, channel id, target, capability family, attachment class). Do not rediscover with broad loaders at request time. Do not fix repeated discovery with scattered caches — move the canonical fact earlier.

## Code Conventions

- TypeScript ESM, strict mode. Avoid `any`; use `unknown`, real types, narrow adapters.
- No `@ts-nocheck`. Lint suppressions must be intentional and explained.
- External boundaries: validate with `zod` or existing schema helpers.
- Runtime branching: discriminated unions/closed codes over freeform strings.
- Keep files under ~700 LOC when splitting improves clarity.
- Product naming: **OpenClaw** in docs/UI; `openclaw` in CLI/package/path/config.
- Import convention: `.js` extension for cross-package ESM imports. `import type` for type-only imports.
- Reuse existing utilities. Search before creating formatters/helpers (e.g., time formatting in `src/infra/`, tables in `src/terminal/table.ts`, themes in `src/terminal/theme.ts`).

## Testing

- Vitest. Colocated `*.test.ts`; e2e: `*.e2e.test.ts`.
- Example model names: `sonnet-4.6`, `gpt-5.5` (GPT: 5.5 preferred, 5.4 ok; no GPT-4.x defaults).
- Prefer behavior tests over string greps.
- Clean timers/env/globals/mocks between tests. `--isolate=false` is safe.
- Prefer injection and narrow `*.runtime.ts` mocks over broad barrel mocks.
- Max 16 test workers. Memory pressure: `OPENCLAW_VITEST_MAX_WORKERS=1 pnpm test`.
- Do not run independent `pnpm test` commands concurrently in one worktree (Vitest cache races).
- Test guide: `docs/reference/test.md`.

## Build & Toolchain

- **Runtime**: Node 22.19+; Node 24 recommended. Bun also supported.
- **Build**: `tsdown` (Rollup-based). Outputs to `dist/`.
- **Typecheck**: `tsgo` (Go-based TS checker), NOT `tsc`. Scripts: `pnpm tsgo`, `pnpm tsgo:core`, `pnpm tsgo:extensions`.
- **Lint/Format**: oxlint + oxfmt. Configs: `.oxlintrc.json`, `.oxfmtrc.jsonc`.
- **Package manager**: pnpm only. No swaps without approval.

## Git

- Commit via `scripts/committer "<msg>" <file...>`. Stage intended files only.
- Conventional-ish commit messages, concise, grouped.
- `main`: no merge commits. Rebase on latest `origin/main` before push.
- `ship it`: changelog if needed, commit, `git pull --rebase`, push.
- Build before push when build output, packaging, lazy/module boundaries, dynamic imports, or published surfaces can change.

## Security

- Never commit real phone numbers, videos, credentials, or live config.
- Channel/provider credentials: `~/.openclaw/credentials/`.
- Dependency patches/overrides need explicit approval.
- Releases/publish/version bumps need explicit approval.

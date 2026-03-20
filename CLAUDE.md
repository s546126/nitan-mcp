# CLAUDE.md — AI Assistant Guide for nitan-mcp

## Project Overview

**nitan-mcp** is a heavily modified Discourse MCP (Model Context Protocol) server tailored for uscardforum.com (a Chinese credit card forum). It exposes forum capabilities as MCP tools for AI agents, with a focus on bypassing Cloudflare protection.

- **Package**: `@nitansde/mcp` (binary: `nitan-mcp`)
- **Version**: 2.0.0
- **Node.js**: ≥18 required
- **Package Manager**: pnpm 10.14.0

---

## Repository Structure

```
src/
├── index.ts                    # Entry point: CLI parsing, server startup
├── http/
│   ├── client.ts               # Core HTTP client with bypass logic
│   ├── browser_fallback.ts     # Playwright-based browser fallback (macOS)
│   ├── browser_fallback_defaults.ts
│   ├── cloudscraper.ts         # Python cloudscraper integration
│   ├── cloudscraper_wrapper.py
│   ├── curl_cffi.ts            # Python curl_cffi integration
│   ├── curl_cffi_wrapper.py
│   └── cache.ts                # HTTP caching
├── tools/
│   ├── registry.ts             # Tool registration (which tools are active)
│   ├── types.ts                # Tool type definitions
│   ├── categories.ts           # Hardcoded forum category mapping
│   ├── builtin/                # 18+ Discourse tool implementations
│   └── remote/
│       └── tool_exec_api.ts    # Remote Tool Execution API
├── site/
│   └── state.ts                # Multi-site state and auth management
├── util/
│   ├── logger.ts               # stderr logger (silent/error/info/debug)
│   ├── redact.ts               # Sensitive field/header redaction
│   └── timestamp.ts
└── test/                       # Node.js native test runner tests
    ├── tools.test.ts
    ├── transport.test.ts
    ├── browser_fallback_defaults.test.ts
    ├── browser_fallback_playwright_reuse.test.ts
    ├── browser_fallback_profile_selection.test.ts
    └── http_client_browser_fallback_autologin.test.ts

skills/nitan/                   # OpenClaw AgentSkill distribution
├── SKILL.md
└── scripts/                    # Shell wrappers for skill invocation

scripts/                        # Build utilities
├── check-python-deps.mjs
├── package-skill.sh
├── sync-fixtures.mjs
└── validate-filter.mjs

.github/workflows/ci.yml        # GitHub Actions CI
```

---

## Development Workflow

### Setup

```bash
pnpm install          # Install Node.js dependencies
pip install -r requirements.txt  # Install Python dependencies (for Cloudflare bypass)
```

### Common Commands

```bash
pnpm build            # Compile TypeScript → dist/ + copy Python files
pnpm test             # Run tests (builds first)
pnpm typecheck        # Type-check without emitting
pnpm dev              # Run with source maps (development mode)
pnpm skill:pack       # Package OpenClaw AgentSkill
pnpm release          # Bump version via standard-version
```

### Build Process

The `build` script does two things:
1. Compiles TypeScript (`tsc`) from `src/` → `dist/`
2. Copies Python wrapper files (`*.py`) from `src/http/` → `dist/http/`

Always run `pnpm build` before running tests; the test runner operates on compiled `dist/` files.

### Testing

Tests use Node.js native test runner (no external framework). Tests live in `src/test/` and are compiled to `dist/test/`.

```bash
pnpm test             # Runs node --test dist/test/**/*.test.js
```

Tests mock HTTP clients and site state where needed. When adding tests, follow the existing pattern of importing from `dist/` paths in the test runner setup.

---

## Key Architectural Concepts

### 1. Multi-Site Support

The server can connect to multiple Discourse sites. Site state is managed in `src/site/state.ts` via the `SiteState` class:
- Maintains per-site HTTP client cache
- Resolves authentication per site
- Supports auth override (API key, User API key, username/password, 2FA)

### 2. Cloudflare Bypass Strategy

Three layers of protection bypass, attempted in order:

1. **Direct HTTP** with browser-like headers (User-Agent: Edge/Windows)
2. **Python bypass** via cloudscraper and/or curl_cffi (configured via `--bypass` flag)
3. **Browser fallback** via Playwright (macOS only) or OpenClaw proxy

Key files: `src/http/client.ts`, `src/http/cloudscraper.ts`, `src/http/curl_cffi.ts`, `src/http/browser_fallback.ts`

### 3. Tool Registry

`src/tools/registry.ts` is the single source of truth for which tools are active. Some tools are intentionally disabled:
- `discourse_list_categories` — uses hardcoded mapping in `categories.ts` instead
- `discourse_list_tags`, `discourse_filter_topics`, `discourse_read_post` — low utility/redundant

Write tools (`create_post`, `create_topic`, `create_category`, `create_user`) are only enabled when:
- `allowWrites=true` AND
- `read_only=false` AND
- A matching `auth_pairs` entry exists for the site

### 4. Authentication

Four auth modes supported (configured per-site in `auth_pairs`):
- **None**: Public read-only access
- **API key**: `api_key` + `api_username`
- **User API key**: `User-Api-Key` + `User-Api-Client-Id`
- **Credentials**: `username` + `password` (+ optional `second_factor_token` for 2FA)

### 5. Transport

Supports both `stdio` (default) and `http` transport modes, configured via `--transport` flag.

---

## Code Conventions

### TypeScript

- **Strict mode** is enabled (`tsconfig.json`)
- **ESM modules** only (`"type": "module"` in package.json, `"module": "NodeNext"`)
- All imports must use `.js` extension even for `.ts` source files (NodeNext resolution)
- Target: ES2020

### Logging

Use the logger from `src/util/logger.ts`. Always log to stderr (never stdout, which is reserved for MCP protocol):

```typescript
import { logger } from '../util/logger.js';
logger.info('message');
logger.debug('object:', obj);
logger.error('error:', err);
```

### Secret Redaction

Before logging any config or request data, run it through `redact()` from `src/util/redact.ts`. Sensitive fields: `password`, `api_key`, `user_api_key`, `second_factor_token`. Sensitive headers: `Api-Key`, `User-Api-Key`, `Authorization`.

### Error Handling

Use `HttpError` class from `src/http/client.ts` for HTTP errors. Include status codes and response bodies in errors for debugging.

### Categories

Forum categories are **hardcoded** in `src/tools/categories.ts`, not fetched dynamically. Update this file when categories change rather than adding dynamic fetching.

---

## Adding New Tools

1. Create implementation in `src/tools/builtin/your_tool.ts`
2. Export a function matching the `ToolHandler` type from `src/tools/types.ts`
3. Register it in `src/tools/registry.ts`
4. Add documentation to `TOOLS.md`

Tool handlers receive `(args: unknown, site: SiteState)` and must return MCP-compatible content.

---

## Configuration Reference

CLI flags (parsed in `src/index.ts`):

| Flag | Description |
|------|-------------|
| `--profile <path>` | JSON profile file path |
| `--site <url>` | Pre-select a site (tethered mode) |
| `--allow-writes` | Enable write tools |
| `--bypass <method>` | Cloudflare bypass: `cloudscraper`, `curl_cffi`, or `both` |
| `--transport <type>` | `stdio` (default) or `http` |
| `--log-level <level>` | `silent`, `error`, `info`, `debug` |
| `--tools-mode <mode>` | `auto`, `discourse_api_only`, `tool_exec_api` |
| `doctor` | Run health check diagnostics |
| `generate-user-api-key` | Generate Discourse user API key |

Profile JSON schema (subset of CLI flags, merged with lower priority):

```json
{
  "auth_pairs": [
    {
      "site": "https://example.com",
      "api_key": "...",
      "api_username": "..."
    }
  ],
  "bypass": "both",
  "transport": "stdio",
  "log_level": "info"
}
```

---

## CI/CD

GitHub Actions workflow (`.github/workflows/ci.yml`) runs on PRs and pushes to main:
1. `pnpm install`
2. `pnpm typecheck`
3. `pnpm build`
4. `pnpm test`
5. `pnpm skill:pack` (uploads skill artifact)

On main branch: publishes to npm automatically.

---

## Python Integration

Python scripts run as child processes for Cloudflare bypass:
- `src/http/cloudscraper_wrapper.py` — called by `src/http/cloudscraper.ts`
- `src/http/curl_cffi_wrapper.py` — called by `src/http/curl_cffi.ts`

Python files are copied to `dist/http/` during build. The `doctor` command checks Python dependency availability.

---

## Skill Distribution (OpenClaw)

The `skills/nitan/` directory contains an OpenClaw AgentSkill for distributing the MCP as a skill:
- Shell scripts wrap common tool invocations
- Default: uses `npx --no-install nitan-mcp` (requires prior install)
- Install-on-demand: opt-in via `NITAN_MCP_ALLOW_INSTALL=1`

Key env vars for skill scripts:
- `NITAN_MCP_PACKAGE` — package name/version (default: `nitan-mcp`)
- `NITAN_MCP_ALLOW_INSTALL` — allow auto-install (default: `0`)
- `NITAN_MCP_RESPONSE_TIMEOUT` — timeout in seconds (default: `120`)
- `NITAN_USERNAME`, `NITAN_PASSWORD` — optional credentials

Package the skill with `pnpm skill:pack`.

---

## Important Notes for AI Assistants

1. **Never log to stdout** — it's reserved for MCP stdio transport. All logs go to stderr via `logger`.
2. **Always redact secrets** before logging config objects.
3. **ESM imports require `.js` extension** in TypeScript source files.
4. **Build before testing** — tests run on compiled `dist/` files, not source.
5. **Python deps are optional** — the server works without them but loses Cloudflare bypass capability. Never make Python deps required.
6. **Write tools are gated** — check `allowWrites`, `read_only`, and `auth_pairs` before assuming write tools are available.
7. **Categories are hardcoded** — don't add dynamic category fetching; update `categories.ts` instead.
8. **Browser fallback is macOS-only** — Playwright-based fallback only works on macOS; other platforms use OpenClaw proxy fallback.

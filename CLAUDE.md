# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is Agent Maestro?

A VS Code extension that acts as a unified proxy and orchestrator for AI coding agents (Roo Code, Cline, Kilo Code) and LLM APIs (Anthropic, OpenAI, Gemini). It exposes REST API endpoints compatible with each provider's format and an MCP server for headless agent automation.

## Commands

```bash
# Build (type-check + lint + bundle)
pnpm run build

# Production build (clean + type-check + lint + minified bundle)
pnpm run build:prod

# Watch mode (esbuild + tsc in parallel)
pnpm run watch

# Type-check only
pnpm run check-types

# Lint
pnpm run lint

# Run all tests (builds tests, builds extension, lints, then runs tests)
pnpm test

# Build tests only (compiles to out/)
pnpm run build-tests

# Watch tests during development
pnpm run watch-tests

# Package VSIX
pnpm run vscode:package

# Version bump via changesets
pnpm run changeset:version
```

**Requirements:** Node.js >=22, pnpm 10.28.2

Tests use `@vscode/test-cli` with Mocha. Test files live in `src/test/` and compile to `out/test/**/*.test.js`. Tests run inside a VS Code instance (electron), so they cannot be run with plain `mocha` directly.

## Architecture

```
src/extension.ts          → activate/deactivate, wires up controller + servers
src/core/
  controller.ts           → ExtensionController: orchestrates adapters
  ExtensionBaseAdapter.ts → Abstract base for extension discovery/activation
  RooCodeAdapter.ts       → Roo Code event queuing, task creation, messaging
  ClineAdapter.ts         → Cline extension management
  McpTaskManager.ts       → MCP parallel task execution
src/server/
  ProxyServer.ts          → Hono HTTP server (default port 23333)
  McpServer.ts            → FastMCP server (default port 23334)
  routes/                 → API endpoint handlers
  schemas/                → Zod request/response validation
  middleware/             → Auth middleware (constant-time API key comparison)
src/commands/             → VS Code command registrations
src/utils/                → Config, logging, crypto, model matching, etc.
```

### Key data flow

1. External tools (Claude Code, Codex, Gemini CLI) send requests to the ProxyServer using their native API format (Anthropic/OpenAI/Gemini).
2. Routes convert incoming messages to VS Code's `LanguageModelChat` API format.
3. The VS Code LM API forwards to whichever model provider the user has configured (e.g., Copilot, other extensions).
4. Responses stream back through SSE, converted back to the caller's expected format.

For headless agent control, REST endpoints create tasks on Roo Code/Cline adapters, which manage event queues and action approval flows.

### Build system

esbuild bundles `src/extension.ts` → `dist/extension.cjs` (CommonJS, required by VS Code). External packages: `vscode`, `@valibot/to-json-schema`, `effect`, `sury`.

### Test conventions

- Framework: Mocha with `assert` (not chai/jest)
- Structure: `suite()` / `test()` pattern
- Tests mirror source structure under `src/test/`
- A separate `tsconfig.test.json` compiles tests to `out/` with CommonJS output

## Code style

- ESLint enforces: camelCase/PascalCase imports, curly braces, strict equality, semicolons, no literal throws
- Prettier with `@trivago/prettier-plugin-sort-imports` auto-sorts imports (third-party first, then local with separator)
- Pre-commit hooks via Husky

## Release process

Use `/release` slash command. It runs `changeset:version`, creates a `release/v{VERSION}` branch, and opens a PR. CI publishes to VS Code Marketplace and Open VSX on version tags.

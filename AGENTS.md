# AGENTS.md

Guidance for AI agents working in this repository.

## Project

`remote_ai` — a Node.js (ESM) Telegram bot that bridges Telegram and the OpenCode
AI agent (`opencode serve`). Two cooperating processes: an `opencode serve` HTTP server
(localhost:4096, Basic Auth) and the Telegram bot (`src/index.js`, grammy long polling).

## Requirements

- Node.js >= 18 (`package.json` engines)
- npm (lockfile is `package-lock.json`)
- `opencode` CLI on PATH (`OPENCODE_BIN` overrides the binary name)

## Setup

1. `npm install`
2. Copy `.env-example` to `.env` and fill in `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USERS`,
   and `OPENCODE_SERVER_PASSWORD`. The supervisor injects the server password into `.env`
   on startup when started via `npm start`.

## Commands

| Command | Purpose |
|---------|---------|
| `npm start` | Run the cross-platform supervisor (`scripts/start.js`): starts `opencode serve`, waits for the port, then starts the bot. Recommended. |
| `npm run start:skip-permissions` | Same, but starts the server with `--dangerously-skip-permissions` (auto-confirm mode, like `/danger on`). |
| `npm run start:bot` | Run **only** the bot (`node src/index.js`), no `opencode serve`. Use when the server is already running. |
| `npm run serve` | Run `opencode serve --port 4096 --hostname 127.0.0.1` directly. |
| `node scripts/start.js [--skip-permissions] [--no-serve]` | Supervisor entry point directly. |
| `./start-bot.sh [--skip-permissions] [--no-serve]` | Linux/macOS thin wrapper over the supervisor. |
| `.\start-bot.ps1 [-SkipPermissions] [-NoServe]` | Windows thin wrapper over the supervisor. |

The supervisor auto-frees port 4096, kills stale bot processes, and restarts the whole
stack when `/danger on|off` writes `data/danger.flag`.

## Verification / Testing

There is currently **no test suite and no linter/formatter configured**
(no ESLint, Prettier, jest, vitest, or CI workflow files exist in the repo).

To sanity-check changes without a live Telegram/OpenCode backend:
- `node -c scripts/start.js` / `node -c src/index.js` — syntax-check a file.
- `node --check <file>` — confirm ESM parses.
- TODO: add a lint setup (e.g. ESLint flat config) and at least module-level smoke tests;
  update this section once present.

## Code conventions

- **ESM only**: `"type": "module"` in `package.json`; use `import`/`export`, no `require`.
- **Plain JavaScript** (`.js`), no TypeScript, no build step.
- **Comments and user-facing strings are in Russian (Cyrillic)** — match this when editing.
- **Lazy singletons**: clients/stores are created on first use and cached in module scope
  (see `ensureClient()` in `src/index.js`, `getClient()` in `src/opencode-client.js`).
- **JSON file persistence**: state lives in `data/` (`sessions.json`, `users.json`),
  which is gitignored. Stores load-once into an in-memory cache and re-serialize on writes.
- **No comments in new code unless asked** (per repo editing guidance); existing code
  comments are explanatory Russian notes.

## Source map

- `src/index.js` — entry point and all Telegram command handlers (`/code`, `/stop`,
  `/new`, `/model`, `/models`, `/session`, `/sessions`, `/switch`, `/projects`, `/danger`).
- `src/opencode-client.js` — builds the `@opencode-ai/sdk` client with Basic Auth.
- `src/session-store.js` — maps chatId → sessionId in `data/sessions.json`.
- `src/user-store.js` — per-user model selection in `data/users.json`.
- `src/stream-handler.js` — SSE streaming, throttled Telegram edits, abort/idle handling.
- `src/message-formatter.js` — splits messages at the 4000-char Telegram limit.
- `scripts/start.js` — cross-platform process supervisor (serve + bot + restart-on-flag).

## Environment variables

Defined in `.env-example` and read across the codebase:

- `TELEGRAM_BOT_TOKEN` (required) — bot token from BotFather.
- `TELEGRAM_ALLOWED_USERS` — comma-separated Telegram user IDs; empty = open to all
  (prints a warning). Restrict access by listing user IDs here.
- `OPENCODE_SERVER_URL` (default `http://localhost:4096`)
- `OPENCODE_SERVER_USERNAME` (default `opencode`)
- `OPENCODE_SERVER_PASSWORD` (required) — Basic Auth password for the server.
- `DEFAULT_MODEL_PROVIDER` (default `opencode`) / `DEFAULT_MODEL_ID` (default `big-pickle`)
- `OPENCODE_PORT` (default `4096`) / `OPENCODE_HOST` (default `127.0.0.1`)
- `OPENCODE_BIN` (default `opencode`, or `opencode.cmd` on Windows)
- `TELEGRAM_THROTTLE_MS` (default `1000`) — min interval between Telegram message edits.
- `SSE_IDLE_TIMEOUT_MS` (default `300000` / 5 min) — idle timeout for SSE subscriptions.
- `DEBUG_UPDATES=true` — verbose logging of incoming Telegram updates.

`data/`, `.env`, `node_modules/`, `.idea/`, and `*.log` are gitignored — never commit these.

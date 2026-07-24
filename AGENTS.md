# AGENTS.md

Guidance for AI coding agents working in this repository.

## Project Overview

Telegram bot that bridges Telegram chats to a local OpenCode AI server (`opencode serve`). Users send `/code <query>` and receive streamed, token-by-token replies in Telegram via SSE. Two cooperating processes:

1. **`opencode serve`** — local HTTP API to AI models (default `127.0.0.1:4096`, Basic Auth).
2. **Node.js bot** (`src/index.js`) — long-polls Telegram, streams responses back.

The cross-platform supervisor (`scripts/start.js`) boots both, watches `data/danger.flag`, and restarts the pair when the user toggles `/danger on|off`.

## Tech Stack

- **Runtime:** Node.js >= 18, ESM (`"type": "module"` in `package.json`)
- **Language:** Plain JavaScript (`.js`), no TypeScript, no build step
- **Dependencies:** `grammy` (Telegram Bot API), `@opencode-ai/sdk` (OpenCode HTTP client), `dotenv`
- **Package manager:** npm (`package-lock.json` present)
- **UI language:** Russian (all bot messages and code comments are in Russian/Cyrillic)

## Setup

1. `npm install`
2. Copy `.env-example` to `.env` and fill in `TELEGRAM_BOT_TOKEN`, `TELEGRAM_ALLOWED_USERS`, and `OPENCODE_SERVER_PASSWORD` (see [Environment Variables](#environment-variables)).
3. Ensure the `opencode` CLI is installed and on PATH.

## Commands

| Command | Purpose |
|---------|---------|
| `npm start` | Run the supervisor (`scripts/start.js`): starts `opencode serve`, waits for port, then starts the bot. Recommended. |
| `npm run start:skip-permissions` | Same, but starts the server with `--dangerously-skip-permissions` (auto-confirm mode). |
| `npm run start:bot` | Run **only** the bot (`node src/index.js`). Use when `opencode serve` is already running. |
| `npm run serve` | Run `opencode serve --port 4096 --hostname 127.0.0.1` directly. |
| `node scripts/start.js [--skip-permissions] [--no-serve]` | Supervisor entry point directly. Flags: `--no-serve` (skip server), `--skip-permissions` (auto-confirm). |
| `./start-bot.sh [--skip-permissions] [--no-serve]` | Linux/macOS thin wrapper over the supervisor. |
| `.\start-bot.ps1 [-SkipPermissions] [-NoServe]` | Windows thin wrapper over the supervisor. |

## Verification / Testing

There is **no test suite and no linter/formatter configured** (no ESLint, Prettier, jest, vitest, or CI workflow files exist).

To sanity-check changes without a live Telegram/OpenCode backend:
- `node --check scripts/start.js` / `node --check src/index.js` — syntax-check a file.

TODO: add a lint setup (e.g. ESLint flat config) and module-level smoke tests; update this section once present.

## Source Modules

| File | Responsibility |
|------|---------------|
| `src/index.js` | Entry point. All Telegram command handlers, middleware (access control, logging), message formatting. |
| `src/opencode-client.js` | Lazy singleton for `@opencode-ai/sdk` client with Basic Auth. Throws if `OPENCODE_SERVER_PASSWORD` is unset. |
| `src/session-store.js` | JSON-file persistence (`data/sessions.json`): chatId -> sessionId mapping. In-memory cache, load-once. |
| `src/user-store.js` | JSON-file persistence (`data/users.json`): per-user model preferences (`provider`, `modelId`). In-memory cache, load-once. |
| `src/stream-handler.js` | SSE subscription, throttled Telegram message edits, abort/idle handling, 429 backoff. |
| `src/message-formatter.js` | Splits long messages at 4000-char Telegram limit, respecting newlines/spaces. Also exports `formatErrorMessage` (currently unused/dead code). |
| `scripts/start.js` | Cross-platform supervisor: port readiness check, stale-process cleanup, danger-flag relay, graceful shutdown. |

## Environment Variables

All variables are read via `dotenv` from `.env` (see `.env-example`). Only variables actually referenced in source code are listed here.

**Required:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `TELEGRAM_BOT_TOKEN` | — | Bot token from @BotFather. Bot exits if unset. |
| `OPENCODE_SERVER_PASSWORD` | — | Basic Auth password for `opencode serve`. Client throws if unset. |

**Access control:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `TELEGRAM_ALLOWED_USERS` | (empty) | Comma-separated Telegram user IDs (whitelist). **Empty = bot is open to everyone** (warning logged at startup). |

> Note: `TELEGRAM_ALLOW_ALL` is mentioned in `README.md` but is **not referenced in any source file**. The actual access control logic simply checks whether `TELEGRAM_ALLOWED_USERS` is empty.

**Model defaults:**

| Variable | Default | Purpose |
|----------|---------|---------|
| `DEFAULT_MODEL_PROVIDER` | `opencode` | Fallback model provider |
| `DEFAULT_MODEL_ID` | `big-pickle` | Fallback model ID |

**Supervisor / server tuning (read by `scripts/start.js`):**

| Variable | Default | Purpose |
|----------|---------|---------|
| `OPENCODE_PORT` | `4096` | Port for `opencode serve` |
| `OPENCODE_HOST` | `127.0.0.1` | Host for `opencode serve` |
| `OPENCODE_BIN` | `opencode` / `opencode.cmd` (Win) | Binary name for `opencode` CLI |

**Bot tuning (read by `src/stream-handler.js` and `src/index.js`):**

| Variable | Default | Purpose |
|----------|---------|---------|
| `TELEGRAM_THROTTLE_MS` | `1000` | Min interval between Telegram message edits while streaming |
| `SSE_IDLE_TIMEOUT_MS` | `300000` (5 min) | Idle timeout for SSE subscription; aborts if no events arrive |
| `DEBUG_UPDATES` | `false` | Set `true` for verbose logging of every incoming Telegram update |
| `OPENCODE_SERVER_URL` | `http://localhost:4096` | OpenCode HTTP API base URL |
| `OPENCODE_SERVER_USERNAME` | `opencode` | Basic Auth username |

**Internal (set by supervisor, not user-configured):**

| Variable | Set by | Purpose |
|----------|--------|---------|
| `OPENCODE_SUPERVISOR_PID` | `scripts/start.js` | PID of supervisor; bot checks this to know if `/danger` can signal a restart via file flag. |
| `OPENCODE_SERVER_ARGS` | `scripts/start.js` | Tracks `--dangerously-skip-permissions` state; read by `/session` command to display danger status. |

## Key Workflows

### Supervisor lifecycle (`scripts/start.js`)

1. **Preflight:** frees `OPENCODE_HOST:OPENCODE_PORT` (kills stale listeners via `lsof`/`fuser` on Unix, `netstat` on Windows), kills stale `node src/index.js` processes (`pgrep` on Unix, `wmic` on Windows).
2. Starts `opencode serve` with optional `--dangerously-skip-permissions`.
3. Waits for TCP connection on configured host:port (polls every 300ms, 20s timeout).
4. Starts bot as child process, sets `OPENCODE_SUPERVISOR_PID` in its env.
5. On bot exit: if `data/danger.flag` exists -> reads mode -> restarts both processes in requested mode; otherwise exits (so an external process manager can intervene).
6. On `opencode serve` exit -> stops bot too.
7. `SIGINT`/`SIGTERM` -> graceful shutdown of both children (SIGTERM then SIGKILL after timeout).

### `/danger on|off` flow

1. Bot writes `on`/`off` to `data/danger.flag` and exits (`process.exit(0)` after 500ms delay for Telegram delivery).
2. Supervisor detects flag on bot exit, reads mode, deletes the flag.
3. Supervisor restarts `opencode serve` with or without `--dangerously-skip-permissions`.
4. Supervisor restarts bot child process.
5. Fallback: if not running under supervisor, on Windows only, spawns `start-bot.ps1` in a new process.

### Streaming response flow (`src/stream-handler.js`)

1. Bot sends placeholder message ("⏳ печатает...").
2. Subscribes to SSE events (`client.event.subscribe()`) and sends `client.session.prompt()` (HTTP) in parallel.
3. Accumulates `session.part` text events into a buffer; renders to Telegram with throttling (`TELEGRAM_THROTTLE_MS`).
4. `permission.updated` SSE events with `state: "pending"` are surfaced to the chat as info messages.
5. On `session.done` or `session.continue` -> SSE stream ends.
6. Final text from the HTTP prompt response is used if longer than the SSE buffer.
7. Message is finalized with "✅ Готово" suffix.
8. Handles 429 rate limits with `retry_after` backoff; silently ignores "message is not modified" errors.
9. `AbortController` per request (`promptController` + `sseController`) -> `/stop` aborts both.

### Session lifecycle

- First `/code` in a chat auto-creates a session titled `Telegram-{chatId}`.
- `/new` without args deletes the local mapping only; next `/code` creates a fresh session.
- `/new <title>` creates a named session on the server immediately.
- `/switch <id>` reuses an existing session; server validates it exists.
- Stale session IDs (deleted on server, 404) are detected on next `/code` and auto-replaced. Non-404 errors are propagated (don't lose context on transient failures).

## Telegram Bot Commands

Registered in `src/index.js`:

| Command | Purpose |
|---------|---------|
| `/start` | Welcome message + command list; shows current model. |
| `/help` | Full command reference; shows current model. |
| `/code <query>` | Send prompt to AI; creates/reuses session, streams response. |
| `/stop` | Abort current generation (AbortController on prompt + SSE). |
| `/new [title]` | Reset conversation. With title -> creates named session on server; without -> clears local mapping only. |
| `/model <provider/model>` | Show or set the active model for the user. Persists to `data/users.json`. |
| `/models` | List available models (shells out to `opencode models` CLI). |
| `/sessions` | List recent sessions (up to 15, always includes current). |
| `/switch <id>` | Switch to an existing session by ID. Server validates existence. |
| `/session [id] [-f]` | Session info (messages, edits, model, folder, danger status). `-f` sends `.md` file with full history. |
| `/projects` | List server projects (filters out `global`). |
| `/danger <on\|off>` | Toggle auto-confirm mode. Writes `data/danger.flag` -> supervisor restarts both processes. |

## Data Files (gitignored)

- `data/sessions.json` — `{ chatId: { sessionId, createdAt } }`
- `data/users.json` — `{ chatId: { provider, modelId } }`
- `data/danger.flag` — transient; contains `on` or `off`; deleted by supervisor after read.

## Conventions

- **ESM only:** use `import`/`export`, not `require`. No build step.
- **Lazy singletons:** clients/stores are created on first use and cached in module scope.
- **JSON file persistence:** state lives in `data/` with in-memory cache, load-once, re-serialize on writes.
- **Comments and user-facing strings are in Russian** — match this when editing.
- Do not commit `data/`, `.env`, `node_modules/`, or `.idea/` (all gitignored).
- Do not edit `data/` (runtime state) directly.

<!-- TODO: add lint/test commands here once configured in package.json. -->

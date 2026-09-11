# AGENTS.md

Reference for AI agents working on this repository.

## Project

Telegram bot that bridges Telegram with [OpenCode](https://opencode.ai) AI-agent.
Sends prompts to `opencode serve` via `@opencode-ai/sdk` and streams responses
back to Telegram token-by-token using SSE (Server-Sent Events).

- **Runtime:** Node.js >= 18 (ESM — `"type": "module"`)
- **Key deps:** grammy (Telegram Bot API), @opencode-ai/sdk, dotenv
- **Language:** JavaScript (no TypeScript, no build step)

## Commands

```bash
npm install                      # install dependencies
npm start                        # supervisor: starts opencode serve + bot (default mode)
npm run start:bot                # bot only — no opencode serve restart
npm run start:skip-permissions   # supervisor with --dangerously-skip-permissions
npm run serve                    # standalone opencode serve on 127.0.0.1:4096
node scripts/start.js [--skip-permissions] [--no-serve]   # direct supervisor
./start-bot.sh                   # Linux/macOS wrapper around scripts/start.js
.\start-bot.ps1                  # Windows wrapper (PowerShell)
```

There is no test suite, linter, or type checker configured in this repo.

## Architecture

Two-process model managed by `scripts/start.js` (the supervisor):

1. **opencode serve** — HTTP API server on `127.0.0.1:4096` (not exposed externally)
2. **Telegram bot** (`src/index.js`) — long-polls Telegram, proxies to the server

The supervisor waits for the server port to be ready before starting the bot,
kills stale bot processes and frees the port on startup, and handles the
`/danger on|off` restart cycle via a file flag at `data/danger.flag`.

## Source map

| File | Role |
|------|------|
| `src/index.js` | Entry point; registers all Telegram bot commands |
| `src/opencode-client.js` | Creates and caches the `@opencode-ai/sdk` client with Basic Auth |
| `src/session-store.js` | Persists chatId → sessionId mapping in `data/sessions.json` |
| `src/user-store.js` | Persists per-user model preferences in `data/users.json` |
| `src/stream-handler.js` | SSE subscription, throttled message editing, abort handling |
| `src/message-formatter.js` | Splits long messages into <= 4000-char Telegram chunks |
| `scripts/start.js` | Cross-platform supervisor (serve + bot lifecycle) |

Data files (`data/*.json`) are gitignored and auto-created at runtime.

## Environment variables

Defined in `.env-example` (copy to `.env`):

| Variable | Purpose |
|----------|---------|
| `TELEGRAM_BOT_TOKEN` | Bot token from @BotFather (required) |
| `TELEGRAM_ALLOWED_USERS` | Comma-separated whitelist of Telegram user IDs; empty = open to all (with console warning) |
| `OPENCODE_SERVER_URL` | Base URL for OpenCode server (default `http://localhost:4096`) |
| `OPENCODE_SERVER_USERNAME` | Basic Auth username (default `opencode`) |
| `OPENCODE_SERVER_PASSWORD` | Basic Auth password (required) |
| `DEFAULT_MODEL_PROVIDER` | Fallback model provider (default `opencode`) |
| `DEFAULT_MODEL_ID` | Fallback model ID (default `big-pickle`) |

Additional variables discovered in source (not in `.env-example`):

| Variable | Source | Purpose |
|----------|--------|---------|
| `OPENCODE_PORT` | `scripts/start.js` | Server port (default `4096`) |
| `OPENCODE_HOST` | `scripts/start.js` | Server bind host (default `127.0.0.1`) |
| `OPENCODE_BIN` | `scripts/start.js` | Path to `opencode` binary (default `opencode` / `opencode.cmd` on Windows) |
| `OPENCODE_SUPERVISOR_PID` | set by supervisor | Read by `/danger` command to signal restart via file flag |
| `OPENCODE_SERVER_ARGS` | set by supervisor | Stores `--dangerously-skip-permissions` state for `/session` display |
| `TELEGRAM_THROTTLE_MS` | `stream-handler.js` | Delay between Telegram message edits (default `1000`) |
| `SSE_IDLE_TIMEOUT_MS` | `stream-handler.js` | SSE idle timeout before closing subscription (default 5 min) |
| `DEBUG_UPDATES` | `src/index.js` | Set to `true` for verbose Telegram update logging |

> TODO: `TELEGRAM_ALLOW_ALL` is referenced in `README.md` as a fail-closed
> toggle, but is **not** implemented in the current source. Actual access
> control: an empty `TELEGRAM_ALLOWED_USERS` opens the bot to all users (with a
> console warning). Reconcile README with code before documenting further.

## Telegram bot commands

| Command | Description |
|---------|-------------|
| `/code <query>` | Send a prompt to the active model |
| `/stop` | Abort current generation (prompt + SSE) |
| `/new [title]` | Start a new session (resets context; optional title) |
| `/model [provider/model]` | View or set the active model for this user |
| `/models` | List available models (shells out to `opencode models` CLI) |
| `/sessions` | List recent sessions (max 15, sorted by last updated) |
| `/switch <id>` | Switch context to an existing session by ID |
| `/session [id] [-f]` | Show session info; `-f` sends full history as a `.md` file |
| `/projects` | List all projects known to the server (excludes `global`) |
| `/danger <on\|off>` | Toggle auto-confirm mode (triggers full supervisor restart) |

## Conventions

- All source files use ESM (`import`/`export`).
- Russian is the primary UI language for bot responses; code comments are in Russian.
- The `opencode` CLI is invoked directly for `opencode models` (no SDK endpoint used).
- Session and user state are simple JSON files with an in-memory cache (no database).
- Message editing is throttled to respect Telegram rate limits (~1 edit/sec per chat).

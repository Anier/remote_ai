# AGENTS.md — Reference for AI Agents

## Project Summary

Telegram bot that proxies user messages to an OpenCode AI server (`opencode serve`) and streams token-by-token responses back to Telegram via long polling + SSE.

- **Runtime:** Node.js ≥ 18, ESM (`"type": "module"`)
- **Stack:** grammy (Telegram), @opencode-ai/sdk (OpenCode HTTP API), dotenv
- **Language:** Russian UI (all bot messages are in Russian)

## Quick Start

```bash
npm install
cp .env-example .env   # fill in TELEGRAM_BOT_TOKEN, OPENCODE_SERVER_PASSWORD, etc.
npm start              # starts supervisor → opencode serve + Telegram bot
```

## npm Scripts

| Script | Command | What it does |
|--------|---------|--------------|
| `start` | `node scripts/start.js` | Supervisor: launches `opencode serve`, waits for port, then starts bot |
| `start:bot` | `node src/index.js` | Bot only (no server lifecycle) |
| `start:skip-permissions` | `node scripts/start.js --skip-permissions` | Supervisor with `--dangerously-skip-permissions` |
| `serve` | `opencode serve --port 4096 --hostname 127.0.0.1` | OpenCode server only |

Supervisor flags: `--no-serve` (skip server), `--skip-permissions` / `-SkipPermissions` (auto-confirm mode).

Shell wrappers: `./start-bot.sh` (bash), `.\start-bot.ps1` (PowerShell) — both delegate to `scripts/start.js`.

## Environment Variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `TELEGRAM_BOT_TOKEN` | — | **Required.** Bot token from @BotFather |
| `TELEGRAM_ALLOWED_USERS` | (empty = open to all) | Comma-separated Telegram user IDs for access control |
| `TELEGRAM_ALLOW_ALL` | — | Set `true` to explicitly allow all users (fail-closed otherwise) |
| `DEBUG_UPDATES` | `false` | Verbose logging of every incoming Telegram update |
| `OPENCODE_SERVER_URL` | `http://localhost:4096` | OpenCode server base URL |
| `OPENCODE_SERVER_USERNAME` | `opencode` | Basic auth username |
| `OPENCODE_SERVER_PASSWORD` | — | **Required.** Basic auth password |
| `DEFAULT_MODEL_PROVIDER` | `opencode` | Fallback model provider |
| `DEFAULT_MODEL_ID` | `big-pickle` | Fallback model ID |
| `OPENCODE_PORT` | `4096` | Port for `opencode serve` (supervisor) |
| `OPENCODE_HOST` | `127.0.0.1` | Host for `opencode serve` (supervisor) |
| `OPENCODE_BIN` | `opencode` / `opencode.cmd` (Win) | Binary name for `opencode` CLI |
| `TELEGRAM_THROTTLE_MS` | `1000` | Min interval between Telegram message edits (rate-limit) |
| `SSE_IDLE_TIMEOUT_MS` | `300000` (5 min) | Idle timeout for SSE subscription |
| `OPENCODE_SUPERVISOR_PID` | set by supervisor | Communicated to bot child process |
| `OPENCODE_SERVER_ARGS` | set by supervisor | Tracks `--dangerously-skip-permissions` state |

## Source Modules

| File | Responsibility |
|------|---------------|
| `src/index.js` | Entry point. Bot commands, middleware (auth, logging), message formatting |
| `src/opencode-client.js` | Lazy singleton for `@opencode-ai/sdk` client with Basic Auth |
| `src/session-store.js` | JSON-file persistence (`data/sessions.json`): chatId → sessionId mapping |
| `src/user-store.js` | JSON-file persistence (`data/users.json`): per-user model preferences |
| `src/stream-handler.js` | SSE subscription, throttled Telegram message edits, abort handling |
| `src/message-formatter.js` | Split long messages at 4000 chars (Telegram limit), respecting newlines/spaces |
| `scripts/start.js` | Cross-platform supervisor: port readiness, stale-process cleanup, danger-flag relay |

## Telegram Bot Commands

| Command | Handler | Notes |
|---------|---------|-------|
| `/start` | Welcome + command list | Shows current model |
| `/help` | Full command reference | Shows current model |
| `/code <query>` | Send prompt to AI | Creates/reuses session, streams response |
| `/stop` | Abort current generation | Uses AbortController on prompt + SSE |
| `/new [title]` | Reset conversation context | With title → creates named session on server; without → just clears local mapping |
| `/model <provider/model>` | Change model for user | Persists to `data/users.json` |
| `/models` | List available models | Shells out to `opencode models` CLI |
| `/sessions` | List recent sessions | Shows up to 15, always includes current |
| `/switch <id>` | Switch to existing session | Validates session exists on server |
| `/session [id] [-f]` | Session info / history download | `-f` sends `.md` file with full message history |
| `/projects` | List server projects | Filters out `global` project |
| `/danger on\|off` | Toggle auto-confirm mode | Writes `data/danger.flag` → supervisor restarts both processes |

## Key Workflows

### /danger Toggle (Mode Switch)

1. Bot writes `on`/`off` to `data/danger.flag` and exits (`process.exit(0)`)
2. Supervisor (`scripts/start.js`) detects flag on bot exit
3. Supervisor restarts `opencode serve` with or without `--dangerously-skip-permissions`
4. Supervisor restarts bot child process

This is the only way to change the permission mode at runtime. Requires running under the supervisor (`npm start`).

### Streaming Response

1. Bot calls `client.session.prompt()` (HTTP) and `client.event.subscribe()` (SSE) in parallel
2. SSE events of type `session.part` (text) are buffered and rendered to Telegram with throttling (`TELETTROTTLE_MS`)
3. On `session.done` / `session.continue` SSE event, stream ends
4. Final text from the prompt HTTP response is used if longer than SSE buffer
5. Message is finalized with "✅ Готово" suffix

### Session Lifecycle

- First `/code` in a chat auto-creates a session (`Telegram-{chatId}` title)
- `/new` without args deletes the local mapping only; next `/code` creates a fresh session
- `/new <title>` creates a named session on the server immediately
- `/switch <id>` reuses an existing session; server validates it exists
- Stale session IDs (deleted on server) are detected on next `/code` and auto-replaced

### Access Control

- `TELEGRAM_ALLOWED_USERS` (comma-separated IDs): empty list = bot open to everyone (with warning)
- Middleware blocks unauthorized users before any command handler runs
- `TELEGRAM_ALLOW_ALL=true` is an explicit opt-in to open access

### Stale Process Cleanup (Supervisor)

On startup, the supervisor:
1. Kills any process listening on `OPENCODE_PORT` (cross-platform: `lsof`/`fuser` or `netstat`)
2. Kills orphaned `node src/index.js` processes (`pgrep` or `wmic`)
3. Waits for port to be free before starting `opencode serve`

## Data Files (gitignored)

- `data/sessions.json` — `{ chatId: { sessionId, createdAt } }`
- `data/users.json` — `{ chatId: { provider, modelId } }`
- `data/danger.flag` — transient; contains `on` or `off`, deleted after read by supervisor

## Testing

No test framework or test files are present in the repo.

TODO: Add unit tests for session-store, user-store, and message-formatter modules.

## Known Patterns & Conventions

- All source files use ESM (`import`/`export`) — no CommonJS
- In-memory cache with lazy JSON-file persistence (session-store, user-store)
- Telegram message edits use exponential backoff on 429 rate-limit errors
- `editWithBackoff` silently ignores non-critical Telegram API errors (next update will overwrite)
- Permission requests from OpenCode (`permission.updated` SSE event) are surfaced to the chat as info messages
- Bot exits with `process.exit(1)` if long polling fails (e.g., token conflict from duplicate instance)

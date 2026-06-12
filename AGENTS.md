# AGENTS.md

Guidance for AI coding agents working in this repository.

## Project Overview

Telegram bot bridging Telegram chats to an OpenCode AI server. Two processes:

1. `opencode serve` — local HTTP API to AI models (default `127.0.0.1:4096`).
2. Node.js bot (`src/index.js`) — long-polls Telegram, streams responses via SSE.

The supervisor `scripts/start.js` boots both, watches `data/danger.flag`, and
restarts the pair when the user toggles `/danger on|off`.

## Workflows

### Run locally

```bash
npm start                        # supervisor: opencode serve + bot
npm run start:bot                # bot only (assumes opencode serve already running)
npm run start:skip-permissions   # supervisor with --dangerously-skip-permissions
npm run serve                    # opencode serve only, port 4096, host 127.0.0.1
```

Direct invocation:

```bash
node scripts/start.js [--skip-permissions] [--no-serve]
./start-bot.sh [--skip-permissions] [--no-serve]   # Linux/macOS wrapper
.\start-bot.ps1 [-SkipPermissions] [-NoServe]      # Windows wrapper
```

The supervisor frees `OPENCODE_PORT` on startup (kills any listener) and also
terminates stale `node src/index.js` processes before booting the bot. It
exports `OPENCODE_SUPERVISOR_PID` to the bot so `/danger` can signal a restart.

### Tests / linting

No test, lint, or build scripts are defined in `package.json`.
TODO: add a test/lint workflow here once one exists.

## Environment

Required `.env` keys (see `.env-example`):

- `TELEGRAM_BOT_TOKEN`
- `OPENCODE_SERVER_URL` (default `http://localhost:4096`)
- `OPENCODE_SERVER_USERNAME`, `OPENCODE_SERVER_PASSWORD`
- `DEFAULT_MODEL_PROVIDER`, `DEFAULT_MODEL_ID`

Access control:

- `TELEGRAM_ALLOWED_USERS=123,456` — comma-separated allowlist of Telegram user
  IDs. If unset, the bot is **open to everyone** (a warning is logged at startup).

Optional tuning:

- `OPENCODE_PORT` (default `4096`), `OPENCODE_HOST` (default `127.0.0.1`)
- `OPENCODE_BIN` (default `opencode` / `opencode.cmd` on Windows)
- `TELEGRAM_THROTTLE_MS` (default `1000`) — min ms between Telegram edits while streaming
- `SSE_IDLE_TIMEOUT_MS` (default 5 min) — abort SSE if no events arrive
- `DEBUG_UPDATES=true` — log every incoming Telegram update (chat/user/text)

## Layout

- `src/index.js` — Telegram handlers and command routing
- `src/opencode-client.js` — OpenCode SDK client (Basic Auth)
- `src/session-store.js` — chatId → sessionId map (persisted to `data/sessions.json`)
- `src/user-store.js` — per-user model selection (persisted to `data/users.json`)
- `src/stream-handler.js` — SSE streaming of model output
- `src/message-formatter.js` — Telegram message chunking/formatting
- `scripts/start.js` — cross-platform supervisor

`data/` is gitignored and is created by the supervisor on first run.

## Telegram commands

Registered in `src/index.js`:

`/start`, `/help`, `/code <q>`, `/stop`, `/new [name]`, `/model [provider/id]`,
`/models`, `/sessions`, `/switch <id>`, `/session [id] [-f]`, `/projects`,
`/danger <on|off>`.

`/danger` writes `data/danger.flag`; the supervisor reads it on bot exit and
restarts the serve+bot pair with or without `--dangerously-skip-permissions`.

## Conventions

- ES modules (`"type": "module"` in `package.json`); use `import`/`export`.
- Node ≥ 18.
- Do not edit `data/` (runtime state) or `node_modules/`.
- Do not commit `.env`.

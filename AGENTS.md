# AGENTS.md

Operational notes for AI agents working in this repo. Keep entries grounded in actual repo usage; prefer TODOs over guesses.

## Project at a glance

- Node.js (>=18) ESM project. Telegram bot bridging users to a local `opencode serve` HTTP API.
- Entry point: `src/index.js`. Cross-platform launcher: `scripts/start.js`.
- Persistent runtime data lives under `data/` (gitignored): `sessions.json`, `users.json`, `danger.flag`.

## Setup

1. Copy `.env-example` to `.env` and fill values (see [Environment variables](#environment-variables)).
2. `npm install`.
3. Ensure the `opencode` CLI is installed and on PATH (the supervisor invokes `opencode serve`).

## Run / dev workflows

Defined in `package.json`:

| Command | What it does |
|---------|--------------|
| `npm start` | Runs the Node supervisor (`scripts/start.js`): starts `opencode serve` then the bot. |
| `npm run start:skip-permissions` | Same as above but passes `--skip-permissions` (server runs `--dangerously-skip-permissions`). |
| `npm run start:bot` | Runs only the bot (`node src/index.js`); assumes `opencode serve` is already up. |
| `npm run serve` | Runs only `opencode serve --port 4096 --hostname 127.0.0.1`. |

Direct invocation of the supervisor (forwards flags):

```bash
node scripts/start.js [--no-serve] [--skip-permissions]
./start-bot.sh        [--no-serve] [--skip-permissions]      # Linux/macOS wrapper
.\start-bot.ps1       [-NoServe]   [-SkipPermissions]        # Windows wrapper
```

Supervisor behavior worth knowing:

- Frees `OPENCODE_HOST:OPENCODE_PORT` on startup (kills stale listeners) and kills stale `node src/index.js` processes.
- Watches `data/danger.flag`. When the bot writes `on`/`off` to it (via `/danger`), the supervisor restarts the serve+bot pair in the requested mode.
- `SIGINT`/`SIGTERM` shut both child processes down cleanly.

There is no test script and no lint script configured in `package.json`. If you add code, run it manually via the supervisor — there is no `npm test`.

## Environment variables

Required (from `.env-example`):

- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_ALLOWED_USERS` — comma-separated Telegram user IDs (whitelist; fail-closed unless `TELEGRAM_ALLOW_ALL=true`)
- `OPENCODE_SERVER_URL` (default `http://localhost:4096`)
- `OPENCODE_SERVER_USERNAME`, `OPENCODE_SERVER_PASSWORD`
- `DEFAULT_MODEL_PROVIDER`, `DEFAULT_MODEL_ID`

Optional tuning (read by `scripts/start.js` and the bot):

- `OPENCODE_PORT` (default `4096`), `OPENCODE_HOST` (default `127.0.0.1`)
- `OPENCODE_BIN` (default `opencode`, or `opencode.cmd` on Windows)
- `TELEGRAM_THROTTLE_MS` (default `1000`) — Telegram message edit throttle
- `SSE_IDLE_TIMEOUT_MS` (default 5 min) — SSE subscription idle timeout
- `TELEGRAM_ALLOW_ALL=true` — disable user whitelist (opt-in)

## Telegram bot commands

Registered in `src/index.js`:

| Command | Purpose |
|---------|---------|
| `/start`, `/help` | Intro and command list. |
| `/code <prompt>` | Send a prompt to the active model. |
| `/stop` | Abort the current generation. |
| `/new [name]` | Start a new session (optional title). |
| `/model [provider/id]` | Show or set the active model for the user. |
| `/models` | List models advertised by the server. |
| `/sessions` | List recent sessions with their working directories. |
| `/switch <id>` | Switch the chat to an existing session. |
| `/session [id] [-f]` | Session stats; `-f` returns the history as a `.md` file. |
| `/projects` | List projects known to the server. |
| `/danger <on\|off>` | Toggle `--dangerously-skip-permissions` (triggers supervisor restart via `data/danger.flag`). |

## Conventions for changes

- Keep edits minimal and grounded; do not invent commands or env vars that aren't referenced in code.
- Don't commit `data/` or `.env`; both are gitignored.
- Branch prefix `tembo/` is used for automated changes.

<!-- TODO: document lint/test commands once they exist in package.json. -->

# telegram-gh-action-dispatcher

A reusable GitHub Action + Cloudflare Worker that replaces cron-based Telegram bot polling with instant webhook delivery.

## How it works

```
Telegram webhook ──▶ Cloudflare Worker ──▶ repository_dispatch ──▶ GitHub Action ──▶ bin/handle_command
```

1. User sends a `/` command in Telegram (`/disable` for example)
2. Telegram sends a webhook POST to your Cloudflare Worker
3. Worker validates the request, parses the command, and calls GitHub's `repository_dispatch` API
4. A workflow in your project picks up the dispatch and runs this action
5. The action calls your project's `bin/handle_command` script

## Consuming this action

### 1. Create `bin/handle_command`

Each project must provide an executable `bin/handle_command` that accepts 4 positional arguments:

```
bin/handle_command <command> <chat_id> <args> <bot_token>
```

| Arg | Description | Example |
|-----|-------------|---------|
| `command` | Detected command name (from Telegram) | `disable` |
| `chat_id` | Telegram chat ID that sent the command | `1234567890` |
| `args` | Everything after the command in the message | `check.yml` |
| `bot_token` | Telegram bot token (for sending replies) | `12345:ABC...` |

The script can be written in any language. The only contract is the positional argument order and that the script exits with code 0 on success.

#### Ruby example

```ruby
#!/usr/bin/env ruby
command, chat_id, args, bot_token = ARGV

require 'bundler/setup'
require 'net/http'
require 'json'

# --- Your command handling logic ---

case command
when 'disable' # e.g. call GitHub API to disable a workflow
  puts "Disabling workflow: #{args}"
when 'enable'
  puts "Enabling workflow: #{args}"
when 'config'
  config = File.read('config.yml')
  puts "Config: #{config}"
else
  puts "Unknown command: #{command}"
end

# --- Send reply to Telegram (optional) ---
uri = URI("https://api.telegram.org/bot#{bot_token}/sendMessage")
Net::HTTP.post(uri, {
  chat_id: chat_id,
  text: "Command `#{command}` processed.",
  parse_mode: 'Markdown'
}.to_json, 'Content-Type' => 'application/json')
```

### 2. Add a workflow

Create `.github/workflows/user_command.yml` in your project:

```yaml
name: Telegram User Command

on:
  repository_dispatch:
    types: [user-command]

jobs:
  handle:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: 01max/telegram-gh-action-dispatcher@v4
        with:
          command: ${{ github.event.client_payload.command }}
          chat_id: ${{ github.event.client_payload.chat_id }}
          args: ${{ github.event.client_payload.args }}
          bot_token: ${{ secrets.TELEGRAM_BOT_TOKEN }}
```

Add any setup steps your handler needs (language setup, config files, etc.) before the action call.

---

## Deploying the Cloudflare Worker

A single Worker handles Telegram commands for any number of GitHub repos. Routing is defined in a Cloudflare KV namespace.

### Prerequisites

- Node.js 20+
- A Cloudflare account (free tier is sufficient)
- A Telegram bot token (from [@BotFather](https://t.me/BotFather))

### Setup

```bash
# 1. Clone this repo
git clone https://github.com/01max/telegram-gh-action-dispatcher.git
cd telegram-gh-action-dispatcher

# 2. Install dependencies
cd worker && npm install && cd ..

# 3. Configure wrangler.toml
cp worker/wrangler.toml.example worker/wrangler.toml

# 4. Create a KV namespace
npx wrangler kv namespace create dispatcher_kv
#   → Copy the namespace ID from the output and paste it into worker/wrangler.toml
#     under id = "..."

# 5. Create projects.json with your repo/chat mappings
cp projects.example.json projects.json
#   → Edit projects.json (list each repo and its authorized chat IDs)

# 6. Seed the KV namespace
./scripts/seed-kv.sh <your-kv-namespace-id>

# 7. Set secrets
cd worker
npx wrangler secret put GITHUB_TOKEN
npx wrangler secret put WEBHOOK_SECRET

# 8. Deploy
npm run deploy
```

### Register webhooks (one-time)

After deploying, register the worker URL for each bot in your `projects.json`:

```bash
curl -X POST https://your-worker.username.workers.dev/register-all \
  -H "X-Setup-Token: <WEBHOOK_SECRET>"
```

This iterates all projects from KV and calls Telegram's `setWebhook` for each bot token. After this, your bots will stop accepting `getUpdates` polling and start sending webhooks to the worker.

---

## Worker API

### `POST /webhook`

Called by Telegram when a user sends a message. The `X-Telegram-Bot-Api-Secret-Token` header is matched against each project's `webhook_secret` in KV to determine which repo should receive the dispatch.

If the message contains a bot command from a chat that maps to the resolved project, the worker dispatches a `repository_dispatch` event to that repo and responds with `200 OK`.

### `POST /register-all`

Registers the webhook for every project configured in KV. Expects a `X-Setup-Token` header matching `WEBHOOK_SECRET`. Calls `setWebhook` for each bot token with the worker URL and returns a JSON array of results.

```bash
curl -X POST https://your-worker.username.workers.dev/register-all \
  -H "X-Setup-Token: <WEBHOOK_SECRET>"
```

### `POST /flush`

Clears the in-memory KV config cache, forcing the next request to re-read from KV. Expects a `X-Setup-Token` header matching `WEBHOOK_SECRET`. Useful after updating the KV config so changes take effect immediately instead of waiting up to 60 seconds.

```bash
curl -X POST https://your-worker.username.workers.dev/flush \
  -H "X-Setup-Token: <WEBHOOK_SECRET>"
```

---

## Environment reference

### Worker bindings (in `wrangler.toml`)

The repo ships with a `wrangler.toml.example` (template for new projects). Consumer projects should copy the example and edit:

```bash
cp worker/wrangler.toml.example worker/wrangler.toml
```

| Binding | Type | Description |
|---------|------|-------------|
| `DISPATCHER_KV` | KV namespace | Stores the project routing config (key `"projects"`, JSON array of `{ repo, chat_ids, bot_token, webhook_secret }`) |

### Worker secrets (`wrangler secret put`)

| Secret | Description |
|--------|-------------|
| `GITHUB_TOKEN` | GitHub PAT with `repo` scope (needs access to every configured repo for `repository_dispatch`) |
| `WEBHOOK_SECRET` | Random string used to protect admin endpoints (`/flush`, `/register-all`) |

Bot tokens and per-bot webhook secrets are stored alongside each project in KV (`bot_token` and `webhook_secret` fields in `projects.json`). The `X-Telegram-Bot-Api-Secret-Token` header on incoming webhooks is matched against each project's `webhook_secret` to identify which bot received the message.

---

## Upgrading

See [UPGRADING.md](UPGRADING.md) for migration guides:

- [v1 → v2](UPGRADING.md#v1--v2-env-vars--kv) env vars to KV
- [v2 → v3](UPGRADING.md#v2--v3-single-bot-token--per-project-bot-tokens) global bot token to per-project tokens

---

## Development

```bash
cd worker
npm install
npm run dev          # Start local dev server (wrangler dev)
npm run type-check   # TypeScript check
npm run lint         # ESLint
npm run format       # Prettier auto-fix
npm run format:check # Prettier check (CI)
npm test             # Vitest
npm run deploy       # Deploy to Cloudflare
```

## Code quality & CI

| Tool | What it covers | Enforced in CI |
|------|---------------|----------------|
| [EditorConfig](.editorconfig) | Line endings, charset, trimming | — (editor-side) |
| [ESLint](worker/eslint.config.js) | TypeScript linting | `lint` job |
| [Prettier](worker/prettier.config.js) | Code formatting | `format` job |
| [ShellCheck](https://shellcheck.net) | `handler.sh`, `scripts/*.sh` | `shellcheck` job |

CI runs on every push and PR to `main` with 5 parallel jobs: `type-check`, `lint`, `format`, `test`, `shellcheck`. All must pass before merging.

Dependabot is configured (`.github/dependabot.yml`) to keep npm dev dependencies and GitHub Actions up to date weekly.

## License

This project is licensed under the [MIT License](LICENSE).

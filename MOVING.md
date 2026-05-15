# Moving NanoClaw to a New Machine

This documents what is and isn't in the repo, and the exact steps to get a
working install on a new machine from a git clone.

## What the repo does NOT contain

These files are gitignored and must be transferred manually:

| Path | What it is |
|---|---|
| `.env` | API keys, `ONECLI_URL`, `WEBHOOK_PORT`, channel credentials |
| `data/v2.db` | Central database — users, messaging groups, agent wiring |
| `data/v2-sessions/` | Per-session SQLite DBs (can be left behind; agents re-create on first message) |
| `groups/` | Per-agent CLAUDE.md, memory, and installed skills |
| `store/` | Channel auth state (e.g. WhatsApp Baileys keystore — only if using WhatsApp) |

## Prerequisites on the new machine

- **Node.js** ≥ 20 (NVM recommended)
- **pnpm** (enabled via corepack: `corepack enable`)
- **Docker Desktop** ≥ 4.26 (ships Compose v2.24+) — older versions fail to
  start OneCLI due to an `env_file` syntax incompatibility
- **Python** 3.12 or earlier for native addon builds. Python 3.14 (Homebrew)
  has a broken `pyexpat` against the macOS system libexpat. If Python 3.14 is
  the system default, add to `.npmrc`:
  ```
  python=/opt/homebrew/bin/python3.12
  ```

## Steps

### 1. Clone and install

```bash
git clone <your-repo-url> nanoclaw-v2
cd nanoclaw-v2
pnpm install
pnpm run build
```

### 2. Copy the non-repo files

From the old machine (adjust paths as needed):

```bash
scp old-machine:/path/to/nanoclaw-v2/.env .
scp -r old-machine:/path/to/nanoclaw-v2/data/v2.db data/
scp -r old-machine:/path/to/nanoclaw-v2/groups .
```

Or copy manually via USB/cloud storage. The `data/v2-sessions/` directory can
be skipped — sessions are stateless and will be recreated on first message.

### 3. Build the agent container

```bash
./container/build.sh
```

### 4. Install OneCLI

OneCLI is the credential gateway — it is not in the repo.

```bash
curl -fsSL onecli.sh/install | sh   # gateway (Docker-based)
curl -fsSL onecli.sh/cli/install | sh  # CLI binary
```

Then point the CLI at the local gateway:

```bash
onecli config set api-host http://127.0.0.1:10254
```

Re-add your Anthropic API key to the vault:

```bash
onecli secrets create --name anthropic --value sk-ant-... --host-pattern "api.anthropic.com"
```

Check the OneCLI agent that NanoClaw will use and make sure it can access the
secret (the agent is created on first container spawn):

```bash
onecli agents list
onecli agents set-secret-mode --id <agent-id> --mode all
```

### 5. Register the system service

**macOS (launchd)**

Edit the plist at `~/Library/LaunchAgents/com.nanoclaw-v2-*.plist` and update
the `WorkingDirectory` path, then:

```bash
launchctl load ~/Library/LaunchAgents/com.nanoclaw-v2-*.plist
```

Or re-run the setup script and choose "register service":

```bash
bash nanoclaw.sh
```

**Linux (systemd)**

```bash
systemctl --user start nanoclaw
```

### 6. Update the Telegram webhook (if your IP/domain changed)

Telegram webhooks are registered to a specific URL. If the new machine has a
different public address, re-register:

```bash
curl "https://api.telegram.org/bot<TOKEN>/setWebhook?url=https://<new-host>/webhook/telegram"
```

Or re-run `/add-telegram` from a Claude Code session inside the repo.

## Verify

```bash
tail -f logs/nanoclaw.log
```

You should see `NanoClaw running` and `Channel adapter started channel=telegram`.
Send yourself a message on Telegram to confirm end-to-end routing works.

# try-hermes

A minimal Docker Compose setup for trying [Hermes Agent](https://github.com/NousResearch/hermes-agent) — Nous Research's self-improving CLI agent — without installing it on the host.

This repo runs the official prebuilt image (`nousresearch/hermes-agent`) as a long-running **gateway** plus a localhost-bound **dashboard**.

## Prerequisites

- Docker Engine 24+ with Compose v2
- ~5 GB free disk (image + state)
- An API key for at least one supported provider (OpenRouter, OpenAI, Anthropic, Nous Portal, etc.)

## One-time setup

### 1. Configure host UID/GID

So state files in `~/.hermes` are owned by your user:

```bash
cp .env.example .env
printf 'HERMES_UID=%s\nHERMES_GID=%s\n' "$(id -u)" "$(id -g)" > .env
```

### 2. Create the data directory

```bash
mkdir -p ~/.hermes
```

### 3. Run the interactive setup wizard

The wizard collects API keys and writes them to `~/.hermes/.env`. It needs a real TTY, so it runs **outside** of compose:

```bash
docker run -it --rm \
  -v ~/.hermes:/opt/data \
  -e HERMES_UID="$(id -u)" \
  -e HERMES_GID="$(id -g)" \
  nousresearch/hermes-agent setup
```

Follow the prompts: pick a provider, paste your API key, choose a default model.

## Run it

```bash
docker compose up -d
```

- **Gateway** runs in the background and serves messaging adapters.
- **Dashboard** is at <http://127.0.0.1:9119> (localhost-only).

Check logs:

```bash
docker compose logs -f gateway
```

## Use the CLI

The Hermes TUI needs an interactive terminal, so exec into the running gateway:

```bash
docker exec -it hermes hermes
```

Other useful commands inside the container:

| Command | Purpose |
|---|---|
| `docker exec -it hermes hermes model` | Pick / switch model |
| `docker exec -it hermes hermes tools` | Enable/disable tools |
| `docker exec -it hermes hermes config set` | Edit config |
| `docker exec -it hermes hermes doctor` | Health check |

## Stop / upgrade / reset

```bash
# Stop
docker compose down

# Upgrade to the latest image
docker compose pull && docker compose up -d

# Wipe all state (irreversible — deletes API keys, sessions, memory, skills)
docker compose down
rm -rf ~/.hermes
```

## What's where

| Path | Contents |
|---|---|
| `~/.hermes/.env` | API keys (created by the setup wizard) |
| `~/.hermes/config.yaml` | Configuration |
| `~/.hermes/sessions/` | Conversation history |
| `~/.hermes/memories/` | Agent-curated long-term memory |
| `~/.hermes/skills/` | Installed/learned skills |

## Caveats

- **One gateway per data dir.** Hermes' session and memory stores have no concurrent-write protection — never run two gateway containers against the same `~/.hermes`. For multiple profiles, use distinct host directories.
- **Dashboard exposes API keys.** The dashboard is intentionally bound to `127.0.0.1`. For remote access, tunnel it: `ssh -L 9119:localhost:9119 <host>`. Do not expose it on a LAN without auth.
- **OpenAI-compatible API server is off.** If you enable it, set `API_SERVER_KEY` (≥8 chars) in the gateway service env and add a port mapping.
- **Messaging adapters (Telegram/Discord/Slack/etc.)** are configured via `~/.hermes/.env` or `hermes config set` after first run — see the [official messaging docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging).
- **Browser tools (Playwright).** `shm_size: 1gb` is set in compose to avoid Chromium crashes.
- **WSL2.** Works as long as Docker Desktop's WSL integration is enabled for your distro.

## References

- [Hermes Agent repo](https://github.com/NousResearch/hermes-agent)
- [Official Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- [Docker Hub image](https://hub.docker.com/r/nousresearch/hermes-agent)

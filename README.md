# try-hermes

A minimal Docker Compose setup for trying [Hermes Agent](https://github.com/NousResearch/hermes-agent) — Nous Research's self-improving CLI agent — without installing it on the host.

This repo runs the official prebuilt image (`nousresearch/hermes-agent`) as a long-running **gateway** plus a localhost-bound **dashboard**, alongside an **Ollama** service for local LLMs. State lives in `./volumes/{hermes,ollama}` next to the compose file.

## Prerequisites

- Docker Engine 24+ with Compose v2
- ~10 GB free disk (images + state; local models add more)
- **One** of:
  - An API key for a cloud provider (OpenRouter, OpenAI, Anthropic, Nous Portal, etc.), **or**
  - An NVIDIA GPU + [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html) if you want usable speeds from the bundled Ollama service (CPU works but is slow)

## One-time setup

### 1. Configure host UID/GID

So state files in `./volumes/hermes` are owned by your user:

```bash
cp .env.example .env
printf 'HERMES_UID=%s\nHERMES_GID=%s\n' "$(id -u)" "$(id -g)" > .env
```

### 2. Run the interactive setup wizard

The wizard collects API keys and writes them to `./volumes/hermes/.env`. It needs a real TTY, so it runs **outside** of compose:

```bash
docker run -it --rm \
  -v "$(pwd)/volumes/hermes:/opt/data" \
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

## Using a local model via Ollama

After `docker compose up -d`, pull a model into the Ollama container. Hermes requires a context window of **≥64K tokens**, so pick a model that supports it (e.g. Qwen 2.5, Llama 3.1/3.3, Mistral Large):

```bash
docker exec -it ollama ollama pull qwen2.5:14b
```

Point Hermes at the in-network Ollama hostname:

```bash
docker exec -it hermes hermes model
```

Pick **Custom endpoint**, then enter:

| Field | Value |
|---|---|
| API base URL | `http://ollama:11434/v1` |
| API key | `ollama` (any non-empty string) |
| Model name | `qwen2.5:14b` (must match what you pulled) |
| Context length | `65536` (or whatever the model supports) |

The `ollama` hostname only resolves from inside the compose network — that's why the gateway can reach it but `localhost` can't.

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
find volumes/hermes -mindepth 1 ! -name .gitkeep -exec rm -rf {} +
```

## What's where

| Path | Contents |
|---|---|
| `volumes/hermes/.env` | API keys (created by the setup wizard) |
| `volumes/hermes/config.yaml` | Configuration |
| `volumes/hermes/sessions/` | Conversation history |
| `volumes/hermes/memories/` | Agent-curated long-term memory |
| `volumes/hermes/skills/` | Installed/learned skills |
| `volumes/ollama/` | Ollama model blobs and manifests |

The whole `volumes/hermes/` and `volumes/ollama/` trees are gitignored (except `.gitkeep`) so secrets, session data, and large model files stay out of git.

## Caveats

- **One gateway per data dir.** Hermes' session and memory stores have no concurrent-write protection — never run two gateway containers against the same `volumes/hermes` directory. For multiple profiles, use distinct host directories (or separate clones of this repo).
- **Dashboard exposes API keys.** The dashboard is intentionally bound to `127.0.0.1`. For remote access, tunnel it: `ssh -L 9119:localhost:9119 <host>`. Do not expose it on a LAN without auth.
- **OpenAI-compatible API server is off.** If you enable it, set `API_SERVER_KEY` (≥8 chars) in the gateway service env and add a port mapping.
- **Messaging adapters (Telegram/Discord/Slack/etc.)** are configured via `~/.hermes/.env` or `hermes config set` after first run — see the [official messaging docs](https://hermes-agent.nousresearch.com/docs/user-guide/messaging).
- **Browser tools (Playwright).** `shm_size: 1gb` is set in compose to avoid Chromium crashes.
- **Ollama GPU passthrough.** The compose file requests an NVIDIA GPU for Ollama. If you don't have one (or don't have NVIDIA Container Toolkit installed), delete the `deploy:` block on the `ollama` service — it will fall back to CPU. Hermes' ≥64K context requirement means models need real horsepower to be usable.
- **Ollama context window.** The OpenAI-compatible endpoint defaults to a smaller context than the model supports. Set `OLLAMA_CONTEXT_LENGTH` (env on the `ollama` service) or use a Modelfile if you need the full window.
- **WSL2.** Works as long as Docker Desktop's WSL integration is enabled for your distro.

## References

- [Hermes Agent repo](https://github.com/NousResearch/hermes-agent)
- [Official Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker)
- [Hermes Docker Hub image](https://hub.docker.com/r/nousresearch/hermes-agent)
- [Hermes Ollama integration docs](https://docs.ollama.com/integrations/hermes)
- [Ollama Docker Hub image](https://hub.docker.com/r/ollama/ollama)

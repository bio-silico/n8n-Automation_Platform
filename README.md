# n8n-Automation_Platform — GeneMolX AI LLC

Install and configure **self-hosted n8n** on your own machine — with **Agents**, the **n8n Assistant**
(running AI-written code in a local sandbox) and **AI connections** to your local models (LM Studio,
Ollama) as well as cloud models.

Everything runs on your machine: n8n is bound to `127.0.0.1`, telemetry is off, and nothing is sent to
n8n's servers.

**Scope: installation and setup.** Building workflows and agents is not covered here.

## Part of the GeneMolX series

| Repository | Layer |
|---|---|
| [Local_AI_Setups](https://github.com/bio-silico/Local_AI_Setups) | the foundation — machine, GPU, local models with Ollama and LM Studio |
| [AI-Brain_Build](https://github.com/bio-silico/AI-Brain_Build) | the database brain — knowledge, memory and data |
| **n8n-Automation_Platform** | **the automation platform** — n8n, Agents, Assistant *(this repository)* |

This repository continues where the foundation stops: the models you installed there, now reachable
from n8n.

## What you get

| File | What it is |
|---|---|
| [`INSTALL.md`](INSTALL.md) | **the guide** — 9 steps, each ending with a check |
| [`stack/docker-compose.yml`](stack/docker-compose.yml) | all services and settings, every security choice commented |
| [`stack/.env.example`](stack/.env.example) | versions and secrets template — placeholders only |
| [`stack/.gitignore`](stack/.gitignore) | keeps `.env` and your data out of git |

| Service | Role |
|---|---|
| `n8n` | editor, workflows, Agents, Assistant — on `127.0.0.1` only |
| `runners` | runs Code-node code in a separate container |
| sandbox (optional) | runs code written by the n8n Assistant |

## What it needs

- Linux host with **Docker Engine** and **Docker Compose v2**
- `openssl`, to generate the secrets
- RAM: n8n about 650 MB; the optional sandbox adds 0.5–4 GB
- Internet access for the first image downloads

Tested on Ubuntu 24.04, Docker Engine 29, Docker Compose v2.39 (2026-09-28), n8n **2.40.7**.

## Local and cloud models — both work

Neither LM Studio nor Ollama appears in n8n's provider list. Both connect through the **OpenAI
credential type with a custom Base URL**, which the guide walks through:

| | Where it runs | Connect with |
|---|---|---|
| Local models | LM Studio or Ollama, on your machine | OpenAI credential + Base URL `http://host.docker.internal:1234/v1` (LM Studio) or `:11434/v1` (Ollama) |
| Cloud models | OpenAI, Anthropic | their own credential types |

**One current limit, worth knowing before you start.** With local models in the new **Agents**
(preview in n8n 2.40.7), the first message works and follow-ups fail: Agents use OpenAI's Responses
API, which local servers don't support. For multi-turn chat with a local model, use the classic
**AI Agent node** in a workflow — that works. The guide explains the cause in full.

## Security built into the install

The compose file is deliberately **not** n8n's reference example. Each choice is commented in the
file:

- the editor and webhooks listen on **this machine only**, not the LAN
- image versions are **pinned**; `latest` is never used
- **each container gets only the secrets it needs** — your model keys and the encryption key never
  reach the sandbox containers
- **no calls to n8n's servers**: telemetry, version checks and templates off
- the public API and the MCP server are off
- the Assistant receives **field names and types, not your data values**
- Code runs in **separate runner containers**, not inside n8n
- the sandbox's TLS keys are private — n8n's own example makes them world-readable

> **The Assistant's sandbox runs privileged** (Docker-in-Docker), which is equivalent to root on the
> host. Start it while you are developing, stop it afterwards. n8n, your workflows and Agents all work
> without it.

## Install

Follow **[INSTALL.md](INSTALL.md)** — images, secrets, first start, owner account and 2FA, AI
credentials, the optional Assistant sandbox, operations and backup.

The shape of it:

```bash
mkdir -p ~/n8n && cd ~/n8n
# copy stack/docker-compose.yml, stack/.env.example and stack/.gitignore here
cp .env.example .env && chmod 600 .env     # then generate the secrets — step 3 of the guide
docker compose up -d
curl -s http://127.0.0.1:5678/healthz      # {"status":"ok"}
```

Then open `http://localhost:5678`, create the owner account and turn on 2FA.

**Running it day to day** — start, stop, restart, the one-line health check and what to do when it
fails: [INSTALL.md → Operations](INSTALL.md#operations). Nothing starts by itself after a reboot.

**Adjust for your machine:** set `TZ` in `.env` (defaults to `UTC`), change `N8N_HOST_PORT` if 5678 is
taken, and if another n8n already runs on the host, change `name:` and the `container_name:` values.

## Disclaimer

- **Provided as is, without warranty.** Test it on your own system first.
- **Your data, your responsibility** — secrets, backups, and what you connect n8n to.
- **Back up `.env`.** The encryption key in it decrypts every credential stored in n8n; lose it and
  they are unreadable.
- **No GeneMolX data.** Every value here is a placeholder.
- **n8n is not included.** This repository holds our compose file and guide; Docker pulls n8n's own
  images, which carry n8n's licence and terms.

Need something custom? GeneMolX AI LLC helps with individual setups — see
[genemolx.com](https://www.genemolx.com).

## Licence

© 2026 GeneMolX AI LLC — licensed under the [Apache License 2.0](LICENSE).

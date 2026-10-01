# n8n self-hosted setup — Docker, Agents, n8n Assistant, AI connections

Step-by-step guide to download, install and configure **n8n 2.40.7** with Docker Compose, the new **Agents** and **n8n Assistant** features (with a local code sandbox), and AI model connections (local and cloud).

Tested on Ubuntu 24.04, Docker Engine 29, Docker Compose v2.39 (2026-09-28).

Scope: installation, setup, AI connections and credentials. Not covered: building workflows or agents.

---

## What you get

| Service | Image | Starts with | Role |
|---|---|---|---|
| `n8n` | `n8nio/n8n:2.40.7` | `docker compose up` | editor, workflows, Agents, Assistant — on `127.0.0.1` only |
| `runners` | `n8nio/runners:2.40.7` | `docker compose up` | runs Code-node code in a separate container |
| `sandbox-certs` | `ghcr.io/n8n-io/n8n-sandbox-service-api:1.5.0` | profile `assistant` | one-shot: creates TLS certificates, then exits |
| `sandbox-api` | same | profile `assistant` | sandbox control service (internal network only) |
| `sandbox-runner-1` | `ghcr.io/n8n-io/n8n-sandbox-service-runner-dind:1.5.0` | profile `assistant` | runs code written by the n8n Assistant — **privileged** |

- The sandbox is needed **only for the n8n Assistant**. n8n, workflows and Agents work without it.
- Data: n8n's own database is SQLite in `./n8n_data/`.
- No telemetry and no calls to n8n's servers (see [Security choices](#security-choices)).

## Files in this repository

| File | Purpose |
|---|---|
| `stack/docker-compose.yml` | all services and settings |
| `stack/.env.example` | template for versions and secrets → copy to `.env` |
| `stack/.gitignore` | keeps `.env` and `n8n_data/` out of git |

---

## Requirements

- Linux host with **Docker Engine** and **Docker Compose v2** (`docker compose version`)
- `openssl` (to generate secrets)
- RAM: n8n ~650 MB; with the sandbox running add ~0.5–4 GB (runner capped at 4 GB)
- Internet access for the first image downloads (Docker Hub, ghcr.io)
- The host user should have **uid 1000** (same as the `node` user inside the n8n image) — check with `id -u`. Otherwise adjust ownership of `n8n_data/` in step 4.

---

## Step 1 — Get the files

```bash
mkdir -p ~/n8n && cd ~/n8n
# copy stack/docker-compose.yml, stack/.env.example and stack/.gitignore from this repository into this folder
```

Running another n8n on the same host? Change `name:` in `stack/docker-compose.yml` (project name), the `container_name:` values, and `N8N_HOST_PORT` in `.env` so nothing collides.

## Step 2 — Download the n8n images

```bash
docker pull n8nio/n8n:2.40.7
docker pull n8nio/runners:2.40.7        # must be the same version as n8n
```

Always use a version tag, never `latest` — an accidental pull of `latest` means an unplanned upgrade.
Latest stable version: `https://github.com/n8n-io/n8n/releases` (or `npm view n8n dist-tags`).

## Step 3 — Create `.env` with secrets

```bash
cp .env.example .env
chmod 600 .env
for k in N8N_ENCRYPTION_KEY N8N_RUNNERS_AUTH_TOKEN SANDBOX_API_KEY \
         SANDBOX_RUNNER_REGISTRATION_TOKEN SANDBOX_RUNNER_API_KEY; do
  sed -i "s|^$k=change-me\$|$k=$(openssl rand -hex 32)|" .env
done
grep -c change-me .env      # must print 0
```

> **Back up `.env`.** `N8N_ENCRYPTION_KEY` encrypts every credential stored in n8n. Lose it → all credentials are unreadable.

## Step 4 — Create the data folder

```bash
mkdir -m 700 n8n_data      # owned by uid 1000; never chmod 777
```

## Step 5 — Validate the configuration

```bash
docker compose config -q && echo valid
docker compose --profile assistant config -q && echo "valid with sandbox"
```

## Step 6 — Start n8n and the task runners

```bash
docker compose up -d
docker compose ps
curl -s http://127.0.0.1:5678/healthz          # {"status":"ok"}
docker compose logs n8n | grep -E "Version:|Registered runner|Editor is now"
```

Expected in the log: `Version: 2.40.7`, `Registered runner "launcher-javascript"` and `"launcher-python"`.

## Step 7 — First login: owner account and 2FA

1. Open `http://localhost:5678` **on the same host** (the port is bound to `127.0.0.1`).
2. Create the owner account.
3. **Turn on 2FA**: Settings → Personal → Two-factor authentication → Enable → scan the QR code with an authenticator app → **save the recovery codes offline**.

> Enforcing 2FA for all users (`N8N_MFA_ENFORCED_ENABLED`) needs a Business license — on the free Community edition it is silently ignored. Each user must enable 2FA themselves.

If the assistant setup wizard appears, choose **"Set up later in Settings"** (the sandbox is not running yet).

---

## Step 8 — AI connections and credentials

Credentials are stored encrypted in n8n's database (with `N8N_ENCRYPTION_KEY`).
For every credential with a real key, set **Allowed HTTP Request Domains → Specific domains** to the provider's domain, so the key can't be sent anywhere else.

### 8a. Local models — LM Studio or Ollama

Neither is a provider in the n8n **Agents** list. Use the **OpenAI** credential type with a custom **Base URL**:

| Field | LM Studio | Ollama |
|---|---|---|
| Credential type | OpenAI | OpenAI |
| API Key | any text (e.g. `lm-studio`) | any text (e.g. `ollama`) |
| Base URL | `http://host.docker.internal:1234/v1` | `http://host.docker.internal:11434/v1` |
| Allowed domains | `host.docker.internal` | `host.docker.internal` |
| Model | the model id, e.g. `google/gemma-4-12b-qat` | e.g. `mistral:latest` |

Check the n8n container can reach the server (lists models, runs nothing):

```bash
docker compose exec n8n wget -qO- http://host.docker.internal:1234/v1/models     # LM Studio
docker compose exec n8n wget -qO- http://host.docker.internal:11434/api/tags     # Ollama
```

In LM Studio, load the model with a **large context** (e.g. 60k tokens): an agent's built-in prompt alone is ~4,200 tokens.

> **Known limit (n8n 2.40.7 Agents, preview):** with LM Studio (0.4.23) the **first message works, follow-up messages fail** (`Invalid type for 'input'`, HTTP 400). n8n Agents use OpenAI's Responses API and send `item_reference` items (links to replies stored server-side); local servers don't store replies and reject them. There is no n8n setting to avoid it. Ollama is expected to behave the same. For multi-turn chat with local models use the classic **AI Agent node in a workflow**, which works.

### 8b. Cloud models — OpenAI / Anthropic

| Provider | Credential type | Allowed domain |
|---|---|---|
| OpenAI | OpenAI | `api.openai.com` |
| Anthropic | Anthropic | `api.anthropic.com` |

### 8c. Test an agent

Agents (Preview) → **Create agent** → Model: pick the credential and model → write **Instructions** (required) → **Preview** → send a message.

---

## Step 9 — n8n Assistant (optional, uses the sandbox)

The n8n Assistant builds and debugs workflows from a chat. It runs AI-written code in the sandbox.

> **Security:** `sandbox-runner-1` is `privileged: true` (Docker-in-Docker) — equivalent to root on the host. It has no host ports and no host folders mounted, but run it **only while developing** and stop it afterwards. A non-privileged alternative is the Sysbox runtime (host change + Docker restart).

### 9a. Start the sandbox

```bash
docker compose --profile assistant up -d sandbox-certs sandbox-api sandbox-runner-1
docker compose ps
docker compose exec n8n wget -qO- http://sandbox-api:8080/healthz       # {"status":"ok"}
docker compose logs sandbox-api | grep "runner registered"
```

The first start downloads the sandbox images from ghcr.io; the runner downloads the per-run sandbox image on first use.

### 9b. Connect the model (in the UI)

Settings → **n8n Assistant** → Model → **Connect**:

1. Provider: OpenAI, Anthropic or OpenRouter; paste the API key.
2. **Model field: it is pre-filled with n8n's suggestion** (e.g. `gpt-5.6-sol`), which your account may not have access to (`Project … does not have access to model …`). Clear it (Ctrl+A, Delete) and **type a model your account can use**.
3. Save → "Enable n8n Assistant" shows **Enabled**.

> Do not set `N8N_INSTANCE_AI_MODEL*` in `stack/docker-compose.yml` or `.env`: any env value **locks** the model field in the UI.
> A local model is possible (`custom/<model>` + `N8N_INSTANCE_AI_MODEL_URL`), but the Assistant's internal prompts reach 30–40k tokens → needs a large context and tool calling.

### 9c. Recommended settings (same page)

- **Web search**: *Disable* (unless you add a SearXNG container).
- **Permissions**: set **Credentials** and **System** to **Ask first**.
- **Data sent to the model**: already limited by `N8N_AI_ALLOW_SENDING_PARAMETER_VALUES=false` (field names/types only, no data values). The UI "AI usage settings → Configure" link does not work in this setup.

### 9d. Use it

- Left sidebar **✨** icon → full Assistant chat.
- Inside a workflow or agent: **✨** button top right.
- Start small (e.g. "create a workflow with a webhook that returns hello") and review what it builds — it is a Preview feature.

### 9e. Stop / start the sandbox

```bash
docker compose --profile assistant stop sandbox-runner-1 sandbox-api     # stop after developing
docker compose --profile assistant start sandbox-api sandbox-runner-1    # start again
```

Stopping wipes running sandboxes (normal); Assistant settings stay in n8n.

---

## Operations

Everything here runs in a shell, from the folder holding `docker-compose.yml` and `.env` — no AI assistant needed.

**Is it running?** One line, one clear answer:

```bash
docker compose ps --format '{{.Service}}: {{.State}}' && \
curl -fsS -m 5 http://127.0.0.1:5678/healthz >/dev/null && echo "✅ n8n is running" || echo "❌ n8n is NOT responding"
```

Use your own port if `N8N_HOST_PORT` is not 5678. If the answer is ❌, read the last log lines:

```bash
docker compose logs --tail 50 n8n
```

**Start, stop, restart:**

```bash
docker compose up -d                               # start — also after a host reboot
docker compose restart n8n                         # restart n8n alone (e.g. it hangs)
docker compose --profile assistant down            # stop everything (data kept in ./n8n_data)
docker compose up -d n8n                           # apply docker-compose.yml changes (recreate, ~10-20 s)
docker compose logs -f n8n                         # live log
```

> **Nothing starts by itself.** The services use `restart: "no"`, so after a reboot or an outage you start them yourself with `docker compose up -d`.

**Assistant sandbox** — only while developing (see [step 9e](#9e-stop--start-the-sandbox)):

```bash
docker compose --profile assistant start sandbox-api sandbox-runner-1
docker compose --profile assistant stop sandbox-runner-1 sandbox-api
```

**The host's IP address changed?** Nothing to do for n8n: the editor is on `127.0.0.1` and the AI credentials use `host.docker.internal`, not the LAN address. Only a credential you created with a fixed LAN IP (e.g. `http://192.168.x.x:1234/v1`) needs editing in n8n → Credentials.

**Backup**: `n8n_data/` + `.env` (the encryption key). While n8n runs, copy the database with `sqlite3 n8n_data/database.sqlite ".backup 'backup.sqlite'"` rather than a plain file copy (WAL mode).

**Update n8n**: back up first → set `N8N_VERSION` in `.env` (n8n and runners use the same value) → `docker compose pull && docker compose up -d`. A newer version migrates the database; going back needs the backup. Sandbox: set `N8N_SANDBOX_VERSION` (api, runner and sandbox images share one release, `https://github.com/n8n-io/n8n-sandbox-service/releases`).

---

## Security choices

Every setting below is in `stack/docker-compose.yml`, with a comment.

| Choice | Why |
|---|---|
| Port bound to `127.0.0.1` | editor/webhooks not reachable from LAN or IPv6; expose deliberately (reverse proxy + HTTPS) if needed |
| Pinned image versions | no surprise upgrades |
| `N8N_ENCRYPTION_KEY` from `.env` | key not only inside the data folder |
| Secrets passed per container (no shared `env_file`) | model keys / encryption key never reach sandbox containers |
| Sandbox TLS keys `0600` | n8n's reference compose makes them world-readable (`--world-readable`) — not used here |
| Log rotation 3 × 20 MB | container logs don't grow forever |
| `runtime: runc` on all services | avoids inheriting a host default runtime (e.g. `nvidia`) |
| Telemetry, version checks, templates off; `mcp-registry`, `community-packages`, `quick-connect` modules off | no calls to `api.n8n.io` / telemetry |
| Public API off, MCP server off | smaller attack surface |
| Assistant: browser-use off, local gateway off, 2 concurrent runs | the AI can't drive users' browsers or PCs |
| `N8N_AI_ALLOW_SENDING_PARAMETER_VALUES=false` | AI gets field names/types, not data values |
| Code node in external runners | internal mode is "insecure by design" and deprecated |
| Sandbox runner capped: 4 GB RAM, 4 CPUs, max 5 sandboxes | a runaway sandbox can't starve the host |
| Execution retention 5000 h, default timeout 900 s | set on purpose, not defaults |

## Lessons learned

- **Verify every setting takes effect** (database or behavior), not only that it's in the compose file. Examples: 2FA enforcement silently ignored on Community; `N8N_INSTANCE_AI_MODEL` locking the UI.
- **Don't copy n8n's reference compose as-is**: it passes the whole `.env` to the sandbox containers and makes TLS keys world-readable, against its own security checklist.
- **Preview features** (Agents, Assistant, Chat) can change or break between versions — review before relying on them for production.
- **Agents knowledge base** needs the Daytona cloud sandbox — not available in a local-only setup.

## Open items

- `/api/v1/workflows` answers `401` instead of `404` although `N8N_PUBLIC_API_DISABLED=true` — not investigated yet.
- Not included: SSRF protection (`N8N_SSRF_PROTECTION_ENABLED`; needs an allowlist for local AI/DB services), SearXNG web search, Sysbox.

## References

- n8n Docker Compose install: https://docs.n8n.io/deploy/host-n8n/install-options/install-using-docker-compose
- Set up n8n Assistant: https://docs.n8n.io/deploy/host-n8n/configure-n8n/set-up-n8n-assistant
- Task runners: https://docs.n8n.io/deploy/host-n8n/configure-n8n/set-up-task-runners
- Isolate n8n (no calls to n8n servers): https://docs.n8n.io/deploy/host-n8n/configure-n8n/basic-configuration/configuration-examples/isolate-n8n
- Sandbox service security model: https://github.com/n8n-io/n8n-sandbox-service/blob/main/docs/security-model.md

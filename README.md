# local-ai-stack

A self-contained, self-hosted **local AI stack** delivered as a single Docker Compose project. It pairs a local large‑language‑model inference engine with a full‑featured web interface and API, so all model serving happens on your own hardware — no cloud LLM provider required.

| Service    | Image                            | Role                                                                      |
| ---------- | -------------------------------- | ------------------------------------------------------------------------- |
| Ollama     | `ollama/ollama`                  | Runs and manages local LLMs; exposes the inference API (container port 11434) |
| Open WebUI | `ghcr.io/open-webui/open-webui`  | Chat UI + admin console + **OpenAI‑compatible API**, backed by Ollama (container port 8080) |

> **Status:** All inference is local. Models are downloaded once and then served from disk at runtime. Outbound traffic is limited to image pulls, model downloads, and (in Traefik mode) certificate issuance.
>
> **Scope of this repo:** exactly four files — `docker-compose.yaml`, `.env.example`, `.gitignore`, and this `README.md`. Deployment is plain `docker compose`; no CI tooling (Jenkins, ArgoCD, etc.) ships in or is assumed by this project.

---

## Table of Contents

1. [Architecture](#architecture)
2. [Repository Layout](#repository-layout)
3. [Prerequisites](#prerequisites)
4. [Getting Started (Option A — Local)](#getting-started-option-a--local)
5. [Traefik Deployment (Option B)](#traefik-deployment-option-b)
6. [Configuration Reference](#configuration-reference)
7. [Working with Models](#working-with-models)
8. [API & Endpoints](#api--endpoints)
9. [Data Persistence & Backup](#data-persistence--backup)
10. [Security Considerations](#security-considerations)
11. [Operations & Troubleshooting](#operations--troubleshooting)
12. [References](#references)

---

## Architecture

```
                         Option B (recommended)                Option A (local dev)
                     ┌───────────────────────┐               ┌───────────────────────┐
                     │   Domain: <DOMAIN>    │               │  Host: <WEBUI_PORT>   │
                     │   TLS via Traefik     │               │  Direct port publish  │
                     └───────────┬───────────┘               └───────────┬───────────┘
                                 │ HTTP/HTTPS                            │
                       ┌─────────▼─────────┐                             │
                       │  Traefik (ext.)   │         ┌───────────────────│─────┐
                       │  network: EXTERNAL│◄────────┤  ai_network       │     │
                       └─────────┬─────────┘ (Option B)└──┬────────────┴───┬──┘
                                 │                         │               │ :8080
                         ┌───────▼───────┐           ┌─────▼─────────┐   ┌─▼───────────────┐
                         │ open-webui    │           │ shared compose│   │  open-webui     │
                         │ :8080 (chat,  │           │ network       │   │  (same image)   │
                         │  API, admin)  │           └───────────────┘   └─┬───────────────┘
                         │               │                                 │
                         │ OLLAMA_BASE_  │                                 │
                         │ URL=http://   │                                 │
                         │ ollama:11434  │                                 │
                         └───────┬───────┘                                 │
                                 │  HTTP://OLLAMA:11434 (internal only)    │
                         ┌───────▼───────────┐                             │
                         │   ollama          │◄──────── internal call ──────┘
                         │   :11434          │
                         │   models + config │
                         │   (ollama-data)   │
                         └───────────────────┘
```

**Data flow**

1. A user (or a client tool) talks to **Open WebUI** over HTTP(S) — via the published host port (Option A) or through Traefik routing on `DOMAIN_NAME` with TLS (Option B).
2. Open WebUI forwards chat completions to **Ollama** using the internal service name `ollama` on port `11434`. Ollama is **never exposed directly to the host network**, so the inference API is reachable only from inside the compose network.
3. Both services persist state in named Docker volumes (`ollama-data`, `open-webui-data`), so containers can be recreated on every deploy — infrastructure is treated as cattle, state lives in volumes.

**Key design decisions**

- **Isolated network:** Both containers share a single compose network. In Traefik mode this is an *external* network (default name from `${TRAEFIK_NETWORK}`, commonly `proxy`) so the proxy can reach the app without the app reaching out.
- **Dual deployment modes:** The same Compose project serves both local testing (published port, isolated bridge) and production (reverse proxy) — you flip two commented blocks, nothing else.
- **Hardened containers:** `security_opt: no-new-privileges=true` on every service.
- **Transport‑agnostic:** Deploys with plain `docker compose`. Nothing here assumes Jenkins, ArgoCD, Talos, or any specific CI/CD or virtualization platform — bring your own pipeline if you need one.

---

## Repository Layout

```
local-ai-stack/
├── README.md              # This document
├── docker-compose.yaml    # Service definitions (ollama + open-webui), network, volumes
├── .env.example           # Committed template — copy to .env and fill in real values
└── .gitignore             # Keeps .env (secrets) out of VCS
```

`docker-compose.yaml` defines:

| Construct              | Detail                                                                               |
| ---------------------- | ------------------------------------------------------------------------------------- |
| `services.ollama`      | Image `ollama/ollama`, data volume `ollama-data` → `/root/.ollama`                    |
| `services.open-webui`  | Image `ghcr.io/open-webui/open-webui`, volume `open-webui-data` → `/app/backend/data`, env `OLLAMA_BASE_URL=http://ollama:11434` + `WEBUI_SECRET_KEY` |
| `networks.ai_network`  | **External** network named by `${TRAEFIK_NETWORK}` (Option B) **or** local bridge (Option A — commented out in the checked‑in file) |
| `volumes`              | `ollama-data`, `open-webui-data` (both persistent named volumes)                      |

> Note: `.env` itself is **git‑ignored by design** (see `.gitignore`). Real secrets never enter the repository — you copy them from `.env.example` into a local `.env`, or inject your own at build time in CI.

---

## Prerequisites

**Always required**

- A host running Docker Engine with the **Docker Compose v2 plugin** (the `docker compose` command, not legacy `docker-compose`).
- Sufficient disk space for the models you plan to run (see [Working with Models](#working-with-models)).
- A GPU and compatible driver only if using CUDA‑capable models — otherwise Ollama and Open WebUI fall back to CPU.

**Required for Option B (Traefik)**

- An existing Traefik deployment that already publishes an **external Docker network** whose name you will set in `TRAEFIK_NETWORK` (commonly `proxy`). The Docker host running this project must be attached to that network.
- A **DNS A (or AAAA) record** for `DOMAIN_NAME` pointing at the Traefik host.
- A Traefik TLS **entrypoint** (`TRAEFIK_ENTRYPOINT`) and a **certificate resolver** (`TRAEFIK_CERTRESOLVER`) already configured in Traefik — this project consumes them via labels and does not configure Traefik itself.
- (For Let's Encrypt) public reachability of the port behind that entrypoint, or a Cloudflare‑style DNS-01 resolver.

> **No CI/CD prerequisites.** This project ships without pipeline tooling; every workflow in this document runs with plain `docker compose` on a single node.

---

## Getting Started (Option A — Local)

Use this path for first‑time setup, local testing, or a single‑node deployment without a reverse proxy.

**1. Switch `docker-compose.yaml` to the local mode.** Two edits, both clearly commented in the file:

- In `services.open-webui`: **uncomment the `ports:` block** (`${WEBUI_LOCAL_PORT}:8080`) and **comment out the entire `labels:` (Option B) block**.
- In top‑level `networks.ai_network`: **uncomment the local bridge definition** and **comment out the external network block** so the project creates its own private network.

**2. Prepare the environment file:**

```bash
cp .env.example .env
# Then edit .env: set WEBUI_LOCAL_PORT (default 3000) and generate a strong secret key:
openssl rand -hex 32
```

(Option B variables in `.env` are inert in this mode.)

**3. Deploy and verify:**

```bash
docker compose up -d --remove-orphans
docker compose ps                 # both services should be "running"
```

**4. First‑run steps:**

1. Open `http://<host-ip>:${WEBUI_LOCAL_PORT}` (e.g. `http://localhost:3000`).
2. Create the **administrator account** — done inside Open WebUI, not via env vars.
3. Pull and run a model (see [Working with Models](#working-with-models)). The first request that touches a model also downloads it on demand if you let Open WebUI fetch it.

---

## Traefik Deployment (Option B)

The compose file **ships in Option B mode by default** — `labels` active, external network referenced. If you are deploying without changes, this is the production path.

**1. Ensure the Traefik prerequisites above exist**, then set the four `TRAEFIK_*` / `DOMAIN_NAME` variables in `.env`:

```ini
DOMAIN_NAME=ai.example.com            # DNS record → Traefik host
TRAEFIK_NETWORK=proxy                 # must match the EXISTING external network name
TRAEFIK_ENTRYPOINT=https              # a TLS-capable Traefik entrypoint
TRAEFIK_CERTRESOLVER=letsencrypt     # e.g. letsencrypt, cloudflare (DNS-01)
```

**2. Keep `docker-compose.yaml` as checked in** (Traefik labels + external network active; the `ports:` and local‑network blocks remain commented).

**3. Deploy:**

```bash
docker compose up -d --remove-orphans
```

**4. Verify routing from an outsider's perspective** (not from inside the Docker host):

```bash
curl -vI https://${DOMAIN_NAME}/   # expect TLS handshake + 200/302 from Traefik
```

Inside Traefik you should see a router `open-webui` matching `Host(\`ai.example.com\`)` on your entrypoint, load‑balancing to port `8080` of the `open-webui` container.

> **Switching between options later:** the two `ports`/`labels` pairs and the two network definitions are mutually exclusive. Exactly one of each pair must be active per service. Change them, then run `docker compose up -d --remove-orphans` again.

---

## Configuration Reference

All values are read from `.env` in the project root (Docker Compose auto‑loads it). The committed template is [`.env.example`](.env.example) — copy it to `.env` before first use; never commit real values.

### Image versions

| Variable             | Default (template) | Consumed by   | Purpose / notes                                                                    |
| -------------------- | ------------------ | ------------- | ---------------------------------------------------------------------------------- |
| `OLLAMA_VERSION`     | `latest`           | `ollama`      | Ollama image tag. **Pin to a release tag/digest in production** for reproducible deploys. |
| `OPEN_WEBUI_VERSION` | `main`             | `open-webui`  | Open WebUI image tag. Same pinning advice.                                          |

### Security

| Variable           | Default (template)                        | Consumed by  | Purpose / notes                                                                                                       |
| ------------------ | --------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `WEBUI_SECRET_KEY` | *generate your own* (e.g. `openssl rand -hex 32`) | `open-webui` | Master secret Open WebUI uses for session/JWT signing. **Never reuse across installations.** Changing it invalidates existing sessions/tokens. |

### Option A — local deployment

| Variable           | Default (template) | Consumed by                 | Purpose / notes                                                                        |
| ------------------ | ------------------ | --------------------------- | -------------------------------------------------------------------------------------- |
| `WEBUI_LOCAL_PORT` | `3000`             | compose (`ports`, Option A block only) | Host port Open WebUI's internal `8080` is published on. Change if the port is taken.  |

### Option B — Traefik deployment

| Variable               | Default (template) | Consumed by     | Purpose / notes                                                                                          |
| ---------------------- | ------------------ | --------------- | --------------------------------------------------------------------------------------------------------- |
| `DOMAIN_NAME`          | `ai.example.com`   | compose labels  | Hostname used in the `Host()` router rule. Must resolve to the Traefik host.                                |
| `TRAEFIK_NETWORK`      | `proxy`            | compose `networks` | **Name of an existing external Docker network** owned by Traefik. Compose does not create it — it must already exist. |
| `TRAEFIK_ENTRYPOINT`   | `https`            | compose labels  | Traefik entrypoint the router is attached to.                                                                |
| `TRAEFIK_CERTRESOLVER` | `letsencrypt`      | compose labels  | Traefik cert resolver name; TLS termination happens entirely in Traefik.                                    |

> In Option A, the four Traefik variables are not used. In Option B, `WEBUI_LOCAL_PORT` is not used. Both sets may coexist safely in `.env`.

---

## Working with Models

Model binaries live **inside the `ollama-data` volume** (`/root/.ollama`). They survive container recreation and image re‑pulls.

**Pull from the host (recommended — gives you progress output):**

```bash
docker compose exec ollama ollama pull llama3.2
docker compose exec ollama ollama list         # what's installed
docker compose exec ollama ollama show llama3.2
docker compose exec ollama ollama rm <model>    # remove
```

**Pull through the UI:** the model "pull" affordance in Open WebUI talks to Ollama for you — useful for first‑time setup since the admin flow is one screen.

Practical guidance:

- Start small (`llama3.2` 3B, `qwen2.5` 7B‑class) to validate RAM/CPU throughput before chasing 70B‑class models.
- Check free disk: each model download plus KV‑cache headroom during inference should fit with margin. `du -sh /var/lib/docker/volumes/local-ai-stack_ollama-data/_data` is the quick way to see real footprint.
- Ollama's inference API is reachable **only from within the compose network** (container‑to‑container via the service name `ollama`); there is no host port to lock down. The true attack surface is Open WebUI, which TLS and (optionally) auth cover.

---

## API & Endpoints

**External surface**

| Target     | Option A                              | Option B                  | Notes |
| ---------- | ------------------------------------- | ------------------------- | ----- |
| Open WebUI | `http://<host>:${WEBUI_LOCAL_PORT}`   | `https://${DOMAIN_NAME}/` | Chat UI, admin console, and the **OpenAI‑compatible** API (`/api/...`) for programmatic chat completions. |

**Internal surface** (not published to the host)

| Target | Address (inside `ai_network`)      | Consumed by                          |
| ------ | ---------------------------------- | ----------------------------------- |
| Ollama | `http://ollama:11434`              | `open-webui` via `OLLAMA_BASE_URL`  |

Because Ollama has **no published host port**, anything needing model access should go through Open WebUI's API (which also carries the auth/JWT layer) rather than reaching for `11434` directly — that address only resolves on the internal compose network.

> **OpenAI‑compatible usage:** Open WebUI exposes an OpenAI‑shaped `/api/chat/completions` (plus `/api/...` helpers). Point standard OpenAI SDKs at the base URL and use an API key created under *Settings → My Profile / Manage Users* instead of the admin token for least‑privilege integrations.

---

## Data Persistence & Backup

| Volume            | Mounted at          | Contents                                                     | Relevance |
| ----------------- | ------------------- | ------------------------------------------------------------ | --------- |
| `ollama-data`     | `/root/.ollama`     | Downloaded models, blobs, Ollama runtime state               | Lose it = re‑pull all models (network + time cost) |
| `open-webui-data` | `/app/backend/data` | Users, API keys, chat history, workspace/prompt data, metadata DB | Lose it = accounts and history are gone |

**Actual on‑host volume names.** Docker Compose v2 derives the project name from the folder containing the compose file, so from here the named volumes are:

- `local-ai-stack_ollama-data`
- `local-ai-stack_open-webui-data`

If you deploy the project from a differently‑named directory or through CI, pin the project name with `COMPOSE_PROJECT_NAME=local-ai-stack` so these names (and thus backups/restores) stay stable.

**Backup (host side, outside compose):**

```bash
# On the Docker host, with $PWD as the backup destination dir:
mkdir -p ./ai-backup/$(date +%F)
for v in local-ai-stack_ollama-data local-ai-stack_open-webui-data; do
  docker run --rm -v "$v":/src:ro -v "$PWD/ai-backup/$(date +%F)":/dst alpine \
    tar czf "/dst/${v}.tgz" -C /src .
done
```

Restore is the inverse: unpack the tarball into the directory the volume provides (i.e. recreate/write to `_data/`) and start the container — Open WebUI re‑attaches to its existing DB on start, and Ollama resumes from the model blobs.

---

## Security Considerations

**Implemented by this project**

- `security_opt: no-new-privileges=true` on **both** containers.
- Ollama's model API is **not host‑publishable by design** — only reachable from the internal compose network.
- Open WebUI's sensitive operations are tied to `WEBUI_SECRET_KEY`; the template directs you to generate a strong per‑installation value rather than shipping one.
- TLS termination (Option B) in Traefik with a cert resolver — no plaintext exposure in production mode.
- Deployments are fully declarative and reproducible; there is no ad‑hoc `docker run` or host‑file mutation during release.

**Operational recommendations for this deployment**

- **Pin image tags.** `latest` / `main` are acceptable for a dev setup; production should pin digests or at least release tags (compose pull + `up` makes that change atomic).
- **Treat `.env` as a code‑adjacent secret.** It is git‑ignored (see `.gitignore`) so real values never enter VCS; keep it out of the working tree in CI by injecting it at build time from your own secret store. This repo deliberately contains no secret material.
- **Least‑privilege for Ollama.** It runs as root inside its container (`/root/.ollama`). If you need a hardened runtime profile, consider a seccomp/AppArmor policy or a wrapping sidecar that drops privileges — but benchmark first against the shared‑memory and NUMA needs of GPU inference.
- **Admin account hardening:** give the Open WebUI admin account a strong password, and (if you use `/api` from other services) prefer scoped API keys over the admin token where possible.
- **Egress posture:** only the image registry, the model source, and (Option B) the certificate provider need internet — any other egress is suspicious and worth a policy check.

---

## Operations & Troubleshooting

**Routine**

```bash
docker compose ps                                # service liveness
docker compose logs -f open-webui                # follow app logs
docker compose logs -f ollama                    # follow inference logs
docker compose exec ollama ollama ps             # loaded models / VRAM
docker compose pull && docker compose up -d      # manual update loop
```

**Common symptoms**

| Symptom                                                   | Likely cause / fix                                                                                                    |
| --------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `network "proxy" not found` (or your own)                 | Option B references a non‑existent external network. Create it on the Traefik side, or switch to Option A in `docker-compose.yaml`. |
| Open WebUI won't find Ollama after a host migration         | `OLLAMA_BASE_URL` still points at a service name that resolves only inside this network — keep both containers in the same project. |
| UI works locally, not via domain (Option B)               | DNS A record missing/pointing at the wrong IP; Traefik entrypoint/labels mismatch (check `traefik_api` router debug output). |
| `WEBUI_SECRET_KEY` change resets sessions                 | Expected and intentional — the key signs JWTs. Change only when rotating, and expect a one‑time re‑login.              |
| Ollama pull is slow, then stops                            | Model download interrupted mid‑write. Re‑run the pull (Ollama resumes), or clear partial state under `/root/.ollama` inside the container. |
| Port conflict (Option A)                                  | `WEBUI_LOCAL_PORT` already bound. Change it in `.env`, recreate.                                                       |
| CPU fan spinning at idle after a GPU model was loaded     | Model still resident in VRAM: `docker compose exec ollama ollama ps` → unload with `ollama stop <model>` or restart ollama.                     |

---

## References

- [Ollama](https://github.com/ollama/ollama) — inference engine; model gallery at `https://ollama.com/library`
- [Open WebUI](https://github.com/open-webui/open-webui) — chat UI + admin + OpenAI‑compatible API
- [Traefik documentation](https://doc.traefik.io/traefik/) — labels, entrypoints, and cert resolvers consumed by Option B
- [Docker Compose reference](https://docs.docker.com/compose/reference/)

---

*Repository: `local-ai-stack` · Stack: Ollama + Open WebUI · Deploy: Docker Compose (Option A local / Option B Traefik) · All inference is local.*

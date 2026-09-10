# LibreChat

A Docker Compose stack for self-hosting [LibreChat](https://www.librechat.ai/) —
an AI assistant UI with multi-provider chat, RAG, and MCP tool integrations —
plus its supporting services (MongoDB, Meilisearch, pgvector, RAG API, admin panel).

## Architecture

Six services on the default compose network:

```
Browser
  │ :3080                    │ :3000
  ▼                          ▼
┌──────────┐  ┌─────────────┐     ┌───────────┐   ┌───────────┐
│   api    │──│  admin-panel│     │  mongodb  │   │ meilisearch│
│ (web UI  │  └─────────────┘     │   :27017  │   │   :7700   │
│  :3080)  │                      └───────────┘   └───────────┘
└──────────┘
  │  ┌───────────┐
  └─▶│  rag_api  │──▶ vectordb (pgvector :5432)
     │  :8000    │
     └───────────┘
```

| Service | Image | Role |
|---------|-------|------|
| `api` | `registry.librechat.ai/danny-avila/librechat-dev:latest` | Main LibreChat web app |
| `admin-panel` | `registry.librechat.ai/clickhouse/librechat-admin-panel:latest` | Admin dashboard |
| `mongodb` | `mongo:8.0.20` | Primary database (`--noauth`, local dev only) |
| `meilisearch` | `getmeili/meilisearch:v1.35.1` | Search index |
| `vectordb` | `pgvector/pgvector:0.8.0-pg15-trixie` | Vector store for RAG |
| `rag_api` | `registry.librechat.ai/danny-avila/librechat-rag-api-dev-lite:latest` | RAG backend |

Only **two ports are published**: `3080` (app) and `3000` (admin panel). Everything
else stays on the internal network.

## How to run

```bash
# 1. One-time: create the env file from the committed template
cp dot_env .env
#    .env is git-ignored — put real API keys/secrets there, never in dot_env

# 2. Start
docker compose up -d

# 3. Verify
docker compose ps
curl -sI http://localhost:3080
```

- App: <http://localhost:3080>
- Admin panel: <http://localhost:3000>

### External MCP dependencies

`librechat.yaml` registers two MCP servers that live **outside** this stack
(see sibling directories in this repo):

| MCP server | Address | Where it runs |
|------------|---------|---------------|
| `searxng` (web search) | `http://host.docker.internal:3001/mcp` | `../searnxg-mcp` |
| `crawl4ai` (page crawler) | `http://host.docker.internal:8055/sse` | `../crawl4ai` |

Start those stacks first if the tools should be available. A custom
**llama.cpp** endpoint (`:8181`) is also configured — optional, only needed
if a local LLM server is running.

## Configuration

### `.env` (per deployment — git-ignored)

Copied from `dot_env`. The compose file overrides the network-relevant values
(`MONGO_URI`, `MEILI_HOST`, `RAG_API_URL`) with service DNS names, so those
pointing at `127.0.0.1` in the template are harmless. What you typically set:

| Variable | Purpose |
|----------|---------|
| `PORT`, `ADMIN_PANEL_PORT` | Host ports for app / admin panel |
| `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GOOGLE_KEY`, … | Provider credentials |
| `JWT_SECRET`, `JWT_REFRESH_SECRET` | Session signing (set before going live) |
| `MEILI_MASTER_KEY` | Meilisearch auth (falls back to a default key if empty) |
| `ADMIN_PANEL_SESSION_SECRET` | Admin panel sessions (weak default if unset) |
| `UID`, `GID` | Container user, for bind-mount file ownership |

### `librechat.yaml` (committed)

- `mcpSettings.allowedAddresses` — MCP address allowlist (SearXNG + Crawl4AI).
- `mcpServers` — registered MCP endpoints and timeouts.
- `endpoints.custom` — the local llama.cpp model (`unsloth/Qwen3.8-27B-GGUF`).

### Compose environment variables

| Variable | Default | Purpose |
|----------|---------|---------|
| `PORT` | `3080` | App port |
| `ADMIN_PANEL_PORT` | `3000` | Admin panel port |
| `RAG_PORT` | `8000` | RAG API port (internal) |
| `UID` / `GID` | `1000` / `984` | User for `api` and `meilisearch` containers |
| `PROXY`, `HTTP(S)_PROXY`, `NO_PROXY` | — | Egress proxying (baked into `api` env) |

## Data & storage

| Path / volume | What's in it |
|---------------|--------------|
| `./data-node/` | MongoDB data (bind mount) |
| `./meili_data_v1.35.1/` | Meilisearch index (bind mount) |
| `./logs/` | LibreChat logs |
| `./uploads/`, `./images/` | User uploads / client images |
| `./skill/` | Mounted at `/app/skill` |
| `librechat-data` (volume) | `/app/data` — accounts, chats, temp credentials |
| `pgdata2` (volume) | pgvector data |

All of the bind-mount directories are git-ignored. To wipe the whole install:
`docker compose down -v` plus deleting the data directories.

## Summary

| Item | Value |
|------|-------|
| App URL | `http://localhost:3080` |
| Admin panel | `http://localhost:3000` |
| Published ports | `3080`, `3000` only |
| Config files | `.env` (secrets, git-ignored) + `librechat.yaml` (MCP/endpoints, committed) |
| MCP tools | SearXNG (web search), Crawl4AI (crawler) — external stacks |
| Persistence | `data-node/`, `meili_data_v1.35.1/` (binds) + `librechat-data`, `pgdata2` (volumes) |

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| `api` container crash-loops | `docker compose logs api` — usually a bad `.env` value (e.g. malformed `JWT_SECRET`); compare against `dot_env` |
| MCP tools missing in chat | External stack down, or address not in `mcpSettings.allowedAddresses` / `MCP_ALLOWED_DOMAINS` |
| Meilisearch 403/invalid key | `MEILI_MASTER_KEY` in `.env` must match the key the `meilisearch` service started with (or leave it empty to use the same default) |
| Uploads/images not readable, permission errors | `UID`/`GID` in `.env` don't match the host user owning the bind dirs: `chown -R 1000:984 data-node meili_data_v1.35.1 uploads images logs skill` |
| Admin panel login loops | Set `ADMIN_PANEL_SESSION_SECRET` to a real secret (≥32 chars) and `docker compose up -d admin-panel` |
| RAG uploads fail | Check `rag_api` and `vectordb` are up: `docker compose logs rag_api vectordb` |

## Security notes (local-dev defaults)

These are intentional for a single-user local setup — **change before exposing
the stack to a network**:

- `mongodb` runs with `--noauth`
- `vectordb` uses `myuser` / `mypassword`
- `MEILI_MASTER_KEY` and `ADMIN_PANEL_SESSION_SECRET` fall back to weak defaults
- `MCP_ALLOW_LOCAL_SERVERS`, `MCP_ALLOW_PRIVATE_NETWORK` and `ALLOW_INTERNAL_URLS`
  are `true` — required to reach MCP servers on `host.docker.internal`, but it
  widens SSRF surface

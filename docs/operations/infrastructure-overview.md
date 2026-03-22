# Infrastructure Overview
Glacial Gems OS
Last Updated: 2026-03-22
Environment: Production (Hybrid VPS + External Hosting)

---

# 1. VPS Baseline

Provider: Contabo
Hostname: vmi2978954
Public IP: 185.230.138.60

Operating System:
- Ubuntu 24.04.3 LTS (noble)
- Kernel 6.8.0-90-generic

Container Runtime:
- Docker 29.1.3
- Docker Compose v5.0.0

---

# 2. Networking Architecture

Primary Docker network: `web`

- Subnet: 172.19.0.0/16
- Gateway: 172.19.0.1
- Purpose: shared reverse-proxy network for all service containers

Containers attached to `web`:

| Container        | IP           | Purpose |
|------------------|--------------|---------|
| caddy            | 172.19.0.3   | Global reverse proxy |
| n8n-n8n-1        | 172.19.0.2   | Automation engine |
| nocodb           | 172.19.0.4   | Database UI |
| mcp              | 172.19.0.5   | Tool orchestration |
| women-db-1       | 172.19.0.13  | Store DB (MariaDB) |
| women-wp-1       | 172.19.0.14  | WordPress store |
| hindsight-db     | hindsight-net | Hindsight Postgres backend |
| hindsight-api    | hindsight-net | Hindsight memory API + UI |

---

# 3. Reverse Proxy Layer (Caddy)

Container: caddy:2
Ports exposed:
- 80 (HTTP)
- 443 (HTTPS TCP + UDP)
- 2019 (admin API internal)

Configuration path:
- /opt/caddy/Caddyfile

Active routes:

- n8n.kolvetni.is → n8n-n8n-1:5678
- baserow.kolvetni.is → nocodb:8080 (BasicAuth protected)
- mcp.kolvetni.is → mcp:3000
- women.is → php_fastcgi women-wp-1:9000
- www.women.is → redirect to https://women.is
- ai.kolvetni.is → OpenClaw gateway (18789) + BasicAuth

TLS certificates are automatically managed by Caddy (ACME).

Nginx also runs on host for auxiliary routing:
- :5173 → Hindsight UI (127.0.0.1:9999)
- :8080 → Chameleon UI

---

# 4. Automation Layer

## n8n

Container: n8n-n8n-1
Image: n8nio/n8n:latest
Port: 5678 (published)
URL: https://vmi2978954.contaboserver.net

Volume:
- n8n_n8n_data

Role:
- Deterministic automation workflows
- Webhooks and scheduling
- Store integrations
- Slack / Telegram / Discord notifications
- Data synchronization

---

# 5. AI / Agent Layer

## OpenClaw Gateway

Version: 2026.3.13
Runs as host process (systemd user service).

Listening port: 0.0.0.0:18789

Configuration directory:
- /root/.openclaw/

Connected channels:
- Telegram (bot token configured, pairing active)
- Discord (bot token configured, intents enabled)
- Slack (bot + app tokens configured)

Public endpoint:
- ai.kolvetni.ai (Caddy + BasicAuth)

Model stack:
- Primary: anthropic/claude-haiku-4-5 (main agent, cost control)
- Leadership agents: anthropic/claude-sonnet-4-6
- Coder agent: ollama/qwen2.5-coder:1.5b (local-first)
- Fallbacks: google/gemini-2.5-pro → ollama/llama3.2

---

## Hindsight Memory System

Version: latest (ghcr.io/vectorize-io/hindsight)
Location: /opt/hindsight/docker-compose.yml
Network: hindsight-net (isolated)

Components:
- hindsight-api: REST API + UI (port 6969 internal, 9999 UI)
- hindsight-db: PostgreSQL 16 backend (port 5434 internal)

LLM backend: Gemini 2.5 Flash (fact extraction)

UI access: http://185.230.138.60:5173 (nginx proxy)
API access: http://127.0.0.1:6969 (localhost only)

Memory bank: `glacial`
- Shared across all OpenClaw agents
- Daily consolidation enabled (target: 50% compression)
- Agents: main, coder, ops, finance, policy, strategy, archivist

Config reference: /root/.openclaw/workspace/hindsight_config.json

Role:
- Persistent cross-agent memory
- Fact extraction and retention
- Replaces OpenClaw built-in memory store

---

## MCP

Container: mcp
Internal port: 3000
Domain: mcp.kolvetni.is

Role:
- Tool orchestration layer (used only when required)

---

# 6. Data Layer

## PostgreSQL (Main)

Container: glacial-postgres
Image: postgres:16
Port binding: 127.0.0.1:5432
Volume: glacial-stack_pgdata
Role: Structured data backend, NocoDB backend

## PostgreSQL (Hindsight)

Container: hindsight-db
Image: postgres:16-alpine
Port binding: 127.0.0.1:5434
Volume: hindsight-db-data
Role: Hindsight memory storage backend

## NocoDB

Container: nocodb
Image: nocodb/nocodb:latest
Internal port: 8080
Domain: baserow.kolvetni.is (legacy subdomain retained)
Authentication: Caddy BasicAuth
Role: Human-friendly database UI, CRUD interface for Postgres

---

# 7. E-Commerce Layer

## Active Store: women.is

WordPress:
- Container: women-wp-1
- Image: wordpress:php8.3-fpm
- Port: 9000 (internal)

Database:
- Container: women-db-1
- Image: mariadb:10.11
- Port: 3306 (internal)

Domain: women.is / www.women.is → redirect
WooCommerce enabled.

---

# 8. Hybrid Hosting (External to VPS)

## kolvetni.is
- Frontend hosted on Vercel
- Subdomains on VPS: ai, n8n, baserow, mcp

## travelsouth.is
- Hosted on Duda CMS (external SaaS)

## airfryer.is / gjafaleit.is / woman.is
- Owned, not deployed

---

# 9. LLM / Developer Tooling

Anthropic:
- Claude Max subscription (powers OpenClaw via OAuth)
- Claude Code (CLI)

Google:
- Gemini API (Hindsight fact extraction, agent fallback)
- Gemini Pro subscription

Microsoft:
- GitHub Copilot Premium

Local:
- Ollama (llama3.2, qwen2.5-coder:1.5b)

The system is vendor-flexible and not locked to a single LLM provider.

---

# 10. Firewall (UFW)

| Port  | Action | Notes |
|-------|--------|-------|
| 22    | ALLOW  | SSH |
| 80    | ALLOW  | HTTP |
| 443   | ALLOW  | HTTPS |
| 5173  | ALLOW  | Hindsight UI (nginx proxy) |
| 8080  | ALLOW  | Chameleon UI |
| 5678  | DENY   | n8n (Caddy only) |
| 18789 | DENY   | OpenClaw gateway (Caddy only) |
| 9999  | ALLOW  | Hindsight UI (direct Docker) |
| 6969  | ALLOW  | Hindsight API (direct Docker) |

---

# 11. Deprecated / Disabled

- Baserow fully removed (replaced by NocoDB)
- Nanobot removed (replaced by OpenClaw)
- Old deployment archived at: /opt/baserow_DISABLED_2026-02-17/

---

# 12. Operational Principles

- One global reverse proxy (Caddy)
- Shared proxy network
- Store-level database isolation
- Deterministic flows in n8n
- AI reasoning only when required (OpenClaw)
- Persistent shared memory via Hindsight
- Postgres as structured backbone
- NocoDB as UI layer
- Slack + Telegram + Discord as human interface layer
- Local-first model stack (Ollama) with cloud fallbacks

---

End of Document

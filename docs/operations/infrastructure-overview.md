# Infrastructure Overview
Glacial Gems OS
Last Updated: 2026-02-18
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

Uptime at last verification: 61+ days

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
| nanobot          | 172.19.0.6   | Telegram AI service |
| women-db-1       | 172.19.0.13  | Store DB (MariaDB) |
| women-wp-1       | 172.19.0.14  | WordPress store |

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
- ai.kolvetni.is → host OpenClaw gateway (18789) + BasicAuth

TLS certificates are automatically managed by Caddy (ACME).

---

# 4. Automation Layer

## n8n

Container: n8n-n8n-1
Image: n8nio/n8n:latest
Port: 5678 (published)

Volume:
- n8n_n8n_data

Role:
- Deterministic automation workflows
- Webhooks
- Scheduling
- Store integrations
- Slack / Telegram notifications
- Data synchronization

---

# 5. AI / Agent Layer

## OpenClaw Gateway

Runs as host process (not container).

Listening ports:
- 0.0.0.0:18789 (primary UI/API)
- 127.0.0.1:18792 (local auxiliary)

Configuration directory:
- /root/.openclaw/

Connected integrations:
- Slack
- Telegram

Public endpoint:
- ai.kolvetni.is (Caddy + BasicAuth)

---

## Nanobot

Container: nanobot
Published port: 18790

Role:
- Telegram AI assistant runtime
- Connected to OpenClaw and tool stack

---

## MCP

Container: mcp
Internal port: 3000
Domain: mcp.kolvetni.is
Health: healthy

Role:
- Tool orchestration layer (used only when required)

---

# 6. Data Layer

## PostgreSQL

Container: glacial-postgres
Image: postgres:16
Port binding: 127.0.0.1:5432

Volume:
- glacial-stack_pgdata

Role:
- Structured data backend
- NocoDB database backend

---

## NocoDB (Replaces Baserow)

Container: nocodb
Image: nocodb/nocodb:latest
Internal port: 8080
Volume: nocodb_data

Domain:
- baserow.kolvetni.is (legacy subdomain retained)

Authentication:
- Caddy BasicAuth

Role:
- Human-friendly database UI
- CRUD interface for Postgres
- Internal operational data

---

# 7. E-Commerce Layer

## WordPress Multi-Store Model

Each store:
- Dedicated WordPress container
- Dedicated MariaDB container
- Routed via global Caddy
- Isolated storage volumes

---

## Active Store: women.is

WordPress:
- Container: women-wp-1
- Image: wordpress:php8.3-fpm
- Port: 9000 (internal)
- Volume: women_women_wp

Database:
- Container: women-db-1
- Image: mariadb:10.11
- Port: 3306 (internal)
- Volume: women_women_db

Domain:
- women.is
- www.women.is → redirect

WooCommerce enabled.

---

# 8. Hybrid Hosting (External to VPS)

## kolvetni.is
- Frontend hosted on Vercel
- Subdomains on VPS:
  - ai.kolvetni.is
  - n8n.kolvetni.is
  - baserow.kolvetni.is
  - mcp.kolvetni.is

## travelsouth.is
- Hosted on Duda CMS (external SaaS)

## airfryer.is
- Owned, not deployed

## gjafaleit.is
- Owned, not deployed

## woman.is
- Owned, not deployed

---

# 9. LLM / Developer Tooling

Google:
- Gemini API account
- Gemini Pro subscription

Anthropic:
- Claude subscription (basic)
- Claude Code

Microsoft:
- GitHub Copilot Premium

The system is vendor-flexible and not locked to a single LLM provider.

---

# 10. Deprecated / Disabled

- Baserow fully removed
- Old deployment archived at:
  /opt/baserow_DISABLED_2026-02-17/

NocoDB is the replacement system.

---

# 11. Operational Principles

- One global reverse proxy
- Shared proxy network
- Store-level database isolation
- Deterministic flows in n8n
- AI reasoning only when required (OpenClaw)
- Postgres as structured backbone
- NocoDB as UI layer
- Slack + Telegram as human interface layer

---

End of Document

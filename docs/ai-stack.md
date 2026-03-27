# AI Stack — Glacial Gems ehf

Last updated: 2026-03-27

---

## Infrastructure

- Everything runs on the **VPS** (vmi2978954, Contabo)
- Work computer does nothing — zero local execution
- RAM: 8GB + 4GB swap (added 2026-03-27)
- OpenClaw config: `~/.openclaw/`
- Main config file: `~/.openclaw/openclaw.json`
- Per-agent model config: `~/.openclaw/agents/<agent>/agent/models.json`
- Gateway runs as: `systemctl --user openclaw-gateway.service` (Node.js)
- Gateway logs: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`
- Ollama on VPS at port `11434`

---

## VPS Git Workflow

All repos cloned at `/opt/repos/`. SSH key configured, no password needed.

| Path | GitHub |
|---|---|
| `/opt/repos/openclaw-workspace` | `aron5109/openclaw-workspace` |
| `/opt/repos/glacial-gems-os` | `aron5109/glacial-gems-os` |

Standard workflow:
```
cd /opt/repos/<repo>
git add <file>
git commit -m "message"
git push origin main
```

Git identity: Aron / aron5109@github.com

---

## Channels

- Discord: `GLACIAL GAMER` bot (guild: Glacial Gems ideas)
- Slack: `gg-control-coordination`, `gg-operations-infrastructure`, `gg-engineering-development`, `gg-finance-vault`, `gg-records-archivist`, `gg-marketing-content`
- Telegram: `@lobsterinnbot`

---

## Agent Model Policy

All agent primaries are set in `~/.openclaw/openclaw.json`.
Per-agent `models.json` files only define available providers, not the primary.
Agents without a `models.json` inherit everything from `openclaw.json`.

| Agent | Primary Model | Type |
|---|---|---|
| main | `openrouter/stepfun/step-3.5-flash:free` | Free (OpenRouter) |
| ops | `anthropic/claude-sonnet-4-6` | Paid |
| finance | `anthropic/claude-sonnet-4-6` | Paid |
| archivist | `openrouter/stepfun/step-3.5-flash:free` | Free (OpenRouter) |
| policy | `anthropic/claude-sonnet-4-6` | Paid |
| strategy | `anthropic/claude-sonnet-4-6` | Paid |
| naggon | `openrouter/stepfun/step-3.5-flash:free` | Free (OpenRouter) |
| coder | `ollama/qwen2.5-coder:1.5b` | Free (local) |
| heartbeat | `ollama/llama3.2:latest` | Free (local) |

### Main/archivist/naggon fallback chain
1. `openrouter/stepfun/step-3.5-flash:free` (primary)
2. `openrouter/nvidia/nemotron-3-super-120b-a12b:free`
3. `ollama/llama3.2:latest` (local)

### Leadership agents fallback chain
1. `anthropic/claude-sonnet-4-6` (primary)
2. `google/gemini-2.5-pro`
3. `ollama/llama3.2:latest` (local)

---

## Token Strategy — Priority Order

### OpenClaw agents (API)
1. `openrouter/stepfun/step-3.5-flash:free` — main agent, always first
2. `openrouter/nvidia/nemotron-3-super-120b-a12b:free` — auto fallback
3. `ollama/llama3.2:latest` — local, free, unlimited
4. `anthropic/claude-sonnet-4-6` — leadership agents only, costs tokens
5. `ollama/qwen2.5-coder:1.5b` — coder agent, local, free
6. OpenRouter $8 credit — **never touch, emergency only**

---

## Free Models Available (OpenRouter)

Rate limit: 200 req/day per model, 20 req/min.

| Model ID | Context | Notes |
|---|---|---|
| `stepfun/step-3.5-flash:free` | 256K | Main agent primary |
| `nvidia/nemotron-3-super-120b-a12b:free` | 262K | Fallback |
| `arcee-ai/trinity-large-preview:free` | 131K | General |

OpenRouter API key: stored in `~/.openclaw/agents/main/agent/auth-profiles.json`

---

## Memory Architecture

Three complementary memory systems:
```
Conversations → OpenViking (port 1933) → auto-captures/recalls
Baldur (archivist) → Hindsight (port 6969) → curated knowledge banks
Baldur (archivist) → Obsidian (/opt/obsidian-vault) → human wiki
```

| System | Purpose | Who writes |
|---|---|---|
| OpenViking | Conversational context engine (automatic) | Plugin auto-captures |
| Hindsight | Curated long-term knowledge bank | Baldur manages |
| Obsidian | Human-readable wiki (74+ notes) | Baldur writes |

---

## OpenViking Context Engine

- Service: `systemctl status openviking` (system-level)
- Port: `1933` (localhost only)
- Config: `~/.openviking/ov.conf`
- Workspace: `/root/.openviking/workspace`
- Logs: `/var/log/openviking.log`
- Plugin: `~/.openclaw/extensions/openviking/` (Plugin 2.0, context-engine slot)
- **Embedding**: `ollama/mxbai-embed-large` (local, 669MB)
- **VLM**: `gemini/gemini-2.0-flash` (Gemini API, free tier)

### How it works
1. Before every agent response → auto-recalls from `viking://user/default/memories/`
2. After every agent response → Gemini extracts facts, mxbai vectorizes, stores memories
3. Memories persist across sessions automatically

### OpenClaw plugin config (in openclaw.json)
```json
{
  "slots": { "contextEngine": "openviking" },
  "allow": ["openviking", "telegram", "slack", "discord"],
  "entries": {
    "openviking": {
      "enabled": true,
      "config": {
        "mode": "remote",
        "baseUrl": "http://127.0.0.1:1933",
        "autoCapture": true,
        "autoRecall": true,
        "recallLimit": 6,
        "recallScoreThreshold": 0.15,
        "recallMaxContentChars": 500,
        "recallTokenBudget": 2000
      }
    }
  }
}
```

---

## Hindsight Memory System

- UI: http://185.230.138.60:5173/banks/glacial?view=data
- External API: `http://127.0.0.1:6969`
- Internal (sandbox): `http://172.22.0.2:8888`
- Docker network: `hindsight_hindsight-net`
- Banks: `glacial`, `naggon`, `skill-collection`, `leadership`, `engineering`, `archivist`

### Sandbox → Hindsight (perl socket)
```perl
perl -e '
use Socket;
socket(my $s, PF_INET, SOCK_STREAM, getprotobyname("tcp"));
connect($s, sockaddr_in(8888, inet_aton("172.22.0.2")));
$s->autoflush(1);
print $s "GET /v1/default/banks HTTP/1.0\r\nHost: hindsight-api\r\nConnection: close\r\n\r\n";
while(<$s>){print}
'
```

---

## Agent Roster

| Agent ID | Name | Role | Model |
|---|---|---|---|
| main | (unnamed) | Coordinator / router | Step 3.5 Flash |
| ops | Kjartan Már | Infrastructure & operations | Sonnet 4.6 |
| finance | Alda Sofía | Accountant / budget tracking | Sonnet 4.6 |
| archivist | Baldur | Records, docs, memory governance | Step 3.5 Flash |
| policy | Selma | Security & compliance | Sonnet 4.6 |
| strategy | Rakel Jónsdóttir | Long-term planning & roadmap | Sonnet 4.6 |
| coder | Coder (Qwen) | Software engineering | qwen2.5-coder:1.5b |
| naggon | Naggon | (private) | Step 3.5 Flash |

---

## Ollama Models

| Model | Size | Purpose |
|---|---|---|
| `mxbai-embed-large:latest` | 669MB | OpenViking embeddings |
| `qwen2.5-coder:1.5b` | 986MB | Coder agent |
| `llama3.2:latest` | 2.0GB | Heartbeat agent |

---

## Docker Services

| Container | Purpose | Port |
|---|---|---|
| hindsight-api | Hindsight memory | 6969→8888 |
| hindsight-db | Postgres for Hindsight | 5434 |
| caddy | Reverse proxy | 80, 443 |
| n8n-n8n-1 | Automation | via Caddy |
| nocodb | Database UI | via Caddy |
| women-wp-1 | WordPress | via Caddy |
| nanobot | Bot service | internal |
| mcp | MCP server | internal |
| glacial-postgres | Main postgres | internal |
| openclaw-sbx-* | Agent sandbox | hindsight_hindsight-net |

---

## Session Management

- Compaction mode: `safeguard` (only valid values: `default`, `safeguard`)
- Session files: `~/.openclaw/agents/<agent>/sessions/`
- Force session reset (rename active .jsonl file):
```bash
mv ~/.openclaw/agents/main/sessions/<id>.jsonl \
   ~/.openclaw/agents/main/sessions/<id>.jsonl.reset.$(date +%Y-%m-%dT%H-%M-%S)
systemctl --user restart openclaw-gateway.service
```

---

## Obsidian Knowledge Vault

- Vault: `/opt/obsidian-vault/` (74+ notes)
- Archivist workspace: `/root/.openclaw/workspace-archivist/obsidian/`
- Structure: `00-inbox`, `01-core`, `02-systems`, `03-operations`, `04-agents`, `05-knowledge`, `99-meta`
- Sync: `rsync -av /root/.openclaw/workspace-archivist/obsidian/ /opt/obsidian-vault/`
- Write helper: `obsidian-write "Title" "Content"`

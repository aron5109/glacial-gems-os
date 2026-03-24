# AI Stack — Glacial Gems ehf

Last updated: 2026-03-24

---

## Infrastructure

- Everything runs on the **VPS** (vmi2978954)
- Work computer does nothing — zero local execution
- OpenClaw config: `~/.openclaw/`
- Main config file: `~/.openclaw/openclaw.json`
- Per-agent model config: `~/.openclaw/agents/<agent>/agent/models.json`
- Gateway runs as: `systemctl --user openclaw-gateway.service` (Node.js)
- Gateway logs: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`
- Heartbeat logs: `/var/log/openclaw-deploy.log`
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
| archivist | `anthropic/claude-sonnet-4-6` | Paid |
| policy | `anthropic/claude-sonnet-4-6` | Paid |
| strategy | `anthropic/claude-sonnet-4-6` | Paid |
| naggon | `anthropic/claude-sonnet-4-6` | Paid |
| coder | `ollama/qwen2.5-coder:1.5b` | Free (local) |
| heartbeat | `ollama/llama3.2:latest` | Free (local) |

### Main agent fallback chain
1. `openrouter/stepfun/step-3.5-flash:free` (primary)
2. `openrouter/nvidia/nemotron-3-super-120b-a12b:free`
3. `ollama/llama3.2:latest` (local)

### Leadership agents fallback chain
1. `anthropic/claude-sonnet-4-6` (primary)
2. `google/gemini-2.5-pro`
3. `ollama/llama3.2:latest` (local)

### Spawning agents
You spawn agents by role — model is tied to the agent, not a slash command.
Example: "spawn a sonnet agent with multiple coder agents helping it."
Sonnet agents = leadership roles. Coder agents = ollama/qwen2.5-coder local.

---

## Anthropic Subscriptions

### Claude Pro — $20/month (personal)
- Access via: claude.ai (web, desktop, mobile)
- Models: Haiku 4.5, Sonnet 4.6, Opus 4.6
- Includes: Claude Code, Cowork, web search, extended thinking
- **Separate from API billing** — subscription ≠ API access

### Claude Code (included in Pro)
- Terminal-based agentic coding tool
- Uses Pro subscription quota — not API tokens
- When quota runs out → do NOT enable pay-as-you-go, switch to OpenRouter free

---

## Token Strategy — Priority Order

### claude.ai / Claude Code (subscription)
1. Use off-peak hours (Iceland = UTC+0 = GMT):
   - Before 12:00 and after 18:00 on weekdays → 2x limits (promotion ends 2026-03-28)
   - 12:00–18:00 weekdays → normal limits
2. Hit limits → stop, switch to OpenRouter free tier

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
Rotate across models to multiply daily budget.

| Model ID | Context | Notes |
|---|---|---|
| `stepfun/step-3.5-flash:free` | 256K | Main agent primary, #1 ranked free |
| `nvidia/nemotron-3-super-120b-a12b:free` | 262K | Coding, agents |
| `arcee-ai/trinity-large-preview:free` | 131K | General |
| `openrouter/free` | auto | Auto-picks best free model |

OpenRouter base URL: `https://openrouter.ai/api/v1` (OpenAI-compatible)
OpenRouter API key: stored in `~/.openclaw/agents/main/agent/models.json`

---

## How to Change a Model

Edit `~/.openclaw/openclaw.json` for agent primaries, then restart:
```bash
systemctl --user restart openclaw-gateway.service
journalctl --user -u openclaw-gateway.service --no-pager | grep "agent model"
```

Always backup first:
```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak-$(date +%Y%m%d)
```

---

## Repos

| Repo | Purpose |
|---|---|
| `aron5109/openclaw-workspace` | Agent workspace, model policy, skills, heartbeat |
| `aron5109/glacial-gems-os` | Company OS, documentation, SOPs (this repo) |

---

## Hindsight Memory System

- UI: http://185.230.138.60:5173/banks/glacial?view=data
- Internal API: `http://hindsight-api:8888` (from within Docker networks)
- External API: `http://127.0.0.1:6969` (from VPS host)
- Docker network: `hindsight_hindsight-net`
- Containers: `hindsight-api` (172.22.0.2), `hindsight-db`
- Banks: `glacial` (main shared), `naggon`, `skill-collection`

### Sandbox → Hindsight access
Agent sandboxes are on `hindsight_hindsight-net` (configured in `openclaw.json`).
Sandbox has no curl/python3 — use perl socket:
```perl
perl -e '
use Socket;
socket(my $s, PF_INET, SOCK_STREAM, getprotobyname("tcp"));
connect($s, sockaddr_in(8888, inet_aton("hindsight-api")));
$s->autoflush(1);
print $s "GET /v1/default/banks HTTP/1.0\r\nHost: hindsight-api\r\nConnection: close\r\n\r\n";
while(<$s>){print}
'
```

### openclaw.json sandbox config
```json
{
  "mode": "non-main",
  "docker": {
    "network": "hindsight_hindsight-net",
    "setupCommand": "apt-get install -y -q curl jq python3 2>/dev/null"
  }
}
```

---

## Agent Roster

| Agent ID | Name | Role | Model |
|---|---|---|---|
| main | (unnamed) | Coordinator / router | Step 3.5 Flash (free) |
| ops | Kjartan Már | Infrastructure & operations | Sonnet 4.6 |
| finance | Alda Sofía | Accountant / budget tracking | Sonnet 4.6 |
| archivist | Baldur | Records, docs, memory governance | Step 3.5 Flash (free) |
| policy | Selma | Security & compliance | Sonnet 4.6 |
| strategy | Rakel Jónsdóttir | Long-term planning & roadmap | Sonnet 4.6 |
| coder | Coder (Qwen) | Software engineering | qwen2.5-coder:1.5b |
| naggon | Naggon | (private) | Step 3.5 Flash (free) |

---

## Hindsight Banks

| Bank | Owner | Purpose |
|---|---|---|
| `glacial` | All agents (read), Baldur (curates) | Glacial Gems company OS memory |
| `leadership` | ops, finance, policy, strategy | Cross-leadership shared context |
| `engineering` | main, coder | Technical decisions, code patterns |
| `archivist` | Baldur only | Archivist private working memory |
| `naggon` | Naggon only | Naggon private memory |
| `skill-collection` | All agents (read), archivist (write) | Agent skills brain |

No access controls enforced — separation is by convention only.

---

## Obsidian Knowledge Vault

An agent-driven knowledge graph system built by Baldur (archivist).

- Vault path: `/opt/obsidian-vault/`
- Archivist workspace: `/root/.openclaw/workspace-archivist/obsidian/`
- Notes: 74 markdown files (growing)

### Vault Structure
```
00-inbox/       — incoming notes
01-core/        — core company knowledge
02-systems/     — infrastructure & system docs
03-operations/  — ops procedures
04-agents/      — agent definitions
05-knowledge/   — schemas, taxonomies
99-meta/        — MOCs, INDEX, entry points
```

### Access Model
| Agent | Access |
|---|---|
| Baldur (archivist) | Read + Write |
| All other agents | Read only (future) |

### Key files
- `99-meta/INDEX.md` — entry point to entire vault
- `99-meta/Glacial OS MOC.md` — company OS map
- `99-meta/OpenClaw System MOC.md` — OpenClaw map
- `99-meta/Agent Architecture MOC.md` — agent map

### Sync workflow
Archivist writes to its workspace, then syncs to vault:
```bash
rsync -av /root/.openclaw/workspace-archivist/obsidian/ /opt/obsidian-vault/
```

### Source data (injected into archivist workspace)
- `/root/.openclaw/workspace-archivist/source_docs` — glacial-gems-os docs
- `/root/.openclaw/workspace-archivist/source_openclaw` — openclaw-workspace

### Write helper CLI
```bash
obsidian-write "Title" "Content"
# Writes to /opt/obsidian-vault/inbox/
```

### Next steps (planned)
- Quartz web UI served via Caddy
- Git sync for versioned vault
- Hindsight → Obsidian memory bridge
- Agent write API replacing CLI script

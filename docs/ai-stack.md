# AI Stack — Glacial Gems ehf

Last updated: 2026-03-24

## Infrastructure

- Everything runs on the **VPS** (vmi2978954)
- Work computer does nothing — zero local execution
- Workspace path: `/opt/repos/openclaw-workspace`
- OpenClaw config: `~/.openclaw/`
- Main config file: `~/.openclaw/openclaw.json`
- Per-agent model config: `~/.openclaw/agents/<agent>/agent/models.json`
- Gateway runs as: `systemctl --user openclaw-gateway.service` (Node.js)
- Heartbeat logs: `/var/log/openclaw-deploy.log`
- Gateway logs: `/tmp/openclaw/openclaw-YYYY-MM-DD.log`
- Ollama on VPS at port `11434`

---

## Channels

- Discord: `GLACIAL GAMER` bot (guild: Glacial Gems ideas)
- Slack: `gg-control-coordination`, `gg-operations-infrastructure`, `gg-engineering-development`, `gg-finance-vault`, `gg-records-archivist`, `gg-marketing-content`
- Telegram: `@lobsterinnbot`

---

## Model Policy

### Main Agent
- Primary: `openrouter/stepfun/step-3.5-flash:free` ✅ (switched 2026-03-24)
- Fallback 1: `openrouter/nvidia/nemotron-3-super-120b-a12b:free`
- Fallback 2: `ollama/llama3.2:latest` (local)
- Config: `~/.openclaw/agents/main/agent/models.json`

### Leadership Agents (ops, finance, archivist, policy, strategy)
- Primary: `anthropic/claude-sonnet-4-6`
- Fallbacks: `google/gemini-2.5-pro` → `ollama/llama3.2:latest`

### Coder Agent
- Primary: `ollama/qwen2.5-coder:1.5b` (local-first, free, unlimited)
- Fallbacks: `anthropic/claude-sonnet-4-6` → `google/gemini-2.5-pro` → `ollama/llama3.2:latest`

### Heartbeat Agent
- Model: `ollama/llama3.2:latest` (local, free, ~1hr interval)
- Returns `HEARTBEAT_OK` unless anomaly detected

---

## Free Model Strategy (OpenRouter)

Goal: **never touch the $8 OpenRouter credit balance** — emergency backup only.

### Free tier rotation (200 req/day per model)
| Model ID | Context | Role |
|---|---|---|
| `stepfun/step-3.5-flash:free` | 256K | Main agent primary |
| `nvidia/nemotron-3-super-120b-a12b:free` | 262K | Main agent fallback 1 |
| `arcee-ai/trinity-large-preview:free` | 131K | Available, not yet assigned |
| `openrouter/free` | auto | Auto-picks best free model |

OpenRouter base URL: `https://openrouter.ai/api/v1` (OpenAI-compatible)
OpenRouter API key: stored in `~/.openclaw/agents/main/agent/models.json`

### Cost philosophy
1. Ollama local → free, unlimited, zero latency
2. OpenRouter free tier → rotate across models, 200 req/day each
3. Claude (claude.ai) → use off-peak for 2x limits (before 12:00 / after 18:00 Iceland time)
4. OpenRouter $8 credit → **never touch unless everything else fails**

---

## How to change a model

Edit the relevant `~/.openclaw/agents/<agent>/agent/models.json` then restart:
```bash
systemctl --user restart openclaw-gateway.service
```

Verify with:
```bash
journalctl --user -u openclaw-gateway.service --no-pager | grep "agent model"
```

---

## Repos

| Repo | Purpose |
|---|---|
| `aron5109/openclaw-workspace` | Agent workspace, model policy, skills, heartbeat |
| `aron5109/glacial-gems-os` | Company OS, documentation, SOPs (this repo) |

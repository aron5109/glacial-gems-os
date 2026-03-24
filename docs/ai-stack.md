# AI Stack — Glacial Gems ehf

Last updated: 2026-03-24

## Infrastructure

- Everything runs on the **VPS** (vmi2978954)
- Work computer does nothing — zero local execution
- Workspace path: `/opt/repos/openclaw-workspace`
- Gateway runs as: `systemctl --user openclaw-gateway.service`
- Heartbeat logs: `/var/log/openclaw-deploy.log`
- Ollama on VPS at port `11434`

---

## Model Policy (per AGENTS.md)

### Main Agent
- Primary: `claude-haiku-4-5` ← **target for replacement with Step 3.5 Flash**
- Fallbacks: `gemini-1.5-pro` → `ollama/llama3.2`

### Leadership Agents (ops, finance, archivist, policy, strategy)
- Primary: `claude-sonnet-4-6`
- Fallbacks: `gemini-2.5-pro` → `ollama/llama3.2`

### Coder Agent
- Primary: `ollama/qwen2.5-coder:1.5b` (local-first, free, unlimited)
- Fallbacks: `claude-sonnet-4-6` → `gemini-2.5-pro` → `ollama/llama3.2`

### Heartbeat Agent
- Model: `ollama/llama3.2` (local, free, ~1hr interval)
- Returns `HEARTBEAT_OK` unless anomaly detected

---

## Free Model Strategy (OpenRouter)

Goal: **never touch the $8 OpenRouter credit balance** — emergency backup only.

### Free tier rotation (200 req/day per model)
| Model ID | Context | Best for |
|---|---|---|
| `stepfun/step-3.5-flash:free` | 256K | Main agent replacement for Haiku |
| `nvidia/nemotron-3-super-120b-a12b:free` | 262K | Coding, agents |
| `arcee-ai/trinity-large-preview:free` | 131K | General fallback |
| `openrouter/free` | auto | Last free resort, auto-picks |

OpenRouter base URL: `https://openrouter.ai/api/v1` (OpenAI-compatible)

### Cost philosophy
1. Ollama local → free, unlimited, zero latency
2. OpenRouter free tier → rotate across models
3. Claude (claude.ai) → use off-peak for 2x limits (promotion or normal)
4. OpenRouter $8 credit → **never touch unless everything else fails**

---

## Pending / TODO

- [ ] Confirm OpenClaw gateway SDK (Anthropic SDK vs OpenAI SDK)
  - Check: `~/.config/openclaw/config.yaml` or `~/.openclaw/config.yaml`
  - Needed to complete Haiku → Step 3.5 Flash swap
- [ ] Pull Qwen 2.5 model list from VPS: `ollama list`
- [ ] Confirm OpenRouter API key is set in VPS environment

---

## Repos

| Repo | Purpose |
|---|---|
| `aron5109/openclaw-workspace` | Agent workspace, model policy, skills, heartbeat |
| `aron5109/glacial-gems-os` | Company OS, documentation, SOPs (this repo) |

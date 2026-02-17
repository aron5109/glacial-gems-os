# OpenClaw → Slack Integration Log

**Date:** 2026-02-17
**Server:** vmi2978954
**User:** root
**OpenClaw Version:** v2026.2.15

---

## 1. Objective

Integrate Slack as an additional communication channel for OpenClaw while keeping Telegram active.

Goal architecture:

```
Slack (Socket Mode)
→ OpenClaw Gateway
→ Agents
→ (Future: Supabase logging / n8n)
```

---

## 2. Environment Discovery

OpenClaw process check:

```bash
ps aux | grep -i openclaw
```

Result:

```
openclaw-gateway
```

Service parent:

```bash
ps -o pid,ppid,cmd -p <PID>
```

Parent:

```
/usr/lib/systemd/systemd --user
```

**Conclusion:** OpenClaw runs as a systemd user service, not a system service.

Service name:

```bash
systemctl --user list-units --type=service | grep -i claw
```

Result:

```
openclaw-gateway.service
```

---

## 3. Slack App Creation

Slack Developer Console: https://api.slack.com/apps

**Created App**
- Name: `clawbrain`
- Workspace: selected workspace

### 3.1 Socket Mode Enabled

Enabled: **Socket Mode**

Generated App-Level Token with scope:

```
connections:write
```

Token: `xapp-...`

### 3.2 Bot Token Scopes Added

Under **OAuth & Permissions → Bot Token Scopes**:

```
app_mentions:read
chat:write
channels:history
channels:read
groups:history
groups:read
im:history
im:read
im:write
mpim:history
mpim:read
mpim:write
users:read
```

Installed App to Workspace.

Generated Bot Token: `xoxb-...`

### 3.3 Event Subscriptions Enabled

Enabled: **Event Subscriptions**

Subscribed to bot events:

```
app_mention
message.im
message.mpim
message.channels
message.groups
```

Saved configuration.

---

## 4. OpenClaw Configuration Changes

Edited:

```
~/.openclaw/openclaw.json
```

Existing Telegram block preserved. Added Slack block under `"channels"`:

```json
"channels": {
  "telegram": {
    "enabled": true,
    "dmPolicy": "pairing",
    "botToken": "TELEGRAM_TOKEN",
    "groupPolicy": "allowlist",
    "streamMode": "partial"
  },
  "slack": {
    "enabled": true,
    "mode": "socket",
    "appToken": "xapp-...",
    "botToken": "xoxb-..."
  }
}
```

---

## 5. JSON Validation

Validated syntax before restart:

```bash
jq . ~/.openclaw/openclaw.json
```

Initial errors:

- Unmatched `}`
- Extra top-level object

Resolved by:

- Removing stray brace at line 70
- Correcting top-level JSON structure

Final validation: ✅ JSON valid

---

## 6. Service Restart

Restarted user service:

```bash
systemctl --user restart openclaw-gateway.service
```

Checked status:

```bash
systemctl --user status openclaw-gateway.service
```

---

## 7. Successful Slack Connection

Journal output confirmed:

```
[slack] starting provider
[slack] socket mode connected
```

Telegram also confirmed:

```
[telegram] starting provider (@lobsterinnbot)
```

**Result:** Slack + Telegram active simultaneously.

---

## 8. Known Issues Observed

### 8.1 Ollama Heartbeat Error

Error:

```
Unknown model: ollama/llama3.2:3b
```

**Cause:** OpenClaw does not recognize provider string for Ollama.

**Action:** Heartbeat configuration to be fixed later. Not blocking Slack functionality.

---

## 9. Current State

**Slack:**
- Socket Mode connected
- Events configured
- Awaiting DM pairing test

**Telegram:**
- Still active
- No disruption

**Gateway:**
- Running as systemd user service
- Listening on `ws://0.0.0.0:18789`

---

## 10. Next Steps (Planned)

- [ ] Confirm Slack DM pairing flow
- [ ] Approve pairing via: `openclaw pairing approve slack <code>`
- [ ] Create per-agent Slack channels
- [ ] Configure per-channel routing rules
- [ ] Implement logging agent → Supabase
- [ ] Fix Ollama heartbeat provider configuration

---

## 11. Infrastructure Summary

**OpenClaw running as:**

```
systemd --user
openclaw-gateway.service
```

**Slack:**
- Socket Mode
- No public webhook needed
- Event subscription active

**Telegram:**
- Still primary fallback channel

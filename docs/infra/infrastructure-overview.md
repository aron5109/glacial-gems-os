# Infrastructure Overview

## Services

### OpenClaw Gateway (Verified)

- Process: `openclaw-gatewa` (system process, not a container)
- Listening:
  - `0.0.0.0:18789` (primary)
  - `127.0.0.1:18792` (local-only auxiliary)
- Public UI Domain: `ai.kolvetni.is` (Caddy + BasicAuth)

**Important:** Since OpenClaw runs on the host (not Docker), Caddy must proxy to the host network reliably.
Recommended upstream: `host.docker.internal:18789` with Docker `extra_hosts` mapping:
`host.docker.internal:host-gateway`.

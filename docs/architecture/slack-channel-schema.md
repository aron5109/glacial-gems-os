# Glacial Gems OS
## Slack Channel Schema (Structured)

**Version:** 1.0
**Status:** Active Design
**Date:** 2026-02-17
**System:** OpenClaw Gateway + Slack (Socket Mode)

---

## 1. Purpose

This document defines the structured Slack channel schema for Glacial Gems OS.

**Objectives:**

- Deterministic routing of OpenClaw agents
- Clear separation of company domains
- Machine-readable channel taxonomy
- Future-proof expansion
- Compatibility with logging pipelines (Supabase / Postgres / n8n)

> This is not a naming preference document.
> This is an operational architecture specification.

---

## 2. Naming Convention

### 2.1 Schema Format

```
gg.<domain>.<capability>
```

Where:

- `gg` = Glacial Gems namespace
- `<domain>` = organizational layer
- `<capability>` = functional specialization

**Properties:**

- Lowercase only
- Dot-separated
- No abbreviations
- No informal short names
- Self-documenting

---

## 3. Domain Layers

The Slack workspace is organized into Tier-1 domains.
Each domain represents a structural function of the company.

### 3.1 Control Layer

| Field | Value |
|---|---|
| **Channel** | `gg.control.coordination` |
| **Agent Role** | Coordinator |

**Function:**
- Task intake
- Prioritization
- Cross-domain routing
- Executive decisions
- Company-level directives

**Behavioral Rules:**
- Short, structured responses
- Maximum 3 recommended next actions
- Route tasks to proper domain channels

---

### 3.2 Operations Layer

| Field | Value |
|---|---|
| **Channel** | `gg.operations.infrastructure` |
| **Agent Role** | Operations Engineer |

**Function:**
- VPS management
- Docker
- Caddy
- Reverse proxy
- Networking
- Deployments
- Monitoring
- Security patches

**Behavioral Rules:**
- Always provide verification commands
- No destructive actions without backup steps
- Prefer minimal change sets

---

### 3.3 Engineering Layer

| Field | Value |
|---|---|
| **Channel** | `gg.engineering.development` |
| **Agent Role** | Developer |

**Function:**
- Application development
- GitHub repos
- Architecture
- Refactoring
- Code review
- API design
- Integrations

**Behavioral Rules:**
- Provide patch-style instructions
- Maintain modular design
- Respect repo structure conventions

---

### 3.4 Finance Layer

| Field | Value |
|---|---|
| **Channel** | `gg.finance.vault` |
| **Agent Role** | Vault Controller |

**Function:**
- Revenue tracking
- Expense tracking
- Cashflow management
- Budget allocation
- Runway forecasting
- Risk evaluation

**Behavioral Rules:**
- Summarize with numbers first
- Highlight risk exposure
- Provide actionable financial recommendation
- Use structured tables when relevant

---

### 3.5 Records Layer

| Field | Value |
|---|---|
| **Channel** | `gg.records.archivist` |
| **Agent Role** | Archivist |

**Function:**
- Daily logs
- Decision records
- Changelog summaries
- Infrastructure change documentation
- Incident reports

**Behavioral Rules:**
- Concise
- Structured markdown
- Timestamped entries
- Link references where possible

---

### 3.6 Marketing Layer

| Field | Value |
|---|---|
| **Channel** | `gg.marketing.content` |
| **Agent Role** | Content Strategist |

**Function:**
- Website copy
- SEO optimization
- Product descriptions
- Social content
- Brand messaging

**Behavioral Rules:**
- Provide ready-to-publish text
- SEO-aware structure
- Clear tone alignment

---

### 3.7 Strategy Layer

| Field | Value |
|---|---|
| **Channel** | `gg.strategy.research` |
| **Agent Role** | Research Analyst |

**Function:**
- Vendor comparison
- Tool evaluation
- Infrastructure alternatives
- Market analysis
- Competitive positioning

**Behavioral Rules:**
- Structured pros/cons
- Risk assessment
- Clear recommendation

---

## 4. Channel Interaction Model

### 4.1 Message Structure

- Top-level message = new task
- Thread = task execution lifecycle
- One task per top-level message

### 4.2 Tagging Protocol

Optional but recommended tags:

- `[DECISION]`
- `[BLOCKER]`
- `[NEXT]`
- `[DONE]`

---

## 5. Routing Model (OpenClaw Integration)

**Routing priority:**

1. Channel name
2. Thread context
3. Mention requirement (optional)
4. Keyword fallback

Channel → Agent mapping is deterministic.

**Example mapping:**

```
gg.finance.vault              → Vault Controller
gg.operations.infrastructure  → Operations Engineer
gg.engineering.development    → Developer
```

---

## 6. Scalability Model

New domains must follow:

```
gg.<newdomain>.<capability>
```

**Examples:**

```
gg.commerce.store
gg.ai.automation
gg.community.events
gg.legal.compliance
```

The schema supports:

- Multi-brand expansion
- Multi-team scaling
- Multi-agent specialization
- Logging partition by domain

---

## 7. Logging & Observability (Planned)

**Future logging architecture:**

```
Slack Event → OpenClaw → n8n → Postgres (glacial-postgres)
```

**Table design:**

```sql
slack_events
- id
- timestamp
- channel
- user
- message
- agent
- thread_id
```

Channel schema simplifies indexing and filtering.

---

## 8. Design Principles

| Principle | Over |
|---|---|
| Explicit | Clever |
| Structured | Informal |
| Deterministic | Ambiguous |
| Scalable | Trendy |
| Machine-readable | Aesthetic |

---

## 9. Current Active Channel Set

```
gg.control.coordination
gg.operations.infrastructure
gg.engineering.development
gg.finance.vault
gg.records.archivist
gg.marketing.content
gg.strategy.research
```

---

## 10. Next Architecture Decision

Choose routing mode:

- **A)** Single OpenClaw instance switching persona by channel
- **B)** Multiple internal agent profiles mapped per channel

> **Recommended for Glacial Gems OS scale: Option B**

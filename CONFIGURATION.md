# Configuration

Configuration is environment variables and (later) a small YAML/TOML file overlay. Secrets never belong in git. Missing mutating flags mean **deny**.

This document describes the intended surface. Files such as `.env.example` are created in Phase 2.

## Files (planned)

| File | Committed | Purpose |
|---|---|---|
| `.env.example` | Yes | Empty placeholders |
| `.env` | No | Local secrets |
| `config/development.yaml` | Yes | Local fake-friendly defaults |
| `config/test.yaml` | Yes | pytest |
| `config/production.yaml.example` | Yes | Production template, no secrets |
| `data/` | No | SQLite, emergency-stop file, uploads |

`.gitignore` must include `.env`, `data/`, `*.pem`, `*.key`, vault files, `.venv/`.

## Core

| Variable | Default | Notes |
|---|---|---|
| `AGENT_MODE` | `read_only` | `read_only` \| `approval_required` \| `maintenance` |
| `AGENT_EMERGENCY_STOP` | `false` | Also file-based stop |
| `EMERGENCY_STOP_FILE` | `data/EMERGENCY_STOP` | Presence disables mutation |
| `PROXMOX_AGENT_MASTER_KEY` | none | Required to decrypt secrets; generate a long random string |
| `DATABASE_URL` | `sqlite+aiosqlite:///data/proxmox_agent.db` | Postgres later |
| `BIND_HOST` | `127.0.0.1` | Do not default to `0.0.0.0` without firewall notes |
| `BIND_PORT` | `8080` | |
| `SESSION_SECRET` | none | Separate from master key; required to start UI |
| `LOG_LEVEL` | `INFO` | |
| `TZ_DISPLAY` | `UTC` | Storage remains UTC |

## Proxmox

| Variable | Default | Notes |
|---|---|---|
| `PROXMOX_API_URL` | none | e.g. `https://pve1.lab:8006` |
| `PROXMOX_TOKEN_SECRET_ID` | none | Points at secret store, not the token value |
| `PROXMOX_TLS_VERIFY` | `true` | |
| `PROXMOX_CA_FILE` | empty | Lab CA PEM |
| `PROXMOX_TIMEOUT_SECONDS` | `30` | |
| `INVENTORY_SYNC_INTERVAL_SECONDS` | `300` | |
| `INVENTORY_SYNC_MIN_INTERVAL_SECONDS` | `30` | Rate floor |

Phase 2 credential: API token with **PVEAuditor**. Write-capable tokens are a later, explicit change.

## LLM

| Variable | Default | Notes |
|---|---|---|
| `LLM_ENABLED` | `false` until Phase 4 | Inventory works without it |
| `LLM_PROVIDER` | `ollama` | Cloud values ignored unless explicitly allowed later |
| `OLLAMA_BASE_URL` | `http://127.0.0.1:11434/v1` | |
| `OLLAMA_MODEL` | empty | Set when enabling Phase 4 |
| `LLM_CLOUD_ENABLED` | `false` | Must stay false unless operator approves |
| `CHAT_MAX_TOOL_ITERS` | `8` | |

## Auth and limits

| Variable | Default | Notes |
|---|---|---|
| `BOOTSTRAP_ADMIN_USERNAME` | empty | Bootstrap only |
| `BOOTSTRAP_ADMIN_PASSWORD` | empty | Bootstrap only; never log |
| `SESSION_IDLE_SECONDS` | `43200` | |
| `LOGIN_RATE_LIMIT` | `10/minute` | |
| `MUTATION_RATE_LIMIT` | `20/hour` | Per user |
| `APPROVAL_TTL_SECONDS` | `900` | |

## Action flags

All default `false`. Names match action families, for example:

- `ACTION_VM_START`
- `ACTION_VM_SHUTDOWN`
- `ACTION_VM_REBOOT`
- `ACTION_VM_PAUSE`
- `ACTION_VM_RESUME`
- `ACTION_VM_CLONE`
- `ACTION_VM_CREATE`
- `ACTION_VM_DELETE`
- `ACTION_VM_MIGRATE`
- `ACTION_SNAPSHOT_CREATE`
- `ACTION_SNAPSHOT_RESTORE`
- `ACTION_GUEST_USER_CREATE` (Phase 7)
- `ACTION_SIMULATION`

Unlisted families are denied.

## Monitoring thresholds (defaults)

Overridable in config files:

| Signal | Warning | Critical |
|---|---|---|
| Node/guest CPU ratio | 0.85 | 0.95 |
| Memory ratio | 0.85 | 0.95 |
| Guest disk ratio | 0.80 | 0.90 |
| Storage used ratio | 0.80 | 0.90 |
| Snapshot age | 30 days | 90 days (informational) |
| Backup failed | n/a | immediately critical |
| Node offline | n/a | critical |
| Guest unexpected stopped (if tagged `wanted:running`) | warning | after grace period critical |

Do not alert on missing qemu-guest-agent as critical; show as unknown/incomplete.

## Notifications

`NOTIFICATION_ADAPTER=null` or `log`. No vendor assumed. Additional adapters implement `NotificationPort` later. A `log` adapter must at least emit the change-report id after executions; storage of the report is independent of notifications.

## Development vs test vs production

**Development:** SQLite in `data/`, optional fake Proxmox URL, `AGENT_MODE=read_only`, bind localhost, LLM optional.

**Test:** in-memory or temp SQLite, fake Proxmox, LLM stub, all action flags false unless a test enables one.

**Production template:** TLS verify true, bind per firewall, `read_only` until flags are reviewed, master key and session secret from a credential manager, persistent volume for `data/`, documented CA file.

## Example environment (illustrative — not real credentials)

```bash
AGENT_MODE=read_only
PROXMOX_API_URL=https://pve.lab.example:8006
PROXMOX_TLS_VERIFY=true
PROXMOX_CA_FILE=/etc/proxmox-agent/lab-ca.pem
PROXMOX_TOKEN_SECRET_ID=proxmox-auditor
LLM_ENABLED=false
LLM_CLOUD_ENABLED=false
DATABASE_URL=sqlite+aiosqlite:///data/proxmox_agent.db
BIND_HOST=127.0.0.1
BIND_PORT=8080
NOTIFICATION_ADAPTER=log
```

Never commit filled `PROXMOX_TOKEN`, passwords, or `PROXMOX_AGENT_MASTER_KEY` values.

## Feature enablement checklist (before flipping a Phase 6 flag)

1. Auditor-only token replaced or supplemented with the **minimum** ACL for that action
2. Backup of agent DB
3. `approval_required` mode, not silent auto-execute (there is no auto-execute mode)
4. Tests for that family green
5. Kill-switch drill once
6. Document the token id in secret metadata (not the secret)

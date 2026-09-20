# API Design

Human overview of HTTP contracts. FastAPI will generate OpenAPI at `/openapi.json` once the app exists. Implementation must not add privileged endpoints that bypass [SECURITY.md](SECURITY.md) or [APPROVAL_WORKFLOW.md](APPROVAL_WORKFLOW.md).

## Conventions

- Prefix: `/api/v1` for JSON.
- UI routes are session-authenticated HTML (not listed exhaustively here).
- JSON: camelCase vs snake_case — **snake_case** in JSON to match Python, documented in OpenAPI.
- Time: ISO-8601 UTC with `Z`.
- Pagination: `limit` (default 50, max 200), `offset` or `cursor` where lists can grow (audit, events).
- Errors: see [Error shape](#error-shape).
- Idempotency: mutating POSTs that create approvals accept optional `Idempotency-Key`.

## Authentication

| Route class | Auth |
|---|---|
| `GET /health` | None |
| `GET /ready` | None (must not leak secrets; may say `db: false`) |
| All `/api/v1/*` except login | Session cookie or (later) hashed API token |
| HTML UI | Session + CSRF on POST |

Login: `POST /api/v1/auth/login` with username/password → Set-Cookie. `POST /api/v1/auth/logout`. `GET /api/v1/auth/me`.

Roles: `viewer`, `operator`, `admin`.

## Error shape

```json
{
  "error": {
    "code": "ambiguous_target",
    "message": "Multiple guests named 'web'. Specify vmid.",
    "details": { "candidates": [ { "vmid": 101, "node": "pve1" }, { "vmid": 202, "node": "pve2" } ] }
  }
}
```

Stable `code` values include: `unauthenticated`, `forbidden`, `not_found`, `conflict`, `validation_error`, `ambiguous_target`, `stale_inventory`, `read_only`, `action_disabled`, `approval_required`, `approval_expired`, `preview_mismatch`, `emergency_stop`, `rate_limited`, `upstream_unavailable`, `partial_failure`, `unverified_result`.

HTTP mapping: 401, 403, 404, 409, 422, 429, 503 as appropriate. `read_only` and `emergency_stop` use 403.

## Health

- `GET /health` — process is up. `{ "status": "ok" }`
- `GET /ready` — `{ "status": "ok"|"degraded"|"not_ready", "checks": { "database": true, "proxmox": true, "llm": false, "emergency_stop": false } }`

`llm: false` is **degraded**, not not_ready. `database: false` is not_ready. `emergency_stop: true` is degraded but ready for reads.

## Inventory

- `GET /api/v1/inventory/cluster`
- `GET /api/v1/inventory/nodes`
- `GET /api/v1/inventory/guests` — query: `status`, `node`, `kind`, `q` (search name/vmid/tags)
- `GET /api/v1/inventory/guests/{id}`
- `GET /api/v1/inventory/storage`
- `GET /api/v1/inventory/networks`
- `GET /api/v1/inventory/guests/{id}/snapshots`
- `GET /api/v1/inventory/guests/{id}/backups`
- `GET /api/v1/inventory/documentation/{guest_id}`
- `PUT /api/v1/inventory/documentation/{guest_id}` — operator overlay (purpose, owner, …); not a Proxmox write
- `POST /api/v1/inventory/sync` — operator+; enqueues a job; returns `job_id`

Guests include allocations, tags, description, last_seen, tombstoned flag, agent facts when present.

## Monitoring

- `GET /api/v1/alerts` — query: `severity`, `status`, `object_type`
- `GET /api/v1/alerts/{id}`
- `GET /api/v1/alerts/{id}/events`
- `POST /api/v1/alerts/{id}/ack`
- `POST /api/v1/alerts/{id}/suppress`
- `GET|POST /api/v1/maintenance-windows`
- `GET /api/v1/metrics/{object_type}/{id}` — recent snapshots

## Approvals and execution

- `GET /api/v1/approvals` — filter status
- `POST /api/v1/approvals` — create from a structured tool proposal (used by UI/chat backend, not raw model JSON from the browser)
- `GET /api/v1/approvals/{id}` — full preview: target, action, reason, impact, risk, API calls, backup, rollback
- `POST /api/v1/approvals/{id}/decide` — `{ "decision": "approved"|"rejected", "comment": "...", "reauth_password": "..." }`  
  `reauth_password` required when risk is `high`. Never echoed back. Never logged.
- Execution starts only after approve; client polls `GET /api/v1/jobs/{id}`

There is no `POST /api/v1/vm/{id}/delete` that skips approval. Chat cannot pass a “pre-approved” flag.

## Audit, jobs, reports, chat

- `GET /api/v1/audit` — filter `since`, `action`, `actor`, `simulated`
- `GET /api/v1/jobs`, `GET /api/v1/jobs/{id}`
- `GET /api/v1/reports/{type}` — types: `inventory`, `cluster_health`, `vm_status`, `resource_usage`, `storage`, `backups`, `incidents`, `changes`, `users`, `security`, `failed_operations`, `troubleshooting`, `agent_activity`, `approvals`
- `GET /api/v1/change-reports`, `GET /api/v1/change-reports/{id}` — narrative “what broke / how it was fixed” for every terminal execution; see [CHANGE_REPORTS.md](CHANGE_REPORTS.md)
- Query `format=json|markdown|csv|html` (PDF later)
- `POST /api/v1/chat` — `{ "conversation_id": null, "message": "...", "incident_id": null }`  
  Response includes `understood`, `target`, `proposed_operation`, `impact`, `risk`, `prerequisites`, `approval_required`, `messages`, optional `approval_id`
- `GET /api/v1/conversations/{id}`

## Control plane

- `POST /api/v1/emergency-stop` — admin; `{ "reason": "..." }`
- `DELETE /api/v1/emergency-stop` — admin clear (audited)
- `GET /api/v1/config` — non-secret effective config (modes, flags, thresholds)
- `PUT /api/v1/config` — admin subset; secrets never accepted here
- Secret **metadata** CRUD under `/api/v1/secrets` (admin): no GET of plaintext. Create takes the value once over HTTPS and returns metadata only.

## Incident reports

`GET /api/v1/incidents`, `POST`, `GET/{id}`, `PATCH/{id}` with fields: incident id, timestamps, affected systems, original error, investigation steps, evidence refs, likely cause, proposed solutions, approved actions, commands/API executed, results, resolution, rollback, lessons.

Closing an incident or finishing an approved execution must leave a persisted change/incident report. Chat returns the report id. Optional `NotificationPort` delivery does not replace storage.

## Versioning and compatibility

Breaking JSON changes require `/api/v2` or additive fields. Error `code` strings are part of the contract.

## Out of scope for v1 HTTP API

- GraphQL
- Websocket streaming (HTMX polling is enough initially)
- Webhook ingress from random URLs
- Proxying Proxmox UI

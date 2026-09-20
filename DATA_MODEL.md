# Data Model

SQLite (SQLAlchemy 2.x) is the default store. Alembic migrations are required from the first schema. Postgres must remain a viable dialect later; avoid SQLite-only SQL in migrations where practical.

Timestamps are stored in UTC as timezone-aware values. Natural keys from Proxmox (node name, VMID) are unique **within** a cluster, not globally guessed from chat.

The agent never persists secret **values** in plaintext. See `SecretMetadata` and [SECURITY.md](SECURITY.md).

## Conventions

- Primary keys: UUID strings (`id`) unless noted.
- Soft-delete is **not** used for inventory: sync upserts live objects and marks missing ones `tombstoned_at` so history remains.
- Inventory snapshots are the current picture; history of config changes is `ConfigChange` plus audit events.
- All mutating domain operations write an `AuditEvent`.

## Entity-relationship overview

```
Cluster 1──* Node 1──* Guest
Node 1──* Storage
Node 1──* NetworkInterface
Cluster 1──* Storage (shared)
Guest 1──* GuestNic
Guest 1──* Snapshot
Guest 1──* Backup
Guest 1──1 DocumentationRecord
Guest *──* Service (via GuestService)
Guest *──* Guest (dependencies)

User 1──* Session
User 1──* ApprovalRequest (requester)
User 1──* ApprovalDecision (decider)

Incident 1──* Conversation 1──* Message
Incident *──* Alert
ApprovalRequest 1──* AuditEvent
Job 1──* Task
SimulationRun 1──* AuditEvent
```

## Inventory

### Cluster

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| pve_cluster_name | str | From API |
| version | str nullable | |
| quorate | bool nullable | |
| last_sync_at | datetime nullable | |
| last_sync_status | str | `ok`, `error`, `stale` |
| last_sync_error | str nullable | No secrets |

### Node

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| cluster_id | FK | |
| name | str | Proxmox node name, unique per cluster |
| status | str | `online`, `offline`, `unknown` |
| cpu_cores | int nullable | |
| cpu_usage_ratio | float nullable | Last sample |
| memory_total_bytes | int nullable | |
| memory_used_bytes | int nullable | |
| uptime_seconds | int nullable | |
| pve_version | str nullable | |
| last_seen_at | datetime nullable | |

Unique: `(cluster_id, name)`.

### Storage

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| cluster_id | FK | |
| node_id | FK nullable | Null if shared/cluster storage |
| storage_id | str | Proxmox storage identifier |
| type | str | `dir`, `zfspool`, `nfs`, … |
| content | str nullable | e.g. images,backup,iso |
| total_bytes | int nullable | |
| used_bytes | int nullable | |
| available_bytes | int nullable | |
| enabled | bool | |
| last_seen_at | datetime nullable | |

### NetworkInterface

Node-level bridges, bonds, VLANs, physical NICs as reported by the node network API.

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| node_id | FK | |
| iface | str | |
| type | str | `bridge`, `vlan`, `bond`, `eth`, … |
| active | bool nullable | |
| address_cidr | str nullable | |
| bridge_ports | str nullable | |
| vlan_id | int nullable | |
| last_seen_at | datetime nullable | |

### Guest

QEMU VMs and LXC containers.

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| cluster_id | FK | |
| node_id | FK | Current host |
| vmid | int | |
| name | str | |
| kind | str | `qemu` or `lxc` |
| status | str | `running`, `stopped`, `paused`, `unknown` |
| ostype | str nullable | From config |
| os_name | str nullable | From guest agent if present |
| os_version | str nullable | |
| hostname | str nullable | |
| description | str nullable | Treated as untrusted text |
| tags | JSON list | |
| cpu_cores | int nullable | |
| memory_bytes | int nullable | |
| onboot | bool nullable | Startup order related |
| startup_order | str nullable | Proxmox `startup` string |
| template | bool | |
| agent_configured | bool nullable | qemu-guest-agent |
| last_seen_at | datetime nullable | |
| tombstoned_at | datetime nullable | Missing from last successful sync |

Unique: `(cluster_id, vmid)`.

Ambiguous name resolution uses `name` **plus** `vmid` and `node.name`. Tools take `guest_id` or `(vmid, node_name)` together, never name alone when collisions exist.

### GuestNic

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| guest_id | FK | |
| slot | str | e.g. `net0` |
| mac | str nullable | |
| bridge | str nullable | |
| vlan | int nullable | |
| ip_addresses | JSON list | Best-effort from agent/config |
| model | str nullable | |

### Snapshot

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| guest_id | FK | |
| name | str | |
| description | str nullable | Untrusted |
| created_at | datetime nullable | |
| includes_ram | bool nullable | |

### Backup

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| guest_id | FK nullable | |
| storage_id | FK nullable | |
| volume_id | str | Proxmox volid |
| format | str nullable | |
| size_bytes | int nullable | |
| created_at | datetime nullable | |
| verified | bool nullable | |
| last_status | str nullable | `ok`, `failed`, `unknown` |

### DocumentationRecord

Maintained documentation overlay (operator-edited plus discovered facts).

| Column | Type | Notes |
|---|---|---|
| id | UUID PK | |
| guest_id | FK unique | |
| purpose | str nullable | |
| owner | str nullable | |
| os_notes | str nullable | |
| backup_notes | str nullable | |
| monitoring_notes | str nullable | |
| last_updated_at | datetime | |
| last_seen_at | datetime nullable | |
| extra | JSON | Extensible, still untrusted if from guests |

Searchable fields: name, vmid, purpose, owner, hostname, IPs, tags, services.

### Service and dependencies

`Service`: id, name, kind, description, owner.

`GuestService`: guest_id, service_id, ports, notes.

`GuestDependency`: from_guest_id, to_guest_id, kind (`network`, `storage`, `app`), notes. Never inferred as authorization.

## Monitoring

### MetricSnapshot

Point-in-time samples. Retention is configurable.

| Column | Notes |
|---|---|
| id | UUID |
| object_type | `node`, `guest`, `storage` |
| object_id | UUID |
| collected_at | datetime |
| cpu_ratio, memory_ratio, disk_ratio | floats nullable |
| extra | JSON |

### Alert, AlertEvent, Acknowledgement, Suppression, MaintenanceWindow

- **Alert**: current or latest state for a rule + object (`ok`, `warning`, `critical`, `unknown`), severity, threshold snapshot, `silenced_until`.
- **AlertEvent**: history of transitions.
- **Acknowledgement**: who, when, comment.
- **Suppression**: rule or object, until, reason.
- **MaintenanceWindow**: time range, objects or global, suppresses alerting (not auditing).

Rules are code + config (not free-form eval). See [CONFIGURATION.md](CONFIGURATION.md).

## Identity and access (agent)

### User

| Column | Notes |
|---|---|
| id | UUID |
| username | unique |
| password_hash | Argon2id |
| role | `admin`, `operator`, `viewer` |
| is_active | bool |
| created_at, last_login_at | |

### Session

id, user_id, created_at, expires_at, last_seen_at, user_agent_hash (optional). Token is random; only a hash is stored if bearer tokens are added later.

## Approvals and jobs

### ApprovalRequest

| Column | Notes |
|---|---|
| id | UUID |
| requester_id | FK User |
| action_family | e.g. `vm.start` |
| risk | `low`, `medium`, `high` |
| status | see [APPROVAL_WORKFLOW.md](APPROVAL_WORKFLOW.md) |
| guest_id / node_id / extra_targets | JSON of inventory UUIDs |
| understood_intent | str |
| expected_impact | str |
| rollback_plan | str |
| backup_plan | str |
| preview_payload | JSON (method, path, body **redacted**) |
| preview_hash | str |
| dry_run | bool |
| simulated | bool |
| expires_at | datetime |
| created_at | datetime |

### ApprovalDecision

id, request_id, decider_id, decision (`approved`, `rejected`), comment, reauth_verified bool, created_at.

### Job and Task

Job: kind (`inventory_sync`, `health_check`, `report`, `execute_approval`), status, created_at, started_at, finished_at, error (redacted).

Task: job_id, name, status, progress 0–100, detail.

## Audit, incidents, chat

### AuditEvent

Append-only. Application must not update or delete rows.

| Column | Notes |
|---|---|
| id | UUID |
| seq | integer monotonic |
| prev_hash | str |
| row_hash | str of canonical fields |
| at | datetime |
| actor_user_id | nullable (system jobs) |
| action | str |
| object_type, object_id | nullable |
| approval_request_id | nullable |
| simulated | bool default false |
| dry_run | bool |
| result | `ok`, `denied`, `error`, `partial` |
| detail | JSON redacted |

`row_hash = H(seq || prev_hash || canonical_payload)`.

### Incident

id, title, status, affected_guest_ids JSON, original_error, likely_cause, resolution, lessons, created_at, closed_at.

### ChangeReport

Narrative record required after every terminal execution and when an incident is closed. See [CHANGE_REPORTS.md](CHANGE_REPORTS.md).

| Column | Notes |
|---|---|
| id | UUID |
| incident_id | FK nullable |
| approval_request_id | FK nullable |
| job_id | FK nullable |
| title | str |
| affected | JSON (guest/node ids, vmid, names) |
| what_was_broken | str (problem, alert, or requested change) |
| investigation | str nullable |
| evidence | JSON redacted |
| likely_cause | str nullable |
| how_fixed | str |
| api_calls | JSON redacted preview/execute list |
| result | `ok`, `failed`, `partial` |
| rollback | str nullable |
| lessons | str nullable |
| simulated | bool |
| created_at, closed_at | datetime |

### Conversation and Message

Conversation belongs to a user and optionally an incident. Messages: role `user` / `assistant` / `system_notice`, content (redacted), created_at. Tool proposals are stored as structured JSON on the assistant message, not as executable state.

## Secrets and simulations

### SecretMetadata

id, name, kind (`proxmox_token`, `ssh_key`, `windows`, `service`, `api_key`), created_at, expires_at, last_used_at, last_rotated_at, revoked_at. **No ciphertext here** if stored in a separate encrypted table/file keyed by id.

### EncryptedSecret

id = SecretMetadata.id, nonce, ciphertext. Decrypt only inside adapters with the master key.

### SimulationRun

id, label, scenario kind, started_at, finished_at, notes. All related approvals and audit rows set `simulated=true`.

### ConfigChange

id, key, old_value, new_value (secrets never stored), actor_id, at. Used for “what changed in the last 24 hours” alongside inventory diffs.

## Indexing (minimum)

- Guest `(cluster_id, vmid)`, `(name)`, `(node_id, status)`
- Alert `(status, severity, object_id)`
- AuditEvent `(at)`, `(action)`, `(actor_user_id)`
- ApprovalRequest `(status, expires_at)`

## Migrations and backup

Every schema change is an Alembic revision. Backup/restore of the SQLite file and secret store is defined in [DISASTER_RECOVERY.md](DISASTER_RECOVERY.md). Tests use an isolated file or in-memory DB with migrations applied.

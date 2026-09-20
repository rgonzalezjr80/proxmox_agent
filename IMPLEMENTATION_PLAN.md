# Implementation Plan

Phased delivery for the local Proxmox homelab agent. Each phase has explicit exit criteria. **Do not start a phase until the previous phase’s exit criteria are met**, except that documentation in Phase 1 is the current work.

Related: [ARCHITECTURE.md](ARCHITECTURE.md), [SECURITY.md](SECURITY.md), [TEST_PLAN.md](TEST_PLAN.md).

## Global rules for every phase

- Follow repository conventions once code exists (`src/proxmox_agent`, pytest, Ruff/type checking).
- No secrets in git. `.gitignore` and `.env.example` land at the start of Phase 2 scaffolding.
- Default `AGENT_MODE=read_only`. Mutating tools stay unregistered or flag-gated until their phase.
- Tests required for new behavior: unit always; integration when talking to adapters; e2e for critical workflows.
- Structured logging and typed errors from the first application commit.
- The language model is not wired to any privileged adapter.
- After any fix or change, produce a detailed operator report (what was broken, how it was fixed). See [CHANGE_REPORTS.md](CHANGE_REPORTS.md).

## Phase 1 — Discovery and planning

**This phase.** Repository was empty (`README.md` only). Stack confirmed: Python 3.12, FastAPI, SQLAlchemy, SQLite, Jinja2/HTMX, Ollama, `httpx` Proxmox adapter.

### Deliverables

The eleven root documents listed in the README.

### Exit criteria

- Documents describe architecture, security, threats, data, API, approval, tests, DR, and configuration.
- No application code required.
- Operator has reviewed and approved before scaffolding.

## Phase 2 — Scaffolding and read-only inventory

### Scaffolding (first commits of this phase)

- `pyproject.toml` (runtime: FastAPI, Uvicorn, Jinja2, httpx, SQLAlchemy, Alembic, Pydantic Settings, Argon2, cryptography, APScheduler, structlog)
- `.gitignore`, `.env.example`, Docker Compose, Dockerfile
- App factory, `/health`, `/ready`, structured logging
- Alembic baseline from [DATA_MODEL.md](DATA_MODEL.md) (inventory subset is enough to start; remaining tables may land in the same baseline to avoid churn)
- Pytest harness, Ruff, `mypy` or `ty` as chosen in `pyproject.toml`
- Local and test config files; production **template** without secrets

### Read-only inventory

- Proxmox token from secret store (Auditor)
- TLS verify default on; optional CA file
- Cluster, node, VM, LXC, storage, network discovery
- Guest NICs, tags, descriptions, snapshots, backup volids when the API provides them
- OS/IP/hostname best-effort via qemu-guest-agent; incomplete data is allowed and visible
- Periodic sync + manual `POST /api/v1/inventory/sync`
- Basic inventory API and HTMX UI
- Stale marking / tombstones when objects disappear

### Exit criteria

- Fake-Proxmox integration tests cover sync of a two-node fixture cluster
- Live cluster is **not** required for CI
- UI lists nodes and guests after a successful sync against the fake API
- Mutating Proxmox methods are not called (test or allowlist assertion)
- `/health` and `/ready` exist

## Phase 3 — Monitoring

- Health checks: node reachability, guest status, CPU/memory/disk, storage capacity
- Thresholds in config (warning / critical)
- Alert documents, event history, ack, suppression, maintenance windows
- Scheduled checks via `JobQueue`
- Dashboard views for alerts and stale inventory
- `NotificationPort` with Null and Log adapters only
- Backup success/failure and snapshot age when data exists
- Placeholders/fields for cert expiry, failed logins, drift — implement when a reliable signal exists; do not invent metrics

### Exit criteria

- Threshold tests with fixture metrics
- Ack/suppress/maintenance behavior tested
- Notifications can be asserted via a recording adapter
- No external notification vendor

## Phase 4 — AI analysis

- `LlmProvider` for Ollama; cloud adapter code may exist but remains disabled
- Natural-language inventory queries (read tools only)
- Error / log analysis with untrusted-content wrapping
- Risk classification of **recommendations** (not execution)
- Distinguish facts vs assumptions; ask for missing evidence
- Conversations stored and optionally linked to incidents
- Fail-safe if Ollama is down

### Exit criteria

- Read tools cannot start/stop/delete regardless of model output
- Injection fixture: VM description containing “delete all VMs” does not invoke delete
- Ambiguous name “web” with two guests → clarification, no tool
- Analysis never claims a fix was applied

## Phase 5 — Approval and auditing

- Local user auth, RBAC (admin / operator / viewer)
- Approval requests, approve/reject, expiry, preview hash
- Command/API preview on the approval page
- Dry-run mode
- Hash-chained audit log
- Emergency stop (config, file, API/UI)
- Rate limits
- Re-auth hook for high-risk (even if no high-risk tool is enabled yet)
- Persist a change report on every terminal execution (success, failure, partial)

### Exit criteria

- Viewer cannot approve
- Execute path refuses mismatched preview hash
- Kill switch blocks a stub mutating tool used only in tests
- Audit chain verifies in tests
- CSRF on cookie POSTs
- Stub execute writes a change report with required fields (see [CHANGE_REPORTS.md](CHANGE_REPORTS.md))

## Phase 6 — Controlled actions (one at a time)

Implement **one** action family per change set. Each family includes: input validation, permission, unique target, approval, optional snapshot, timeout, progress, result verification, audit, error handling, rollback guidance, and a stored change report.

### Order (low risk first)

1. `vm.start` (already-known guest)
2. `vm.shutdown` (ACPI/graceful if available)
3. `vm.reboot`
4. `vm.pause` / `vm.resume`
5. Tags and description updates
6. Snapshot create
7. Snapshot restore (medium)
8. Clone from **approved** template (medium)
9. Create from **approved** template (medium)
10. CPU/memory modify (disruptive → medium/high)
11. Disk resize where supported
12. NIC add/remove
13. Migrate between nodes
14. Startup order / HA-ish settings that Proxmox exposes
15. Backup job trigger
16. `vm.delete` last among guest lifecycle (high; typed confirm name+VMID)

Guest user-account actions are **Phase 7**, not this list.

Windows/Linux **guest** creation is “create from approved template” only — no ad-hoc ISO wizard in the first pass.

### Per-action exit criteria

- Flag default off
- Preview shows exact API calls
- Unique-target tests
- Approval required; high-risk re-auth
- Handler verifies post-condition (e.g. status `running`)
- Failure and partial-failure tests
- Rollback/recovery text on the approval record
- Change report on success, failure, and partial

**Privileged Proxmox token is introduced only when the first mutating family is enabled**, documented as a config change, least privilege for that family if Proxmox ACL allows.

## Phase 7 — Guest OS management

Only after Phase 6 start/stop/reboot are stable in read-approved use.

- Linux: SSH allowlisted command templates (id, groups, keys — no free-form shell)
- Windows: WinRM or equivalent allowlisted operations
- Directory integration only if explicitly configured later
- Password reset via secret workflow: generate, store encrypted, show once or out-of-band; never log
- Same approval + audit as VM actions
- Command allowlists; no model-supplied scripts

### Exit criteria

- Secrets never appear in audit detail or chat
- Ambiguous username/host refuses
- Tests use fake SSH/WinRM adapters

## Phase 8 — Scenario testing and reporting

- Dry-run and simulated failures (VM, node, network, disk-full, backup fail, service fail)
- Recovery and backup-restore **tests** against fixtures or an isolated lab, never unlabeled production
- Reports: inventory, health, capacity, backups, incidents, **changes/fixes**, approvals, agent activity
- Exports: Markdown, HTML, JSON, CSV; PDF if practical (optional library)
- Incident and change-report fields as specified in [CHANGE_REPORTS.md](CHANGE_REPORTS.md)
- After every executed (or failed/partial) change, a stored report plus chat/UI summary; optional notify via `NotificationPort`
- Scheduled reports as jobs

### Exit criteria

- Every simulation audit row has `simulated=true`
- Production destroy APIs are not called from simulation tests
- At least one e2e report generation test
- Closing an incident or finishing an execution without a change/incident report fails the test

## Suggested timeline (indicative, not a promise)

| Phase | Relative size |
|---|---|
| 1 | Done when documents are accepted |
| 2 | Largest initial engineering slice |
| 3 | Medium |
| 4 | Medium (depends on Ollama quality; keep tools strict) |
| 5 | Medium |
| 6 | Long; many small PRs |
| 7 | Medium, high caution |
| 8 | Medium |

## Dependencies to add in Phase 2 (planned, not installed yet)

fastapi, uvicorn, jinja2, python-multipart, httpx, sqlalchemy, alembic, pydantic-settings, argon2-cffi, cryptography, apscheduler, structlog, pytest, pytest-asyncio, httpx/respx or a local fake server, ruff.

Optional later: weasyprint or similar for PDF; asyncssh; pywinrm.

## What this phase does not do

No `pyproject.toml`, Docker, or Python package until Phase 1 is approved and Phase 2 starts.

# proxmox_agent

Local-first AI agent for a Proxmox homelab: inventory, documentation, monitoring, analysis, and later **approved** administration.

The agent is designed to run inside the lab. Cloud models and SaaS notification providers stay **off** unless you explicitly configure and approve them. It starts in **read-only** mode. Destructive actions cannot run until you enable them one family at a time and approve each request.

## Status

**Phase 1 — planning documents only.** There is no application package, Docker image, or live API yet. Do not point a Proxmox token at this repository until Phase 2 exists and you have reviewed [SECURITY.md](SECURITY.md).

## Confirmed stack (when implementation starts)

Python 3.12, FastAPI, SQLAlchemy + Alembic, SQLite, Jinja2 + HTMX, Ollama (optional), `httpx` to the Proxmox API. See [ARCHITECTURE.md](ARCHITECTURE.md).

## What it will do

1. Inventory the cluster, nodes, VMs, LXC, storage, and networks
2. Keep searchable infrastructure documentation
3. Monitor health and raise alerts (notifications behind an interface)
4. Analyze errors and logs without treating them as instructions
5. Propose fixes with risk and rollback — **not** apply them automatically
6. After approval: manage VMs, then later guest accounts
7. Simulate failures with labels so they cannot be confused with production
8. Generate reports and keep an append-only audit log
9. After every fix or change, store and show a detailed report of what was broken and how it was fixed ([CHANGE_REPORTS.md](CHANGE_REPORTS.md))

## What it will not do (defaults)

- Guess a VM or user when the name is ambiguous
- Delete or mutate anything without an approval record
- Execute arbitrary shell or API from the model
- Log or display secrets
- Require Ollama for inventory to work
- Run destructive tests against production

## Planning documents

| Document | Contents |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | Layers, adapters, failure modes, deploy topology |
| [SECURITY.md](SECURITY.md) | RBAC, allowlists, secrets, injection, kill switch |
| [THREAT_MODEL.md](THREAT_MODEL.md) | Assets, STRIDE, residual risk |
| [IMPLEMENTATION_PLAN.md](IMPLEMENTATION_PLAN.md) | Phases 1–8 and exit criteria |
| [DATA_MODEL.md](DATA_MODEL.md) | Entities and SQLite conventions |
| [API_DESIGN.md](API_DESIGN.md) | HTTP contracts and error codes |
| [APPROVAL_WORKFLOW.md](APPROVAL_WORKFLOW.md) | Risk, preview, decide, execute |
| [TEST_PLAN.md](TEST_PLAN.md) | Unit/integration/e2e; no live cluster in CI |
| [DISASTER_RECOVERY.md](DISASTER_RECOVERY.md) | Agent backup/restore |
| [CONFIGURATION.md](CONFIGURATION.md) | Env vars, flags, thresholds |
| [CHANGE_REPORTS.md](CHANGE_REPORTS.md) | Required report after every fix or change |
| [COWORKER_PROMPT.md](COWORKER_PROMPT.md) | Prompt to hand another engineer or coding agent |
| [COWORKER_PROMPT.pdf](COWORKER_PROMPT.pdf) | Same prompt as a printable PDF |

## Repository inspection (Phase 1 start)

The git history began with a single-line README. No languages, tests, or dependencies were present. The first implementation work after these documents are **approved** is Phase 2: scaffolding plus read-only inventory.

## Run / deploy

Not applicable until Phase 2. Planned: Docker Compose on a **management VM** (not on the hypervisors), Proxmox API token with `PVEAuditor`, TLS verification on.

## License / operators

Private homelab project. Treat API tokens as production credentials even at home.

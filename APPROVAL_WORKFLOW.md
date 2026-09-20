# Approval Workflow

No disruptive or destructive operation executes because the model “agreed” or the operator typed a command in chat. Execution happens only after this workflow succeeds. See [SECURITY.md](SECURITY.md) and [API_DESIGN.md](API_DESIGN.md).

## When approval is required

| Category | Approval |
|---|---|
| Read inventory, docs, metrics, audit, reports | No |
| Documentation overlay (purpose/owner in agent DB only) | No extra Proxmox approval; operator+ role; audited |
| Inventory sync | Operator+; no Proxmox mutation |
| Any Proxmox or guest mutation | **Yes** |
| Alert ack/suppress | Operator+; audited; not a cluster mutation |
| Simulations labeled `simulated=true` | Yes, against the simulation framework; never silently aimed at production |

`AGENT_MODE=read_only` rejects mutation before a request is created, with a clear error.

## Risk classes

Assigned statically per action family (not by the model). The model may **display** risk; the server value wins.

| Risk | Examples | Approver | Re-auth | Snapshot/backup default |
|---|---|---|---|---|
| Low | `vm.start`, tag/description update, `vm.resume` | operator or admin | No | Optional |
| Medium | `vm.shutdown`, `vm.reboot`, `vm.pause`, clone, snapshot create, NIC change, migrate | admin (operator may **propose** only unless a future flag allows operator-approve-medium) | No | Recommended; offer create snapshot |
| High | `vm.delete`, disk resize, snapshot restore, template create that consumes storage, guest user create/disable/delete, password reset | admin | **Yes** | Required unless explicit audited waiver |

Safe default: medium is admin-approve only. This can be relaxed later by config, default remains strict.

## Request states

```
draft ─► pending ─► approved ─► executing ─► succeeded
                 │           │            └─► failed
                 │           │            └─► partial
                 │           └─► cancelled
                 ├─► rejected
                 └─► expired
```

- `draft`: orchestrator assembling preview (not yet visible as actionable, or visible as “preparing”).
- `pending`: operator/admin must decide. Expires at `expires_at` (default 15 minutes; configurable).
- `approved`: decision recorded; job enqueued. Not yet success.
- `executing`: handler in progress.
- Terminal: `succeeded`, `failed`, `partial`, `rejected`, `expired`, `cancelled`.

Expired and rejected requests cannot be executed. A new request is required (new preview hash).

## Preview contents (mandatory)

The approval UI and `GET /api/v1/approvals/{id}` must show:

1. What the agent understood
2. Exact target (cluster, node, VMID, name, kind)
3. Proposed action family and human description
4. Reason (operator prompt + structured intent)
5. Expected impact
6. Risk level (server)
7. Commands or API calls that will be executed (method, path, redacted body)
8. Whether a backup or snapshot will be created
9. Rollback or recovery plan
10. Dry-run vs live, simulated vs real
11. Prerequisites (guest agent, quorum, free storage, flags)
12. Expiry time
13. Re-auth required or not

The canonical JSON of items 2, 3, 7, 10 is hashed as `preview_hash`. Execute refuses a mismatch.

## Decision

`POST /api/v1/approvals/{id}/decide`:

- `rejected`: terminal; audit
- `approved`:
  - Role sufficient for risk
  - Kill switch off
  - Flags and mode allow the family
  - Target still uniquely present
  - High-risk: password (or equivalent) checked in the same request
  - Optional typed confirmation: for delete, operator must send `confirm_name` and `confirm_vmid` matching the target

The language model is not a party to this POST.

## Execution

1. Load request; status must be `approved`.
2. Re-check policy (flags, kill switch, RBAC, hash, target).
3. Optional snapshot/backup step; if it fails, **do not** continue high-risk action; mark `failed` with reason.
4. Perform allowlisted API calls with timeout.
5. Verify post-condition (e.g. guest status from API).
6. Write audit: start, each call, verify, terminal status.
7. Write a change report (what was broken or requested, what ran, verified result, rollback). See [CHANGE_REPORTS.md](CHANGE_REPORTS.md).
8. Only then may chat or UI say “succeeded”, and only with the report id.

Partial completion (e.g. snapshot created, delete failed) is `partial` with a recovery note and a change report. Never implied success.

## Dry-run

When `dry_run=true`, handlers build the preview and, where the upstream supports it, use Proxmox dry-run or skip the write. Status should become `succeeded` with `dry_run=true` in audit **without** changing cluster state. Tests assert no write methods.

## Simulation

Simulation runs set `simulated=true` on the request and all audit rows. UI labels **SIMULATED** in banners. Simulation adapters must not be the production `ProxmoxClient` write path.

## Chat behavior

For every mutating user utterance, the assistant response includes the preview fields and a link/id to the approval record. It does **not** execute.

If the target is ambiguous or missing, **no** approval is created; the agent asks for VMID/node.

## Cancellation and emergency stop

- Requester or admin may cancel `pending` or `approved` (not yet executing) requests.
- Kill switch prevents new executes; in-flight handlers should check a cancellation token between steps where practical.

## Rate limits

Per user: bounded pending approvals and bounded executes per window, to reduce panic-clicking and token abuse.

## Audit events (minimum)

`approval.created`, `approval.approved`, `approval.rejected`, `approval.expired`, `approval.cancelled`, `execution.started`, `execution.call`, `execution.verified`, `execution.succeeded`, `execution.failed`, `execution.partial`, `change_report.written`.

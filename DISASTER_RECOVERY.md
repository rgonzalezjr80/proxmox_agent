# Disaster Recovery

How to recover the **agent** and how the agent should talk about recovering **guests**. This is not a substitute for Proxmox backup policy on the cluster itself.

## What must be backed up

| Item | Location (planned) | Priority |
|---|---|---|
| SQLite database | `data/proxmox_agent.db` (configurable) | Critical |
| Encrypted secret store | `data/secrets` or encrypted table + same DB | Critical |
| Alembic version (inside DB) | with the DB | Critical |
| Configuration | `.env` or systemd env; **not** in git | Critical |
| Master key | systemd credential or password manager — **not** only next to the DB | Critical |
| Emergency-stop file | optional; recreatable | Low |
| Ollama models | separate; agent runs without them | Low |

If the SQLite file and the master key are both lost, inventory history, approvals, and secrets are unrecoverable. Proxmox itself is unaffected.

## Backup procedure (agent)

Until automation exists in Phase 8, operators should:

1. Stop writers if possible (`AGENT_EMERGENCY_STOP` or stop the compose service) so the SQLite file is consistent. If stopping is impossible, use SQLite online backup API once implemented.
2. Copy `data/proxmox_agent.db` (and `-wal`/`-shm` if present) to encrypted offline storage.
3. Copy secret store files if separate from the DB.
4. Confirm the master key is available in a password manager. **Do not** place the master key in the same backup tarball as the database unless that tarball is encrypted with a different key.
5. Record the application version / git revision with the backup.
6. Start the agent again and clear the kill switch if it was only used for backup consistency.

Frequency: at least daily once the agent holds documentation and audit history; before enabling Phase 6 write tokens.

## Restore procedure (agent)

1. Deploy the same or newer application version (Alembic upgrades forward; do not skip migrations).
2. Stop the agent.
3. Replace the data volume with the backed-up DB and secret store.
4. Provide `PROXMOX_AGENT_MASTER_KEY` matching the backup.
5. Start the agent. Check `/ready`.
6. Run a **read-only** inventory sync. Diff against Proxmox; do not assume agent docs are newer than the cluster.
7. Audit chain: run the verify routine (to be implemented) and record the result.
8. Rotate the Proxmox token if the old management VM may have been compromised.

Do not restore a DB over a running process.

## Master key loss

Without the key, encrypted secrets cannot be decrypted. Recovery:

1. Restore key from password manager, **or**
2. Revoke old Proxmox tokens on the cluster, create new tokens, bootstrap an empty secret store, re-enter secrets. Agent DB inventory can remain; secret ciphertext rows are useless and should be replaced.

## Proxmox / guest recovery guidance (product behavior)

The agent must not imply it can undo a delete without a snapshot or backup.

For high-risk actions it will, when technically possible:

- Create a snapshot or verify a recent backup before proceeding
- Record volid / snapshot name on the approval
- On failure, print recovery steps (restore snapshot, restore vzdump, do not retry delete)

Simulated recovery tests in Phase 8 use fixtures or an isolated VM, never unlabeled production.

## Cluster disaster vs agent disaster

| Event | Agent role |
|---|---|
| Management VM dies | Restore VM from hypervisor backup; restore data volume; same as restore procedure |
| One Proxmox node dies | Inventory will show node offline; do not auto-migrate |
| Accidental VM delete via Proxmox UI (bypassing agent) | Agent tombstones on next sync; cannot magically undelete |
| Ransomware on management VM | Assume secrets stolen; rotate tokens/keys; restore DB from **offline** backup |

## Emergency stop during incidents

If a handler misbehaves: set `data/EMERGENCY_STOP` or POST emergency-stop. Mutating tools halt. This is not a cluster freeze; it only stops the agent.

## RTO / RPO (targets, not guarantees)

- Agent RPO: last successful backup (goal: 24h, tighten when audit/docs matter)
- Agent RTO: time to restore VM + data volume (goal: under one hour for a homelab)
- Guest RPO/RTO: whatever Proxmox backup jobs provide; the agent reports them, it does not invent them

## What we will not do

- Store the only copy of the master key in git or in `ARCHITECTURE.md`
- Use `verify=false` TLS as a recovery “workaround” without documenting the risk
- Restore production guests as part of CI

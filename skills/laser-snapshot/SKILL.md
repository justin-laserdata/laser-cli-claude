---
name: laser-snapshot
description: >-
  Manage diagnostic snapshots with `deployment monitor snapshot`. A snapshot is an HTML report of each node's system state, runtimes, certificates, network, kernel parameters and logs. Snapshots are not data backups, use /laser-backup for restore points.
---

# laser-snapshot

A *snapshot* is a diagnostic report of a deployment's nodes: system and hardware, runtimes and services, kernel parameters, certificate chains, network configuration and recent logs, packaged as an interactive HTML report. Secrets are masked by default.

**A snapshot is not a backup.** It captures system state, not data. For point-in-time recovery of storage volumes, see `/laser-backup`.

## Verbs

```sh
laser deployment monitor snapshot list -d orders -o json
laser deployment monitor snapshot create -d orders
laser deployment monitor snapshot create -d orders --node 3
laser deployment monitor snapshot create -d orders --include-iggy false
laser deployment monitor snapshot download -d orders 615380456123456861
laser deployment monitor snapshot delete -d orders 615380456123456861 --yes
```

`create` captures every node, or the repeated `--node` values. Two booleans, both on by default:

- `--redact-secrets <true|false>`: mask credentials, tokens and keys.
- `--include-iggy <true|false>`: include a streaming server snapshot.

Only one snapshot runs for a deployment at a time. After `create`, read `snapshot list -o json` until the new entry completes. Cap the wait at about 10 minutes, then warn and stop polling.

`download` prints a signed download address. It needs no `-o json`.

The plan sets how many snapshots a deployment keeps and for how long. Old ones expire on their own. See <https://docs.laserdata.com/deployments/snapshots>.

## Hard rules

1. Never turn off `--redact-secrets` unless the user asks for it explicitly.
2. The download address is a credential for the report. Show it once, suggest saving straight to a file the user names (`curl -L "<address>" -o snapshot.zip`), and don't repeat it in later answers.
3. `delete` is irreversible. Show the snapshot's name and creation time, ask the user to confirm, and only then run it with `--yes`. Without a terminal the CLI refuses a delete that lacks `--yes`.
4. There is no restore from a snapshot. For data rollback, hand off to `/laser-backup`.
5. Reading and downloading need `deployment:read`, creating and deleting need `deployment:manage`.

## Common asks

- "diagnose deployment orders": `snapshot create -d orders`, wait for it to complete, then `snapshot download` and tell the user to open the HTML report.
- "snapshot only the broken node": `snapshot create -d orders --node 3`.
- "show me snapshots": `snapshot list -o json`, render name, created and status.
- "delete old ones": list, pick by creation time, delete one id at a time with per-entry confirmation.

## Don't

- Don't conflate snapshot with backup. Different commands, different artifacts.
- Don't save the download address to a file the user did not name.
- Don't poll forever on a running snapshot.

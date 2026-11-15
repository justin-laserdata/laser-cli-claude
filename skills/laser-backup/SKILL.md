---
name: laser-backup
description: >-
  Manage point-in-time backups of a deployment's storage volumes with `deployment settings backup`. List, show, create, restore and delete, with confirmation before every destructive step. Needs AWS Network Drive storage and backups in the plan.
---

# laser-backup

A *backup* is a point-in-time copy of a deployment's storage volumes. It is the artifact you restore from after data loss. For a diagnostic report of system state, see `/laser-snapshot`.

## Availability

- AWS deployments on Network Drive storage. Local NVMe instance storage cannot be backed up.
- The plan must include backups, and it sets how many a deployment keeps and for how long. See <https://docs.laserdata.com/deployments/backups>.

Check before offering a backup: `laser deployment settings plan show -d orders -o json` shows the storage, and `laser deployment show -d orders -o json` the cloud.

## Verbs

```sh
laser deployment settings backup list -d orders -o json
laser deployment settings backup show -d orders 615380456123456901 -o json
laser deployment settings backup create nightly -d orders --expires-in-days 7
laser deployment settings backup create pre-upgrade -d orders --remarks "Before the tier change"
laser deployment settings backup create node-2 -d orders --node 2
laser deployment settings backup restore -d orders 615380456123456901 --yes
laser deployment settings backup delete -d orders 615380456123456901 --yes
```

`create` takes a name, every node by default or one `--node`, optional `--remarks`, and `--expires-in-days` (the plan retention by default). It runs in the background: read `backup list -o json` until the new entry completes. Only one backup runs for a deployment at a time. If `create` is refused because one is in flight, wait and retry.

## Hard rules

1. `restore` rolls the deployment's data back to the backup. Show the backup's name and creation time, say plainly that everything written since then is lost, ask the user to type the deployment name, and only then run it with `--yes`.
2. `delete` is irreversible. Same confirmation as restore.
3. Without a terminal the CLI refuses `restore` and `delete` unless `--yes` is passed. Never add `--yes` before the user agreed.
4. If the deployment is not on AWS Network Drive storage, refuse and explain. A storage change goes through `deployment settings plan upgrade` (see `/laser-deploy`).
5. Before a risky change (plan upgrade, configuration activation, runtime upgrade), offer a backup first.

## Common asks

- "back up orders": `laser deployment settings backup create manual -d orders`, then poll the list until it completes.
- "restore orders to last night": `backup list -o json`, resolve the name to an id, confirm, then `backup restore` with `--yes`.
- "delete old backups": one id at a time with per-entry confirmation.
- "schedule backups": the CLI has no schedule. Suggest a cron job that runs `laser deployment settings backup create nightly -d orders --expires-in-days 7` with `LD_API_KEY` from a secret store.

## Don't

- Don't conflate backup with snapshot.
- Don't suggest a backup for local NVMe storage. It will fail.
- Don't bulk-delete without per-entry confirmation.
- Don't pass `--silent`, you need the JSON to track status.

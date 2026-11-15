---
name: laser-troubleshoot
description: >-
  Diagnose a struggling deployment. Pulls health, metrics, heartbeats, throughput, runtime logs, activity, tasks and access rules through `deployment monitor` and `deployment connect`, summarizes health, names the likely cause, and proposes one concrete next step, such as a runtime restart on a single node.
---

# laser-troubleshoot

Argument: a deployment name or id. Pass it as `-d` on every command. Without one, use the deployment scope of the context, and if there is none, start from the workspace list:

```sh
laser deployment list --health degraded,unavailable -o json
```

If `laser auth status` fails, stop and point the user at `/laser-onboard`.

## Gather (in parallel where possible)

```sh
laser deployment show -d orders -o json
laser deployment monitor health -d orders -o json
laser deployment monitor metrics --full -d orders -o json
laser deployment monitor heartbeats -d orders -o json
laser deployment monitor throughput -d orders --interval 5m -o json
laser deployment monitor logs -d orders --level error -o json
laser deployment monitor activity -d orders -o json
laser deployment monitor task list -d orders --active -o json
laser deployment connect access-rule list -d orders -o json
laser deployment connect network -d orders -o json
laser deployment settings show -d orders -o json
```

Do not pass `--silent`. Read the JSON and summarize the findings. `metrics --full` includes metrics, throughput, health, and heartbeats. Report null sections with their errors as missing evidence. Use individual commands when you need node, runtime, or time filters.

## Drill down

```sh
laser deployment monitor readiness -d orders --node 3 -o json
laser deployment monitor metrics -d orders --node 3 --runtime iggy --from 2026-11-11T08:00:00Z -o json
laser deployment monitor heartbeats -d orders --node 3 --runtime connectors -o json
laser deployment monitor logs -d orders --runtime connectors --level warn --message timeout -o json
laser deployment monitor logs -d orders --node 3 --from 2026-11-11T08:00:00Z --to 2026-11-11T09:00:00Z -o json
laser deployment monitor task show -d orders 615380456123456871 -o json
laser deployment monitor activity -d orders --resource topic --action deleted -o json
```

- `--node` narrows metrics and heartbeats to one node. Adding `--runtime` (`iggy` for the streaming server, `connectors`, or `connector:<id>`) pages that runtime's history on the node between `--from` and `--to`.
- `logs` reads `iggy` logs by default. Filter with `--runtime`, `--node`, `--scope`, `--level` (`error`, `warn`, `info`, `debug`, `trace` or `any`), `--message`, `--from` (inclusive) and `--to` (exclusive). Logs come in pages, run the command again for newer lines.
- `throughput` returns network throughput in buckets from `10s` to `1d`, the last hour in 30 second buckets by default, at most 1440 buckets.
- `activity` lists the operations LaserData ran on the deployment and the changes inside it.
- `task list` shows node tasks newest first, `--active` only the running ones.

## Check, in order

1. **Status and running operation** (`deployment show`, `monitor health`). `initialized` is ready and `failed` is a failed operation. Anything else is an operation in progress. Quote how long it has been in that state and let the user judge whether it is stuck.
2. **Settings requested against applied** (`deployment show`, `settings show`). A rename, move, data access or metadata sync change that stays `pending` explains a deployment that "did not take" a change. `laser deployment watch -d orders -o json` waits for it.
3. **Serving health** (`monitor health`). Nodes, runtimes and connectors that are not serving.
4. **Heartbeats**. Each node and runtime should report recently. Quote the gap between now and the last heartbeat, don't invent a staleness threshold.
5. **Metrics and throughput**. CPU, memory and disk figures are evidence, not verdicts. Quote the sample and its time.
6. **Activity and tasks**. The last failed operation or a task still running explains most "why is it stuck" questions.
7. **Logs**. Quote the most recent error line verbatim.
8. **Access rules, network and data access**. "I can't connect" is usually no rule for the caller's address, or an expired `valid_to`. When the rules look right, `connect network` confirms the platform network a peered client must reach. Data commands that fail through the data access proxy may mean data access is off (`settings show`).

## Output shape

Return a short report:

- status: the state and since when, such as "initialized, operation finished 2h ago"
- health: green, amber or red with the evidence, such as "amber: cpu 92% on node 3 for 40 min"
- last operation: one line from activity or tasks
- recent errors: up to three quoted log lines
- likely cause: one sentence
- next step: one concrete `laser` command

If the deployment is fine, say so and stop. Don't pad.

## Likely cause to next step

- CPU or memory pegged: price a larger compute profile with `deployment create preview`, then `laser deployment settings plan upgrade -d orders --tier large --wait`. See `/laser-deploy`.
- Disk near full: `laser deployment settings plan upgrade -d orders --storage-type network_balanced --storage-size-gb 1000 --wait`. Storage cannot shrink.
- The caller cannot connect: add a rule for the caller's address with `/laser-access`.
- One runtime wedged on one node while the deployment is up: `laser deployment settings runtime restart -d orders --runtime iggy --node 3 --yes`. `--runtime connectors` or `--runtime connector:<id>` target connectors. `--node` repeats.
- A settings change stuck in `pending`: `laser deployment watch -d orders --timeout-mins 30 -o json`, then report what stays pending.
- Data commands fail while the deployment is healthy: check data access in `settings show`, then retry the data command with `--direct` (needs an access rule for the caller's address).
- No clear cause: `/laser-snapshot` for a full diagnostic snapshot.

A runtime restart, stop or upgrade is destructive: without a terminal the CLI refuses it unless `--yes` is passed. Print the command, say which nodes and runtime it touches, ask the user to type the deployment name, and only then run it with `--yes`.

## Don't

- Don't fabricate root causes. If the data points nowhere, say so and suggest `/laser-snapshot`.
- Don't loop on `metrics` or `heartbeats`. One read is enough for a first diagnosis.
- Don't restart every node when one node is the problem.
- Don't suggest deleting the deployment to fix it.

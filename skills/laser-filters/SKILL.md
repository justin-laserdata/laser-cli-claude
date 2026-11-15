---
name: laser-filters
description: >-
  Consumer filters that decide which records a consumer group receives. Tries a filter without saving it (validate, test, preview), saves and revises it in the catalog, binds consumer groups to a revision, pauses, resumes, archives and drops filters, and reads bindings, operations and metrics.
---

# laser-filters

A *consumer filter* is a policy the streaming server applies to every read of a consumer group. Filters live in a catalog: each saved filter has an id and immutable revisions, and a group is bound to one revision. A filter definition is a JSON file.

## Try before saving

```sh
laser filter validate --filter paid-only.json -d orders
laser filter test --filter paid-only.json --payload '{"status":"paid","amount":120}' -d orders -o json
laser filter test --filter-id 7 --revision 2 --payload '{"status":"refunded"}' -d orders -o json
laser filter preview --filter paid-only.json --stream checkout --topic events --max-records 20 -d orders -o json
laser filter preview --filter paid-only.json --stream checkout --topic events --partition 2 --from-offset 5000 --explain -d orders -o json
```

`validate` checks a definition and prints its digest. `test` judges one sample message and explains the verdict: `--payload` for JSON, `--payload-base64` for CBOR, Avro or Protobuf, `--headers` for typed headers. `preview` runs the filter over stored messages of one partition, joins no group and stores no offset: `--from-offset`, `--max-examined` (at most 10000) and `--max-records` (at most 100), with `--explain` for every examined message. `test` and `preview` also take a saved filter with `--filter-id` and `--revision` in place of `--filter`. Nothing here is stored.

## Read the catalog

```sh
laser filter list --state active -o json
laser filter list --live -d orders -o json
laser filter show 7 -d orders -o json
laser filter revisions 7 -d orders -o json
laser filter bindings 7 -d orders -o json
laser filter binding --stream checkout --topic events --group billing -d orders -o json
laser filter operation 1849302 -d orders -o json
laser filter metrics -d orders
```

`filter list` reads the workspace's filters from synced metadata, or one deployment's catalog with `--live`, filtered by `--state` (`active`, `archived` or `dropped`) and `--name`. `binding` shows the filter one group is bound to. `operation` reads the outcome of a catalog change by its id. `metrics` prints the deployment's filter metrics in Prometheus text. `laser group policy checkout events billing -d orders -o json` shows what the streaming server applies to that group.

## Change the catalog

```sh
laser filter register --name paid-only --filter paid-only.json --description "Paid orders" -d orders
laser filter revise 7 --filter paid-only-v2.json -d orders
laser filter bind 7 --stream checkout --topic events --group billing --revision 1 -d orders
laser filter configure-group --stream checkout --topic events --group fraud --name fraud-policy --filter fraud.json -d orders
laser filter describe 7 --description "Paid orders only" -d orders
laser filter pause 7 --revision 2 -d orders --yes
laser filter resume 7 --revision 2 -d orders
laser filter unbind --stream checkout --topic events --group billing -d orders --yes
laser filter archive 7 -d orders --yes
laser filter drop 7 -d orders --yes
```

- `register` saves a filter at revision 1. A filter name is never reused.
- `revise` adds a revision. Bound groups keep their revision until bound again.
- `bind` attaches a consumer group to a revision, the latest by default.
- `configure-group` saves a filter owned by one group and binds the group in one step.
- `pause` makes the groups bound to a revision refuse new filtered reads until `resume`.
- `unbind` releases a group, so its consumers receive every record.
- `archive` keeps the filter running for bound groups and refuses new bindings and revisions.
- `drop` releases every bound group, and the id is never reused.

Catalog changes take `--operation-id` and are resent under the same id while their outcome is unknown, so a lost reply never applies twice. If a change ends with its outcome unknown, read it with `filter operation` before running it again, or run it again with the same `--operation-id`.

## Hard rules

1. Always `validate`, `test` and `preview` a definition against real messages before `register`, `revise` or `configure-group`. Show the user what it lets through and what it drops.
2. Binding changes what a live consumer group receives. Print the command, name the group, and wait for approval.
3. `pause`, `unbind`, `archive` and `drop` are destructive: without a terminal the CLI refuses them unless `--yes` is passed. Say which groups are affected (from `filter bindings`), ask the user to confirm, and only then add `--yes`.
4. A dropped filter's id and a registered name never come back. Prefer `archive` when the user may want it later.
5. Never retry a catalog change with a fresh operation id after an unknown outcome. Read `filter operation` first.

## Common asks

- "only send paid orders to billing": write the definition with the user, `validate`, `test` with a paid and an unpaid sample, `preview` on `checkout`/`events`, then `register` and `bind` to `billing` after approval.
- "what filter does fraud use": `filter binding --stream checkout --topic events --group fraud -o json`.
- "roll billing back to revision 1": `filter bind 7 --stream checkout --topic events --group billing --revision 1` after approval.
- "stop filtering billing": `filter unbind` with `--yes` after the user confirms that billing then receives every record.

## Don't

- Don't register a filter that was never tested against stored messages.
- Don't drop a filter that still has bound groups without saying they will receive every record.

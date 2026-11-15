---
name: laser-query
description: >-
  Query and shape a deployment's managed data. Runs filter or SQL queries against projection indexes, registers, binds and drops projections, manages writer schemas, creates, promotes and deletes forks of the read model, and reads and writes the key-value store.
---

# laser-query

Projections turn stream records into queryable rows or graphs. A *binding* routes a source topic into projections and names the *index* a query reads. Writer schemas decode records, forks give a copy-on-write view of the read model, and the key-value store holds small entries by namespace and key.

All of it lives on one deployment and is read live through the data access proxy, or directly with `--direct`. Check `laser deployment capabilities -d orders -o json` first: it lists whether the deployment serves managed data, query, KV, graph and backends. Projection data is stored in the deployment's backends (see `/laser-connectors`).

## Queries

```sh
laser query run --index orders -d orders --where status=paid --order created_at:desc --limit 20 -o json
laser query run --index orders -d orders --where status=paid --fields order_id,amount --offset 20 -o json
laser query run --index orders -d orders --sql "SELECT * FROM <index> WHERE amount > 100" -o json
laser query run --index orders -d orders --sql "SELECT status, count(*) FROM <index> GROUP BY status" --dialect postgres -o json
laser query run --index orders -d orders --fork what-if --where status=paid -o json
laser query run --index orders -d orders --json '{"select": {"payload": true}, "page": {"limit": 20}}' -o json
```

- `--where FIELD=VALUE` keeps rows whose field equals the value, and repeats. Numbers, `true`, `false` and `null` are typed, a value in double quotes stays text.
- `--fields` picks fields, `--payload` adds the original message, `--order FIELD:asc` or `FIELD:desc` sorts and repeats.
- `--sql` runs one read-only `SELECT` or `WITH` statement. Write `<index>` where the index name goes, inside the quoted SQL. The dialect is the one the deployment's backend advertises, or `--dialect` (`postgres`, `my_sql` or `sqlite`).
- A page holds `--limit` rows, 50 by default and at most 1000. Read the next page with `--offset`.
- `--fork` reads a fork's view instead of the trunk.

## Projections

```sh
laser projection list -o json
laser projection list --kind graph -o json
laser projection list --live -d orders -o json
laser projection show orders.v1 -d orders -o json
laser projection register --file orders-projection.json -d orders
laser projection bind --file orders-binding.json -d orders
laser projection unbind --stream checkout --topic events --projection orders.v1 -d orders --yes
laser projection drop orders.v1 -d orders --yes
```

`projection list` reads the workspace's projections from synced metadata, filtered with `--kind` (`row` or `graph`) and `--name`, or one deployment's with `--live`. `register` and `bind` read JSON definitions, and the deployment applies a projection in the background. `unbind` stops routing a source topic into one projection, or into all of them without `--projection`. `drop` stops serving the projection's rows.

## Writer schemas

```sh
laser schema list -d orders -o json
laser schema show 12 -d orders -o json
laser schema register --format avro --file telemetry.avsc --name telemetry --schema-version 1 -d orders
laser schema register --format protobuf --file shop.desc --message-type shop.Order --stream checkout -d orders
laser schema decode 12 --payload-file sample.bin -d orders -o json
laser schema drop 12 -d orders --yes
```

`register` takes `--format` (`avro`, `json_schema` or `protobuf`) and a `--file` with the Avro or JSON Schema text, or a compiled Protobuf `FileDescriptorSet` with `--message-type`, and prints the new schema id. Schemas live in the deployment-wide registry, or in one stream's registry with `--stream`. `decode` decodes `--payload-file` or `--payload-base64`. `drop` removes a schema from the active set, keeps its id reserved, and stored records keep decoding.

## Forks

```sh
laser fork list --status open -o json
laser fork list --live -d orders -o json
laser fork create what-if --kind severed --table orders -d orders
laser fork put-row what-if --file row.json -d orders
laser fork promote what-if -d orders --yes
laser fork delete what-if -d orders --yes
```

`create` makes a copy-on-write fork of the read model, from the trunk or a `--parent` fork. A `continuous` fork, the default, sees the trunk grow, and a `severed` fork freezes the repeated `--table` values, or every table, at the current offsets. `put-row` writes one speculative row from a JSON file with `table`, `partition_id`, `offset`, `fields`, `payload_b64` and `tombstone`. `promote` splices the fork's rows onto the trunk, then squashes the fork. `delete` squashes it and discards its rows.

## Key-value store

```sh
laser kv namespaces -d orders -o json
laser kv scan fleet --prefix sat- -d orders -o json
laser kv get fleet sat-7 -d orders -o json
laser kv set fleet sat-7 --value '{"mode":"safe"}' --ttl 3600 -d orders
laser kv cas fleet sat-7 --value '{"mode":"nominal"}' --expect-version 3 -d orders
laser kv cas fleet sat-8 --value-file sat-8.json --expect-absent -d orders
laser kv delete fleet sat-7 -d orders --yes
laser kv delete-many fleet --prefix sat- -d orders --yes
```

`scan` reads one namespace in key order, filtered with `--prefix`, `--start`, `--end`, `--key-contains` and `--conversation`. `set` writes `--value` text or `--value-file` bytes, and `--ttl` expires the entry after that many seconds. `cas` writes only when the entry holds `--expect-version` or, with `--expect-absent`, does not exist, and prints the new version. `delete-many` deletes the entries that match the same bounds as `scan`, or the whole namespace without bounds. On a deployment that scopes namespaces by stream, pass `--stream`.

## Hard rules

1. Queries, lists, `show`, `kv get` and `kv scan` are read-only. Start there.
2. `--sql` accepts only `SELECT` or `WITH`. Don't try to write data through it.
3. Destructive commands (projection `drop` and `unbind`, schema `drop`, fork `promote` and `delete`, kv `delete` and `delete-many`) need `--yes` without a terminal. Show what is affected, ask the user to confirm, and only then add `--yes`. For `kv delete-many`, run the same bounds through `kv scan` first and quote the count. Without bounds it deletes the whole namespace.
4. `fork promote` changes the trunk that every reader sees. Treat it like a production write.
5. Prefer `kv cas` over `kv set` when another writer may hold the key.
6. Don't write projection, binding or graph definitions from memory. Start from `projection show` of an existing one.

## Common asks

- "how many paid orders today": `query run --index orders --sql` with a `count(*)` and a time filter in the backend's dialect.
- "what would happen if we cancel these orders": `fork create what-if`, `fork put-row` the changes, then `query run --fork what-if`, and `fork delete` when done.
- "what's in the fleet namespace": `kv scan fleet -o json`, paged with `--page`.
- "this payload won't decode": `schema list`, then `schema decode <id> --payload-file`.

## Don't

- Don't run unbounded `kv delete-many` without the user naming the namespace and agreeing to lose all of it.
- Don't guess an index name. It comes from the projection binding (`projection show`).
- Don't page through a large index into the chat. Aggregate with `--sql` instead.

Explore graphs by `projection.name`. Use `projection.id` only for catalog operations such as show, bind, or drop. `graph nodes` includes one-hop edges in both directions. Session-written graphs can exist without a registered projection, so an empty graph registry does not prove that a named graph has no data.

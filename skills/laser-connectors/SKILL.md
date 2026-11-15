---
name: laser-connectors
description: >-
  The Connectors and Backends tabs of a deployment. Browses the connector catalog, adds a source or sink, gives it a configuration, and deletes it. Lists, configures, enables and disables the backends that store projection data (embedded, Postgres, VeloDB).
---

# laser-connectors

Connectors move data between the deployment's streams and outside systems: a *source* writes into a stream, a *sink* reads from one. Backends are the storage engines that hold projection data for queries.

## Connectors

```sh
laser deployment connector catalog -d orders --type sink -o json
laser deployment connector list -d orders -o json
laser deployment connector add iggy.postgres.sink -d orders --type sink --name orders-to-postgres
laser deployment connector show -d orders 615380456123456891 -o json
laser deployment settings config schema -d orders --kind connector:sink:postgres -o json
laser deployment settings config create -d orders --kind connector:sink:postgres --file postgres-sink.json --activate
laser deployment connector delete -d orders 615380456123456891 --yes
```

1. `catalog` lists the connectors this deployment can add, `--type source` or `sink`.
2. `add` takes a catalog key and `--type`. `--name` and `--key` (lowercase with hyphens) are generated when omitted.
3. A new connector starts pending until it has a configuration. Read the schema, write the values file, and create the configuration with `--kind connector:<type>:<connector>`. Configuration history, activation and rollback are in `/laser-config`.
4. Health and errors of a connector are in `laser deployment monitor health -d orders -o json`, and its logs in `laser deployment monitor logs -d orders --runtime connectors --level error -o json`. Restart with `deployment settings runtime restart --runtime connector:<id>` (see `/laser-config`).

`delete` removes the connector and every configuration version.

## Backends

```sh
laser deployment backend list -d orders -o json
laser deployment backend show postgres -d orders -o json
laser deployment backend configure postgres -d orders --dsn "$PG_DSN" --ssl-mode verify-full --schema laser
laser deployment backend configure embedded -d orders --sync-url libsql://orders-example.turso.io --auth-token "$TURSO_TOKEN"
laser deployment backend configure embedded -d orders --sync-off
laser deployment backend configure velodb -d orders --dsn "$VELODB_DSN" --fe-http-url https://velodb.example.com:8030
laser deployment backend enable postgres -d orders
laser deployment backend disable embedded -d orders
```

- `embedded` is the built-in engine. `configure embedded` syncs it to Turso Cloud, and `--sync-off` stops syncing. A Turso Cloud sync URL needs an auth token with full access to that database.
- `postgres` connects the user's own Postgres 16 or newer. `--host-managed-stores` also hosts the managed stores in a new Postgres backend.
- `velodb` connects the user's own VeloDB or Doris cluster and needs `--fe-http-url`.
- Secrets come back masked. The default `***` for `--dsn` and `--auth-token` keeps the saved secret, so a change to other settings needs no secret.

Saving a backend configuration turns the backend on and briefly restarts the deployment's managed data service. The Backends page needs `deployment:config:manage`.

## Hard rules

1. Never put a DSN, password or token on the command line as literal text. Read it from an environment variable the user set (`"$PG_DSN"`), and never echo it.
2. Print every `add`, `configure`, `enable`, `disable` and `delete` and wait for approval. Say that configuring a backend briefly restarts the managed data service.
3. `connector delete` is destructive: without a terminal the CLI refuses it unless `--yes` is passed. Show the connector's name, type and state, ask the user to type the connector id, and only then run it with `--yes`.
4. Disabling a backend stops projections that write to it. Confirm with the user first.
5. Don't author connector values from memory. Pull `config schema` for the connector's kind.

## Common asks

- "send orders into Postgres": `catalog --type sink`, `add` the Postgres sink, `config schema`, write the values file with the user, `config create --activate`, then `monitor health` to see it run.
- "why is my sink failing": `monitor health -d orders -o json` for the connector's last error, then its logs.
- "store projections in our Postgres": `backend configure postgres` with the DSN from an environment variable, after approval.

## Don't

- Don't echo backend secrets or connector credentials into the chat.
- Don't delete a connector to reset it. A new configuration version or a runtime restart usually does.

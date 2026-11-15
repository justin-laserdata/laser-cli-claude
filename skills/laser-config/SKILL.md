---
name: laser-config
description: >-
  Manage versioned runtime configuration and the runtimes themselves under `deployment settings`. Lists, shows and creates configuration versions for the streaming server, connectors and single connectors from a JSON file, checks them against the schema, activates and deletes them, and starts, stops, restarts or upgrades a runtime.
---

# laser-config

A deployment's runtimes are driven by versioned configurations, grouped by *kind*. Each kind has one active version. A new version comes from a JSON file of values and is activated explicitly. Older versions stay for rollback until deleted.

## Kinds

`--kind` takes:

- `iggy`: the streaming server. The default when `--kind` is left out.
- `connectors`: the connectors runtime.
- `connector:<type>:<connector>`: one connector, such as `connector:sink:postgres`. The connector must exist first (see `/laser-connectors`).

## Configuration verbs

```sh
laser deployment settings config list -d orders --kind iggy -o json
laser deployment settings config list -d orders --kind iggy --name iggy -o json
laser deployment settings config show -d orders --kind iggy -o json
laser deployment settings config show -d orders --kind iggy --name iggy --version 3 -o json
laser deployment settings config show -d orders 615380456123456931 -o json
laser deployment settings config schema -d orders --kind iggy -o json
laser deployment settings config create -d orders --kind iggy --file iggy.json
laser deployment settings config create -d orders --kind iggy --file iggy.json --activate
laser deployment settings config activate -d orders --kind iggy --name iggy --version 3
laser deployment settings config delete -d orders --kind iggy 615380456123456931 --yes
```

- `show` without arguments returns the active configuration, with `--name` and `--version` one version, with an id one configuration.
- `schema` lists the sections and fields the kind takes. Read it before writing a values file.
- `create` reads a JSON object of values from `--file`. `--name` defaults to one picked by kind. `--activate` makes the new version active at once.
- `activate` takes the configuration name and the version number, not an id.
- `delete` takes the configuration id and removes a version that is not active.

## Runtime verbs

```sh
laser deployment settings runtime restart -d orders --runtime iggy --yes
laser deployment settings runtime restart -d orders --runtime iggy --node 2 --yes
laser deployment settings runtime stop -d orders --runtime connectors --node 2 --yes
laser deployment settings runtime start -d orders --runtime connectors --node 2 --yes
laser deployment settings runtime upgrade -d orders --version 0.9.2 --yes
```

`--runtime` is `iggy`, `connectors` or `connector:<id>`. `start`, `stop` and `restart` act on every node, or on the repeated `--node` values. `upgrade` moves a runtime (`iggy` by default) to `--version` one node at a time, and `--digest TARGET=SHA256` pins the release artifact of one target.

## Hard rules

1. Print every write (`create`, `activate`, `delete`, any runtime verb) and wait for the user's approval. A bad activation can stop a runtime from serving.
2. Check a values file against `config schema` before `create`. Report mismatches and stop on them.
3. Use `create --activate` only after the user has seen the difference against the active version from `config show`. Otherwise create, show the new version, then `activate` separately.
4. Before `activate`, confirm the version number against `config list` and show the candidate next to the active one.
5. `delete` is irreversible and refuses the active version. Pick an inactive one.
6. Runtime start, stop, restart and upgrade, and configuration deletes, are destructive: without a terminal the CLI refuses them unless `--yes` is passed. Name the runtime and the nodes, ask the user to type the deployment name, and only then add `--yes`. Restart one node before all of them when one node is the problem.
7. Offer a backup (`/laser-backup`) before a runtime upgrade.
8. Reading configuration and schemas needs `deployment:config:read`. Creating, activating and deleting need `deployment:config:manage`.

## Common asks

- "what is the streaming server running with": `config show -d orders --kind iggy -o json`.
- "show the iggy history": `config list -d orders --kind iggy --name iggy -o json`, then `config show` with `--name iggy --version <n>` for one version.
- "roll back to version 3": show version 3 next to the active one, confirm, then `config activate -d orders --kind iggy --name iggy --version 3`.
- "apply this values file": schema check, show the difference, then `config create --file values.json --activate` after approval.
- "restart connectors on node 2": confirm, then `runtime restart -d orders --runtime connectors --node 2 --yes`.

## Don't

- Don't write values files from memory. Pull `config schema` first.
- Don't activate without showing the candidate and the active version side by side.
- Don't delete the active version or bulk-delete versions.
- Don't pass `--silent`, you need the output for the next step.

Use `deployment settings config show --schema -o json` to read the field metadata before changing a configuration. `--reconfigure` with `config create --activate`, or with `config activate`, applies the active Iggy version through the existing task API. If that step fails, report that activation succeeded and retry applying the existing version after approval. Do not create duplicate versions as a retry.

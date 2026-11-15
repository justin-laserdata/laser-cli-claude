---
name: laser-cluster
description: >-
  Inspect the Cluster, Users and Roles pages of a deployment. Manage native users, passwords, permissions and governance roles. Shows the node roster and server counters, connected clients and their consumer groups, the deployment's own user accounts, the governance roles over its managed data, your own effective grants, and what the deployment serves.
---

# laser-cluster

These commands read the deployment live through the data access proxy, or directly with `--direct`. Over the proxy, user and role changes need `deployment:manage`. Over `--direct`, the CLI signs in as root only when the key holds `deployment:manage` and `deployment:credentials:read`. Otherwise it signs in as the deployment admin login `ld-admin`, which cannot manage users or roles, so user and role commands there need both permissions (see `/laser-credentials`). They cover the deployment's own accounts and roles, which are separate from organization members and roles (see `/laser-iam`).

## Cluster and clients

```sh
laser deployment cluster -d orders -o json
laser deployment client list -d orders -o json
laser deployment client show -d orders 42 -o json
laser deployment capabilities -d orders -o json
```

`cluster` shows the configured node roster and the server counters of the node that answers. `client list` shows the connections to that node, and `client show` the consumer groups one client joined. `capabilities` lists what the deployment serves: managed data, query, KV, graph, sessions, filters and backends. Check it before using a data command the deployment may not serve.

## Users and governance roles

```sh
laser deployment user list -d orders -o json
laser deployment user list -d orders --system -o json
laser deployment user show analytics -d orders -o json
laser deployment user roles analytics -d orders -o json
laser deployment user bind-roles analytics -d orders --role kv-reader --role query-reader --yes
laser deployment role list -d orders -o json
laser deployment role show kv-reader -d orders -o json
laser deployment role define -d orders --file kv-reader.json
laser deployment role delete kv-reader -d orders --yes
laser deployment whoami -d orders -o json
```

- `user list` hides the LaserData system accounts unless `--system` is passed.
- `bind-roles` replaces every governance role of a user. Passing no `--role` clears them all.
- `role` manages the governance roles over the deployment's managed data. `define` reads a role such as `{"name": "kv-reader", "grants": [...]}` from a JSON file, replaces a role of the same name, and needs the `authz:admin` grant.
- `whoami` shows your own governance roles and effective grants on the deployment.

## Native user management

```sh
laser deployment user update analytics -d orders --name analytics-reader --status active
laser deployment user permissions analytics -d orders --file permissions.json --yes
laser deployment user permissions analytics -d orders --clear --yes
laser deployment user delete analytics -d orders --yes
```

`user create` takes a username and `--new-password`, with optional `--status` and `--permissions-file`. `user password` takes `--current-password` and `--new-password`. Give password operations to the user to run privately. Do not request or put passwords in chat. `--username` and `--password` sign a direct connection in as another user instead of root or `ld-admin`.

`permissions` replaces the complete native permission tree. Read `user show` first and show the proposed change before applying it. `--clear` removes all native permissions. Deletion and permission replacement need confirmation. System accounts remain read only, including when selected by numeric ID.

## Hard rules

1. `bind-roles` replaces, it does not add. Read `user roles` first, build the full list, show the before and after, and wait for approval.
2. `role delete` removes the grants of every user bound to the role. List who holds it first, then ask the user to type the role name, and only then run it with `--yes`.
3. `bind-roles` and `role delete` are destructive: without a terminal the CLI refuses them unless `--yes` is passed.
4. Never touch the LaserData system accounts shown by `--system`.
5. Don't write a role definition from memory. Start from `role show` of a similar role and change only what the user asked.

## Common asks

- "who is connected to orders": `client list -d orders -o json`, then summarize the clients of the node that answered.
- "which consumer groups does client 42 hold": `client show -d orders 42 -o json`.
- "give analytics read access to KV": `user roles analytics` for the current roles, add `kv-reader`, confirm, then `bind-roles` with the full list and `--yes`.
- "why can't I query": `whoami -d orders -o json` for the effective grants, and `capabilities` to see whether query is served at all.

## Don't

- Don't confuse deployment users and roles with organization members and roles.
- Don't call `bind-roles` with one `--role` to "add" a role. It drops every other role.

Use `deployment role users ROLE` to read assignments. `deployment role assign ROLE USER --yes` and `deployment role revoke ROLE USER --yes` preserve other bindings. Read the current state and obtain approval before either change. Use `role define --create` when creating a new role so an existing grant set is not silently replaced. Governance grants use all, literal, or prefix resource scopes. A prefix is a starts-with match, not a glob or regular expression. Custom resource and namespace strings are supported by literal and prefix scopes.

---
name: laser-credentials
description: >-
  Read a deployment's connection details safely with `deployment connect show`: the connection string, endpoints with their ports, and the platform network. The deployment admin login by default, root only for a key with `deployment:manage`, and rotation of either with `deployment connect credentials rotate`. The password stays masked in the chat, and the full value is revealed only in the user's own terminal.
---

# laser-credentials

`laser deployment connect show` prints what a client needs to connect: the connection string, the endpoints with their ports, and the platform network. The password shows only its first three characters unless `--reveal` is passed.

## Two logins

Every deployment has two streaming-server logins, and the API key's permissions decide which ones the CLI can read.

| Login | User | What it can do | The key needs |
| --- | --- | --- | --- |
| Deployment admin | `ld-admin` | Read and write every stream, topic and message, and use every managed feature (key-value, queries, graphs, forks, filters, sessions). No user, role or server administration | `deployment:credentials:read` |
| Root | `root` | Everything, including users, roles and the server | `deployment:credentials:read` and `deployment:manage` |

Applications, Laser SDK programs and agents use the deployment admin login. Root is only for user and role administration that the console or the data access proxy cannot do.

## Verbs

```sh
laser deployment connect show -d orders -o json
laser deployment connect show -d orders --reveal
laser deployment connect show -d orders --root -o json
laser deployment connect credentials rotate admin -d orders
laser deployment connect credentials rotate root -d orders --yes
```

- `connect show` reads the deployment admin login. Its JSON has `username`, the masked `password`, the `connection_string` built from them, and `login` (`admin`). Run it yourself.
- `--reveal` prints the full password and the connection string Laser SDK takes. The user runs it in their own terminal, never you. Anything you run lands in this conversation.
- `--root` shows the root login instead, still masked unless `--reveal` is added, with `login` set to `root`. When the key lacks `deployment:manage`, the output keeps the deployment admin login and `credentials_note` says so. Say that to the user and don't retry.
- A 403 on `connect show` means the key's role lacks `deployment:credentials:read`: say so and don't retry.

## Rotation

`deployment connect credentials rotate <admin|root>` replaces the password of one login. It needs `deployment:manage` and `deployment:credentials:read` on the deployment's environment, for either login. It answers at once with the rotation queued (`state` is `queued` in the JSON), and the CLI exits 0. The rotation runs as one task on one node of the deployment, never one per node, and it does not restart the streaming server.

- `admin`: the next `connect show` returns the new password at once. Applying the rotation deletes and recreates `ld-admin`, which ends the old password, every personal access token, live sessions and earlier HTTP access tokens. Progress shows in `laser deployment monitor activity -d orders`.
- `root`: `connect show --root` keeps returning the old root password until that node has applied the change, then the new one. That ends the old root password and every root personal access token. Root's live sessions and earlier HTTP access tokens keep working until they end or expire, 1 hour by default and refreshable, so a rotation does not cut off a root session already open.
- Applications that use the old password stop connecting. They must read the new one, so tell the user which applications to update before you rotate.
- Rotating root asks the user to type the deployment name. Without a terminal it needs `--yes`, so confirm with the user first, then run it with `--yes`.
- A 403 means the key lacks one of the two permissions. A 404 means the supervisor does not know the deployment. A 409 means another operation or rotation holds the deployment: wait for it in `monitor activity` and run the rotation again.

Rotate when a password may have leaked, when someone who knew it leaves, or on the user's schedule.

## How the CLI itself reaches the data

Live data commands (streams, topics, messages, queries, filters, KV, sessions, memory and the rest) go through the LaserData data access proxy of the deployment's zone, signed in with the API key. Data access is on by default, and LaserData adds the access rule it needs. When data access is off or the proxy refuses, the CLI connects straight to the deployment with the deployment's credentials, which needs an access rule for the caller's address.

```sh
laser deployment settings show -d orders -o json
laser stream list --live -d orders --direct
```

- `--direct` skips the proxy and connects straight to the deployment. It signs in as root when the key has `deployment:manage`, otherwise as the deployment admin login.
- On a direct connection, user and role management (`deployment user`, `deployment role`) needs root. As the deployment admin login those commands are refused, and the CLI says that root needs `deployment:manage`. Through the proxy they work with `deployment:manage` alone.
- `--username` and `--password` sign in as another deployment user and also connect directly. Avoid `--password` on a command line, it lands in shell history.
- Turning data access on or off is `deployment settings data-access` (see `/laser-deploy`). Admitting an address is `/laser-access`.

## Wiring a client

Clients connect with the values from `connect show`. Build the snippet from the connection string and endpoints, and leave the password as a placeholder such as `${LASER_PASSWORD}` that the user fills from a secret store. The SDK's connect guide is <https://docs.laserdata.com/laser-sdk/connect>, and <https://docs.laserdata.com/agents> covers every path from an API key.

To run a Laser SDK program yourself without the secret ever reaching the chat, capture the connection string in the same command that runs the program, so nothing is printed:

```sh
LASER_CONNECTION_STRING="$(laser deployment connect show -d orders --reveal -o json | jq -r .connection_string)" python app.py
```

The string is the deployment admin login, already percent-encoded for `connect()`. A paid deployment also needs the caller's address admitted, `access-rule allow-my-ip -d orders --protocol tcp` (see `/laser-access`).

For an MCP client, `laser deployment connect mcp -d orders --client claude-code` prints the configuration for the organization, region and deployment data endpoints. It reads the key from `LD_API_KEY` and never contains it. Clients are `claude-code`, `claude-desktop`, `cursor`, `vscode` and `codex`.

## Hard rules

1. Never run `connect show --reveal` yourself where its output reaches the chat, with or without `--root`, and never print a full password or connection string. The capture pattern above, which prints nothing, is the one exception. If the user asks for the password again, have them run the command in their terminal.
2. Never capture or pass the root login to a program. Programs and agents use the deployment admin login.
3. Never rotate without the user's go-ahead, and name the login (`admin` or `root`) and what stops working before you ask.
4. Never write connection details to a file the user did not name, and never into shell history.
5. Don't pass `--silent` or `-o name`. The user wants the output.
6. Don't invent connection string formats. Use the string `connect show` returns.

## Common asks

- "how do I connect to orders": `connect show -d orders -o json`, then show the connection string with the password masked and list the endpoints with their ports.
- "give me the password": tell the user to run `laser deployment connect show -d orders --reveal` in their own terminal and store it in a secret manager.
- "I need the root password": check that the user needs user or role administration that the console cannot do, then have them run `laser deployment connect show -d orders --root --reveal` in their own terminal. Without `deployment:manage` the key gets no root.
- "the password leaked" or "rotate the password": ask which applications use it, confirm, then `connect credentials rotate admin -d orders`. For root, confirm and add `--yes`. Follow up with `monitor activity` until the rotation task has finished.
- "my client can't connect but the CLI can": the CLI goes through the data access proxy and the client goes direct. Check for an access rule covering the client's address with `/laser-access`, and that the client has the current password after a rotation.

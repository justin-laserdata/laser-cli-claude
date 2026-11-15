---
name: laser-onboard
description: >-
  First-run setup for the LaserData CLI. Installs the binary, signs in with an API key, picks the workspace and deployment scope, and sanity-checks the active context.
---

# laser-onboard

You are walking the user through their first `laser` setup. Be terse. Each step prints what it did, then waits before moving on.

## Step 1. Install the binary

If `command -v laser` already returns a path, skip the install and report that path with the output of `laser version`. Offer `laser update`, which replaces the binary in place with the latest release.

Otherwise:

```sh
curl -fsSL https://cli.laserdata.cloud/install.sh | sh
laser version
```

The installer checks the archive's SHA-256 checksum and puts `laser` in `$HOME/.local/bin`, or `$LD_PREFIX` when set. If `laser` is not on `PATH` afterwards, read the installer's hint back to the user. `sh -s -- --with-cc-skills` installs this skill pack in the same step.

## Step 2. Sign in

The CLI signs in with an API key created in the console. Ask the user to run this in their own terminal, not through you:

```sh
laser auth login
```

It prompts for the key with masked input, validates it, binds the organization the key belongs to, and stores the key in the OS keyring (Keychain on macOS, Secret Service on Linux). Only the key's name is written to disk. Never ask the user to paste a key into the chat and never put one on a command line.

- The organization is discovered from the authenticated key. Do not ask for an organization id. An explicit `--org` does not grant access outside the key's organization.
- A key scoped to one workspace also selects that workspace.
- Login saves the key in a context, `default` unless named: `laser auth login --context staging`. See `/laser-context`.
- On Linux without a Secret Service provider, `LD_ALLOW_INLINE_KEY=1` before login stores the key in the config file (mode 0600). Suggest it only when no keyring exists.

Agents and CI need no saved context. With `LD_API_KEY` exported, every command signs in for that one invocation, and `LD_WORKSPACE` and `LD_DEPLOYMENT` set the scope. If `LD_API_KEY` is already set where you run, `laser auth login` saves it without a prompt.

## Step 3. Confirm

```sh
laser auth status
laser org key whoami -o json
laser org show -o json
```

`auth status` shows the context, organization, key, workspace and deployment scope. `org key whoami` shows the key's role and permissions. `org show` returns the organization, its plan and its totals. A failure here means a wrong, expired or revoked key: hand off to `/laser-debug`.

## Step 4. Pick the scope

```sh
laser workspace list -o json
laser workspace use main
laser deployment list -o json
laser deployment use orders
```

`workspace use` and `deployment use` save the scope in the context. Without them the CLI falls back to the only workspace of the organization and the only deployment of the workspace. `laser deployment use --all` clears the deployment scope so workspace lists span every deployment again. `-w` and `-d` take an id or a name and override the scope for one command. The CLI never asks for an environment, it reads it from the organization structure.

`deployment list` merges every zone that serves the workspace. When a zone does not answer, stderr says `1 of 2 zones did not answer` and the JSON `total_results` is `null`. Report the list as incomplete, never as empty.

## Step 5. What's next

Tell the user they can now:

- `laser tui`: the console in the terminal. It is interactive, so the user runs it. It needs no login first: without a key it opens on a welcome screen that takes the key.
- `laser deployment --help`, and `--help` on any group or command: every command and flag.
- `/laser-workspace`: the workspace overview and activity.
- `/laser-deploy`: create a deployment from natural language, with a price estimate first.
- `/laser-troubleshoot`: diagnose a deployment.
- `/laser-streams`, `/laser-agents`, `/laser-query`: read the data inside a deployment.

## Exit codes

Every skill branches on these:

| Code | Meaning |
| --- | --- |
| 0 | Success |
| 1 | Invalid input, configuration or scope, or a destructive command run without `--yes` off a terminal |
| 2 | The API refused the request, or the command line did not parse |
| 3 | Network or file error |
| 4 | A remote operation failed, was refused, or did not finish in time |
| 70 | Internal error |
| 130 | Interrupted, or a typed confirmation did not match |

`laser docs` prints the API documentation addresses. The docs live at <https://docs.laserdata.com/cli>.

Stop. Don't keep going.

## Dashboard management

`laser tui` opens the dashboard with the same context and scope. Its Actions menus provide forms for supported management commands. Use the dashboard for guided fields, dynamic choices, typed confirmations, and progress while saving. Deployment metrics include charts, node selection, and heartbeat history. Enter opens a row as a data page. The headless CLI remains the interface for agent automation.

The dashboard exposes direct New, Edit, and Delete controls. `Shift+L` opens sign-out confirmation. `/` opens command completion, and list search is visible above the rows. The create form selects Managed or BYOC. Use the free quick-start action for a free deployment.

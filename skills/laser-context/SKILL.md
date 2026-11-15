---
name: laser-context
description: >-
  Manage named CLI contexts, each with its API key, organization, workspace and deployment scope. List, switch, create, rename, delete, sign out, and explain how the scope of a command is resolved.
---

# laser-context

A *context* holds a reference to an API key (the secret lives in the OS keyring), an organization, a workspace and a deployment scope. One machine can hold many contexts, one is active at a time. Use contexts to switch between organizations or keys without signing in again.

## Verbs

```sh
laser context list -o json
laser context current
laser context show -o json
laser context show prod -o json
laser context create prod --activate
laser auth login --context prod
laser context use prod
laser context rename prod prod-eu
laser context delete prod-eu
laser auth status
laser auth logout
laser auth logout --all
```

`ctx` is an alias for `context` and `context switch` an alias for `context use`. `context create` makes an empty context, and `auth login --context <name>` gives it a key and binds the key's organization. `context show` defaults to the active context and never prints the key.

One invocation can use another context with `--context prod` or `LD_CONTEXT=prod`, for example `laser deployment list --context prod`.

## Scope

Every command acts in one organization and one workspace. Data commands act on one deployment, or on every deployment where the console offers a workspace-wide list.

| Scope | Resolved from, in order |
| --- | --- |
| Organization | `--org`, `LD_ORG`, the context, authenticated key discovery |
| Workspace | `-w` or `--workspace` (id or name), `LD_WORKSPACE`, the context, the only workspace of the organization |
| Deployment | `-d` or `--deployment` (id or name), `LD_DEPLOYMENT`, the context, the only deployment of the workspace |

```sh
laser workspace use main
laser deployment use orders
laser deployment use --all
laser topic list -d payments
```

`workspace use` and `deployment use` save the scope in the active context. Switching the workspace clears a deployment scope outside it. `deployment use --all` clears the deployment scope. Flags and variables override the saved scope for one command only.

## Local files

- `$XDG_CONFIG_HOME/laser/config.toml`: the contexts with their scope, mode 0600. Only key names are stored here.
- `$XDG_STATE_HOME/laser/active`: the name of the active context.

macOS uses the platform equivalents. `--config <path>` points one command at another config file.

## Hard rules

1. Never print, ask for, or pass an API key. Keys go in through `laser auth login` in the user's own terminal, or through `LD_API_KEY` in CI.
2. `context delete` removes the context and its keyring entry. `auth logout` clears the key and keeps the context, `--all` does it for every context. Confirm with the user before either.
3. `auth purge` deletes every context, every key and the config file. It is destructive: without a terminal the CLI refuses it unless `--yes` is passed. Show what will go, ask the user to confirm, and only then run `laser auth purge --yes`.
4. `LD_ALLOW_INLINE_KEY=1` stores a key in the config file. Recommend it only when the machine has no OS keyring.

## Common asks

- "what context am I on": `laser context current`, then `laser auth status` for the organization, key, workspace and deployment.
- "switch to staging": `laser context use staging`. If it does not exist, `laser context create staging --activate`, then the user runs `laser auth login --context staging` in their terminal.
- "work on payments for a moment": pass `-d payments` on each command instead of changing the saved scope.
- "show every deployment again": `laser deployment use --all`.
- "which key is this": `laser org key whoami -o json` shows the key's name, role and permissions.

## Don't

- Don't edit `config.toml` by hand. Use the verbs.
- Don't suggest `auth purge` to fix one broken context. `auth logout` and a new `auth login` do that.

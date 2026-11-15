---
name: laser-workspace
description: >-
  The workspace overview, its activity log, and workspace and environment settings. Lists and switches workspaces, summarizes deployments, health, agent sessions and streams, reads what changed across deployments, finds resources by name, and creates, updates or deletes workspaces and environments.
---

# laser-workspace

An organization holds workspaces, a workspace holds environments, and an environment holds deployments. This skill covers the console's workspace switcher, Workspace overview, Activity and Workspace settings pages.

## Overview

```sh
laser workspace list -o json
laser workspace use main
laser workspace show -o json
laser workspace show -w analytics -o json
laser org structure
laser find checkout -o json
```

`workspace show` is the overview: deployments, health, agent sessions, streams and recent activity. `org structure` prints every workspace, environment and deployment as a tree. `find` looks for a workspace, deployment, stream or topic whose name contains the text, at least 2 characters, ignoring case. `ws` is an alias for `workspace`.

## Activity

```sh
laser activity --from 24h -o json
laser activity --env production --resource topic --action deleted -o json
laser activity --kind change --from 7d -o json
laser activity --kind gap --from 7d -o json
```

`activity` is the workspace's change log across its deployments: streams, topics, consumer groups, users, tokens, roles, consumer filters, projections, forks and schemas. Filters:

- `--env`: one environment, id or name.
- `--kind`: `change`, `gap` (a time range the log could not record) or `skipped` (one record that was not stored).
- `--resource`: `stream`, `topic`, `consumer_group`, `user`, `personal_access_token`, `managed_role`, `role_binding`, `filter`, `filter_binding`, `projection`, `fork` or `schema`.
- `--action`: `created`, `updated`, `deleted`, `partitions_changed`, `password_changed`, `permissions_updated`, `defined` or `bindings_updated`.
- `--from` and `--to`: RFC 3339 or an age such as `30m`, `24h` or `7d`. The plan's retention limits how far back it reads.

Activity of one deployment, including the operations LaserData ran on it, is `laser deployment monitor activity -d orders` (see `/laser-troubleshoot`). Who changed organization settings, members or keys is the audit log (see `/laser-audit`).

## Workspaces

```sh
laser workspace create --name analytics --description "Analytics team"
laser workspace update -w analytics --email data@example.com --protected true
laser workspace delete -w analytics --yes
laser workspace delete -w analytics --code 482913
```

`create` and `update` take `--name`, `--description`, `--email` and `--protected`. `delete` removes the workspace with every environment and deployment in it.

## Environments

```sh
laser env list -o json
laser env show production -o json
laser env create --name staging --description "Pre-release"
laser env update staging --protected true
laser env delete staging --yes
laser env delete staging --code 482913
```

Environments group the deployments of the workspace. `environment` and `envs` are aliases for `env`. `env delete` removes the environment with its deployments.

## Hard rules

1. Start read-only. `workspace show`, `activity` and `org structure` answer most questions.
2. Running from Claude Code there is no terminal, so `create` neither prompts nor shows a summary: pass every value as a flag, print the command, and wait for the user's approval before running it.
3. Deletes are destructive and cascade to every deployment underneath. Name what will go (run `laser org structure` and quote the subtree), ask the user to type the workspace or environment name to confirm, and only then run the delete with `--yes`. Without a terminal the CLI refuses a delete that lacks `--yes`.
4. A protected workspace or environment emails a confirmation code to the organization email when you delete it, and the command exits asking for it. Ask the user for the code and run the delete again with `--code`. Never guess a code.
5. Turning protection on is safe. Turning it off before a delete is the user's decision, never yours.

## Common asks

- "what's going on in main": `laser workspace show -w main -o json`, then summarize deployments by health, active agent sessions and the latest activity.
- "what changed yesterday": `laser activity --from 24h -o json`, grouped by deployment, resource and action.
- "who deleted the refunds topic": `laser activity --resource topic --action deleted --from 7d -o json` for the change, then `/laser-audit` if the user needs the member.
- "where is the checkout stream": `laser find checkout -o json`.
- "add a staging environment": `laser env create --name staging` after approval.

## Don't

- Don't treat a `gap` entry as "nothing happened". It marks a range the log could not record.
- Don't delete a workspace or environment to "clean up" without listing the deployments inside it first.
- Don't create workspaces or environments the user did not ask for.

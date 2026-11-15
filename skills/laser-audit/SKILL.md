---
name: laser-audit
description: >-
  Query the organization audit log with `org audit list`. Filters by time window, event type, workspace, environment, deployment, the member who acted, the member the event is about, the API key used, or correlation id. For incident forensics, compliance reviews and "who changed what, when".
---

# laser-audit

The audit log records who changed what in the organization, newest first: members, roles, keys, invitations, cloud accounts, workspaces, environments, deployments, access rules, networking, backups, configuration, billing, and health alerts.

## Verbs

```sh
laser org audit types -o json
laser org audit list --from 2026-11-10 -o json
laser org audit list --from 2026-11-01 --type member_updated -o json
laser org audit list --from 2026-11-10T00:00:00Z --to 2026-11-11T00:00:00Z --type deployment_deleted --type deployment_deleting -o json
laser org audit list -w main -d orders --from 2026-11-04 -o json
laser org audit list --env production --from 2026-11-04 -o json
laser org audit list --author 615380456123456790 --from 2026-11-04 -o json
laser org audit list --user 615380456123456791 -o json
laser org audit list --key 615380456123456821 --from 2026-11-04 -o json
laser org audit list --correlation-id 9123dc1a6dbe054b67a88a5129790dab -o json
```

- `--type` repeats. Take type names from `org audit types`, don't infer them.
- `--from` is inclusive and `--to` exclusive, each RFC 3339 or `YYYY-MM-DD` in UTC.
- `--author` is the member who acted, `--user` the member the event is about, `--key` the API key used. A member demoting another shows the first as author and the second as user.
- `-w` and `-d`, or `LD_WORKSPACE` and `LD_DEPLOYMENT`, narrow the log to one workspace or deployment. The scope saved in the context never narrows it, so the log is organization-wide by default. `--env` narrows to one environment.
- `--correlation-id` (32 hex characters) returns every event of one request.
- Pages hold 25 rows by default, at most 100 with `--results`. Walk `--page` until a page comes back empty.

Each row names the event, who acted (a member, an API key, or LaserData itself), the workspace, environment or deployment it touched, the correlation id and the time.

## Hard rules

1. Always bound the time window. Default to the last 24 hours unless the user asks for more.
2. Parse `-o json` for forensics. The table shortens the detail.
3. Cap a sweep at about 20 pages and say "more results, narrow the window" instead of paging on.
4. Rows can hold emails and destinations. Quote the event, the ids and the time. Redact personal data unless the user asked for it.
5. Reading the log needs `audit:read`.

## Common asks

- "who deleted orders yesterday": `org audit list -d orders --type deployment_deleting --type deployment_deleted --from <yesterday>` and the matching `--to`, then map the author to a member with `laser org member list -o json`.
- "what did this member do this week": `--author <member id>` with the week's window, grouped by event type.
- "trace this request": `--correlation-id <id>` from the error's `request_id` or an alert.
- "every configuration activation this quarter": `--type deployment_config_activated` over the quarter, paged until empty.
- "what did this API key do": `--key <key id>` with a window.

## Don't

- Don't use the audit log as telemetry. It records actions, not metrics. Use `/laser-troubleshoot` for runtime data.
- Don't confuse it with workspace activity, which records changes inside deployments such as topics and consumer groups (see `/laser-workspace`).

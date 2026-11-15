---
name: laser-channel
description: >-
  Notification channels for the organization or one workspace with `org notification`. Creates Slack, webhook and email channels, tests them, turns them on or off, subscribes them to event types scoped to workspaces, environments or deployments, and reads what each channel was sent.
---

# laser-channel

A *channel* is a destination for alerts: a Slack incoming webhook, a generic webhook, or an email address. A *subscription* decides which event types a channel receives, optionally only for some workspaces, environments or deployments. Channels belong to the organization, or with `--workspace-channels` to the selected workspace (the console's workspace settings).

## Channels

```sh
laser org notification channel list -o json
laser org notification channel list --workspace-channels -w main -o json
laser org notification channel show 615380456123456851 -o json
laser org notification channel create ops-slack --kind slack --destination "$SLACK_WEBHOOK_URL" --slack-channel "#alerts" --remarks "Production alerts"
laser org notification channel create ops-hook --kind webhook --destination https://hooks.example.com/laser --webhook-method POST --webhook-header "Authorization=Bearer $HOOK_TOKEN"
laser org notification channel create oncall-mail --kind email --destination oncall@example.com
laser org notification channel test 615380456123456851
laser org notification channel update 615380456123456851 --enabled false
laser org notification channel history 615380456123456851 --type replication_condition -o json
laser org notification channel delete 615380456123456851 --yes
```

`--kind` decides what `--destination` takes: a Slack or webhook https URL, or an email address. Slack channels also take `--slack-channel`, `--slack-username` and `--slack-icon-emoji`. Webhooks take `--webhook-method` (`POST`, `PUT` or `PATCH`) and repeated `--webhook-header NAME=VALUE`, which replace the stored headers. `update` changes only the flags you pass, and `--enabled false` pauses a channel without losing its settings. `history` lists notifications already sent, filtered with `--type`. Offer the Slack, webhook and email kinds.

## Subscriptions and event types

```sh
laser org notification types -o json
laser org notification subscription list 615380456123456851 -o json
laser org notification subscription add 615380456123456851 --type replication_condition --scope-deployment orders
laser org notification subscription add 615380456123456851 --type node_unreachable --type runtime_degraded --scope-workspace main
laser org notification subscription set 615380456123456851 615380456123456861 --type runtime_degraded --scope-environment production
laser org notification subscription delete 615380456123456851 615380456123456861 --yes
```

`types` lists the event types a subscription can select. Take type names from it, don't invent them. `subscription add` selects repeated `--type` values, optionally limited with repeated `--scope-workspace`, `--scope-environment` or `--scope-deployment` (ids or names). `set` replaces the types or the scope of one subscription and keeps what is left out. Every channel, subscription and type command takes `--workspace-channels` to work on the workspace's channels instead.

## Hard rules

1. Webhook and Slack URLs often carry a token. Read them from an environment variable the user set, and never echo a destination back into the chat after creation. `channel show` prints it, so run it only when the user asks.
2. Print `create` and `update --destination` and wait for approval. The user must see the final destination before it is saved.
3. After `create`, run `channel test` once and ask the user to confirm the message arrived before calling the channel working.
4. Deleting a channel deletes its subscriptions with it, and deleting a subscription stops those alerts. Show the channel or subscription, ask the user to confirm, and only then add `--yes`. Without a terminal the CLI refuses a delete that lacks `--yes`.
5. Reading channels needs `notifications:read`, changing them `notifications:manage`.

## Common asks

- "alert #alerts on Slack when orders breaks": `channel create` with `--kind slack`, `channel test`, then `subscription add` with the failure types from `types` and `--scope-deployment orders`.
- "what channels do we have": `channel list -o json`, render name, kind and enabled.
- "pause the old webhook": `channel update <id> --enabled false`.
- "what did this channel get this week": `channel history <id> -o json`.

## Don't

- Don't paste destinations or webhook headers into the chat.
- Don't subscribe a channel to every type when the user named a few.
- Don't bulk-delete channels without per-entry confirmation.

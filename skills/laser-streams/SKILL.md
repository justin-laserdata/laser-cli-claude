---
name: laser-streams
description: >-
  Streams, topics, consumer groups, messages and dead letters. Lists them across the workspace, reads one deployment live, creates, renames and deletes streams and topics, manages partitions and consumer groups, reads a partition without moving any consumer, sends test messages, and reads the dead letter queues.
---

# laser-streams

A deployment's streaming server holds streams, a stream holds topics, a topic holds partitions, and consumer groups share a topic's partitions. Commands take a stream and a topic by name or numeric id.

## Where the data comes from

- Workspace lists (`stream list`, `topic list`) read the metadata each zone keeps in sync, merged across every zone that serves the workspace, narrowed to the deployment scope when one is set. They need no connection to the deployments.
- `--live` and every other command read one deployment live through the LaserData data access proxy, signed in with the API key. `--direct` skips the proxy and needs an access rule for the caller's address (see `/laser-credentials`).
- When a zone does not answer, the list still prints, stderr says how many zones were missing, and `total_results` is `null` in the JSON. Report the list as incomplete.

## Streams

```sh
laser stream list -o json
laser stream list --agentic -o json
laser stream list --env production --name check -o json
laser stream list --live -d orders -o json
laser stream show checkout -d orders -o json
laser stream create checkout -d orders
laser stream update checkout -d orders --name checkout-v2
laser stream purge checkout-v2 -d orders --yes
laser stream delete checkout-v2 -d orders --yes
```

`stream list` filters with `--env`, `--name`, `--projected`, `--filtered` and `--agentic`, and `--system` adds the LaserData system streams. `show`, `create`, `update` and `delete` act on one deployment. The server assigns a new stream's numeric id. `update` renames it, and producers and consumers that use the numeric id are unaffected. `delete` removes the stream with its topics and messages.

## Topics

```sh
laser topic list -o json
laser topic list --stream checkout -d orders -o json
laser topic list --live --stream checkout -d orders -o json
laser topic show checkout events -d orders -o json
laser topic create checkout events -d orders --partitions 6 --expiry "7 days" --max-size "10 GiB" --replication-factor 3
laser topic update checkout events -d orders --expiry never
laser topic partitions checkout events -d orders -o json
laser topic producers checkout events -d orders -o json
laser topic partitions checkout events -d orders --add 2
laser topic partitions checkout events -d orders --remove 1 --yes
laser topic purge checkout events -d orders --yes
laser topic delete checkout refunds -d orders --yes
```

`create` takes `--partitions` (1 by default, one partition keeps a single-writer order), `--compression` (`none` or `gzip`), `--expiry` such as `7 days` or `never`, `--max-size` such as `10 GiB` or `unlimited`, and `--replication-factor` from 1 to 32, at most the node count and fixed at create. `update` changes `--name`, `--expiry` or `--max-size`. `partitions --add` appends partitions, and hash-keyed producers route differently afterwards. `--remove` deletes the highest partitions with their messages and offsets.

## Consumer groups

```sh
laser group list checkout events -d orders -o json
laser group show checkout events billing -d orders -o json
laser group create checkout events billing -d orders
laser group policy checkout events billing -d orders -o json
laser group delete checkout events billing -d orders --yes
```

Consumers in a group share the topic's partitions and join by the group's name. `show` lists the members. `policy` shows the consumer filter the streaming server applies to every read of the group (see `/laser-filters`). `delete` removes the group with its stored offsets.

## Messages

```sh
laser message read checkout events -d orders --partition 0 --kind first --count 50 -o json
laser message read checkout events -d orders --kind offset --value 1200 --payload json -o json
laser message read checkout events -d orders --kind timestamp --value 1762848000000000 -o json
laser message send checkout events -d orders --message '{"order_id":42,"status":"paid"}' --key order-42 --header source=cli
laser message send checkout events -d orders --file order.json --partition 2
```

`read` reads one batch of one partition and stores no consumer offset, so it never moves a consumer. `--kind` is `first`, `last` (the default), `offset`, `timestamp` with `--value` in microseconds, or `next` after the stored offset of consumer 1. `--count` is 10, 20, 50 or 100. `--payload` prints as `text`, `json` or `base64`.

`send` sends repeated `--message` texts or `--file` payloads through the data access proxy and needs the deployment manage permission. The server balances the batch unless `--partition` or `--key` is set, and messages with the same key reach the same partition. `--header KEY=VALUE` adds a string header. A send is never resent, so a timeout means "check before sending again".

## Dead letters

```sh
laser dlq list -d orders -o json
laser dlq list -d orders --source agent --stream support --count 20 -o json
laser dlq list -d orders --source managed -o json
```

`dlq list` reads the deployment's dead letters, newest first, and is read only. `--source managed` reads failures of the managed data features: changes that failed to apply, projection batches that could not be processed, and records dropped by schema decoding. `--source agent` reads agent messages that exhausted their retries or were rejected. Both are read by default.

## Hard rules

1. Start read-only. Lists, `show`, `message read` and `dlq list` change nothing.
2. Stream delete, topic delete, `partitions --remove` and group delete destroy messages or offsets. Show what goes (the message and partition counts from `show`), ask the user to type the name, and only then run the command with `--yes`. Without a terminal the CLI refuses them unless `--yes` is passed.
3. `stream purge` and `topic purge` remove messages while preserving the resource. Read the target, explain the data loss, and obtain confirmation before using `--yes`. Never substitute delete and recreate for purge.
4. `message send` writes real data that consumers will process. Confirm the stream, topic and payload with the user first, and never send to a production topic "to test".
5. Adding partitions changes key routing for hash-keyed producers. Say so before `--add`.
6. Never touch the LaserData system streams that `--system` reveals.

## Common asks

- "what streams do we have": `stream list -o json`, grouped by deployment.
- "peek at the latest orders": `message read checkout events -d orders --kind last --count 20 --payload json -o json`.
- "who consumes events": `group list checkout events -d orders -o json`, then `group show` for the members of each group.
- "why did messages disappear": `dlq list -d orders -o json`, and `laser activity --resource topic --from 24h -o json` for deletes and partition changes.

## Don't

- Don't use `message read --kind next` expecting it to advance a consumer. It never stores an offset.
- Don't loop `message read` to tail a topic. Each call is one batch.
- Don't create streams or topics the user did not ask for.

---
name: laser-agents
description: >-
  Agents and what they did. Lists agents, agentic streams and live agent connections, finds agent sessions and reads their timeline, state, links, sources and change feed, threads conversations, inspects and forgets agent memory, and lists, walks and queries knowledge graphs.
---

# laser-agents

Agents publish to *agentic streams*, the streams with an `agent.sessions` topic. Each unit of agent work is an *agent session* with a 26 character id, sessions can nest under a root session, and conversations group sessions by `gen_ai.conversation.id`. Agents also write memory and knowledge graphs on the deployment.

Workspace lists (`agent list`, `agent streams`, `session list`) read synced metadata merged across zones and need no connection to the deployment. Everything marked "live" reads one deployment through the data access proxy, or directly with `--direct`.

## Agents

```sh
laser agent list -o json
laser agent list --connected true --search planner -o json
laser agent list --quarantined true -o json
laser agent streams -o json
laser agent show planner-01 -d orders -o json
laser agent connections -d orders -o json
```

`agent list` has one row per agent card on an agentic stream, or per agent seen only on a live connection. Filter with `--env`, `--search` (3 or more characters), `--connected`, `--quarantined` and `--stream`. `streams` lists the agentic streams with their session counts. `show` returns one agent of the deployment with its cards on every stream and its live connections. `connections` lists live agent connections and the presence each advertises, `--limit` at a time, continued with `--after <client id>`.

## Sessions

```sh
laser session list --status active -o json
laser session list --agent planner-01 --label refund -o json
laser session list --status failed --stream 1 -d orders -o json
laser session list --live --stream support --text refund -d orders -o json
laser session show 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 --stream 1 -d orders -o json
laser session show 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 --live --stream support -d orders -o json
laser session events 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 --stream support -d orders -o json
laser session events 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 --stream support --kind context.assembled -d orders -o json
laser session state 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 --stream support -d orders -o json
laser session links 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 --stream support --surface memory -d orders -o json
laser session sources 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 --stream support -d orders -o json
laser session changes --stream support -d orders -o json
```

- `session list` reads synced metadata, newest activity first. Filter with `--status` (`submitted`, `active`, `paused`, `completed`, `failed` or `canceled`), `--agent`, `--root` for one session tree, `--label` and `--text`. `--live` reads one stream's sessions from the deployment instead.
- Synced reads take the stream's numeric id in `--stream`, live reads take its name. `laser stream list --agentic -o json` lists the agentic streams with both.
- `show` returns the status, counters, budget and hierarchy. `events` (the timeline), `state` (the state document and its history), `links` (what the session wrote, recalled or touched, `--surface` one of `memory`, `kv`, `graph_node`, `graph_edge`, `projection` or `child`), `sources` (the partitions holding its records) and `changes` (the stream's session change feed) read live.
- `--as-of` and `--series` pin the walk bound of a paged read so later pages keep the same rows. The CLI reads the bound itself when they are left out.

## Conversations

```sh
laser conversation list --stream support --count 500 -d orders -o json
laser conversation list --stream support --agent planner-01 -d orders -o json
laser conversation show 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 --stream support --payload json -d orders -o json
```

Conversations are threaded from a window of the stream's newest messages, `--count` per partition (200 by default, at most 1000) of the `agent.*` topics, not from the session index. Without `--stream`, the CLI chooses `laser` when it exists, or the only readable stream. Otherwise, select a stream explicitly. An older conversation may fall outside the window: raise `--count` before concluding it does not exist.

## Memory

```sh
laser memory namespaces -d orders -o json
laser memory list support-notes -d orders -o json
laser memory list support-notes --conversation 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 -d orders -o json
laser memory show support-notes 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 -d orders -o json
laser memory forget support-notes 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 -d orders --yes
```

`memory list` shows what agents remembered in one namespace, newest first. `forget` deletes one stored memory, which an agent can remember again. On a deployment that scopes namespaces by stream, pass `--stream`.

## Graphs

```sh
laser graph list -d orders -o json
laser graph show knowledge.v1 -d orders -o json
laser graph neighbors knowledge 01JC8ZK3S2W5Q6R7T8V9X0Y1Z2 --dir both --depth 2 -d orders -o json
laser graph nodes knowledge --limit 200 -d orders -o json
laser graph query knowledge --file who-knows.json -d orders -o json
laser graph register --file knowledge.json -d orders
laser graph drop knowledge.v1 -d orders --yes
```

`neighbors` walks the edges from one node: `--dir` (`out`, `in` or `both`), `--edge-type`, `--depth` up to 8, `--limit`, `--as-of` in Unix microseconds, and `--conversation`. `query` runs a graph query from a JSON file with `start`, `traverse`, `return_` and `limit`. `register` reads a graph projection from a JSON definition, and the deployment applies it in the background. `drop` keeps the nodes and edges and stops listing the graph.

## Hard rules

1. Everything here except `memory forget`, `graph register` and `graph drop` is read-only. Prefer it.
2. `memory forget` and `graph drop` are destructive: without a terminal the CLI refuses them unless `--yes` is passed. Show the memory or the graph, ask the user to confirm, and only then add `--yes`.
3. Session state, memory and conversation payloads can hold user data. Summarize them, quote only what the user asks for, and never paste whole payloads into the chat unasked.
4. Don't write a graph definition or query from memory. Start from `graph show` of an existing graph.

## Common asks

- "what are my agents doing": `session list --status active -o json`, grouped by agent.
- "why did session X fail": `session show` for status and counters, then `session events` for the timeline up to the failure, then `dlq list --source agent` (see `/laser-streams`).
- "what does the planner remember about this conversation": `memory list <namespace> --conversation <id>`.
- "forget that": confirm the memory with `memory show`, then `memory forget` with `--yes` after approval.
- "who is connected right now": `agent connections -d orders -o json`.

## Don't

- Don't mix synced and live stream identifiers: numeric id for synced reads, name for live reads.
- Don't treat a missing conversation as gone before widening `--count`.
- Don't forget memories in bulk. One at a time, each confirmed.

Conversation commands no longer assume an absent `laser` stream. Pass `--stream` explicitly when several readable streams exist. Memory lists accept `--prefix`, `--start`, `--end`, `--key-contains`, and `--conversation`. Preserve the stream and deployment when following graph references from sessions.

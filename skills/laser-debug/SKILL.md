---
name: laser-debug
description: >-
  Explain why a laser command failed. Reads the error and exit code, the local debug log, and the API's problem details, groups recent failures, and maps each to a next step. Documents the platform headers (`ld-request`, `idempotency-key`, `idempotent-replayed`, `link`, `retry-after`) and the `application/problem+json` envelope (RFC 7807).
---

# laser-debug

Start from what the failing command printed, then the debug log.

## Read the error

A failure prints `error: <message>` on stderr, then `note:` lines for API errors (`title`, `request_id`, `type`, `retryable`, and one `field issue` per invalid field) and `help:` lines with a suggestion. The exit code says which kind of failure it was:

| Code | Meaning | Next step |
| --- | --- | --- |
| 1 | Invalid input, configuration or scope | Fix the flag or scope the message names. A destructive command without `--yes` off a terminal also exits 1. |
| 2 | The API refused the request, or the command line did not parse | Read the `note:` lines. For a parse error, run the command's `--help`. |
| 3 | Network or file error | Check the network and the file path. |
| 4 | A remote operation failed, was refused, or did not finish in time | Read the deployment's activity and tasks (`/laser-troubleshoot`). |
| 70 | Internal error | Collect the log lines and the version (`laser version`). |
| 130 | Interrupted, or a typed confirmation did not match | Nothing ran. |

## Read the debug log

The CLI writes a daily debug log for every run, including `--silent` runs and dashboard sessions:

- Linux: `$XDG_DATA_HOME/laser/logs/laser-YYYY-MM-DD.log`, which is `~/.local/share/laser/logs/` by default.
- macOS: `~/Library/Application Support/laser/logs/`.

1. Open today's file, or the newest one in the directory.
2. Read the last 200 lines. Each line has a time, a level, a target and a message.
3. Filter `WARN` and `ERROR` first.
4. Group by target (the module path) so the user sees where it failed.
5. For an HTTP failure, quote the method, the address and the status. For an API error, quote `code`, `reason` and the request id.

To capture more, have the user rerun the failing command with `LD_LOG_FILE=trace` in front of it. `LD_LOG` sets the stderr level the same way, and `--debug` prints debug lines on stderr for one run. Both take `tracing` directives.

## Map the failure

- `401`: the key is wrong, expired or revoked. Confirm with `laser org key whoami -o json`, then the user signs in again with `laser auth login` in their own terminal.
- `403`: the key's role lacks a grant. `laser org key whoami -o json` shows the resolved permissions next to what the command needs. Some actions need a browser session and refuse every API key: creating or deleting an organization, account settings, adding a card, linking a marketplace purchase, and Allow my IP.
- `403` with "a capability is required for this command" on a session, memory, key-value, graph or schema command: the deployment gives this answer for a missing grant and also for a stream name it does not know. Check the stream first. The CLI resolves a numeric stream ID to its name, so a wrong name is the usual cause. Then read the grants with `laser deployment whoami -d <deployment> -o json`.
- A workspace or deployment that is not found: check the scope with `laser auth status` and the name with `laser find orders -o json`.
- `409` with an idempotency code: a retry reused an `--idempotency-key` with a different body, or while the first request was still running. Retry the same body shortly, or use a new key.
- `429`: rate limited. A zone limits how fast one key sends requests, and the CLI already waits and resends. Quote `retry-after` when present and back off.
- `no active context`: the user signs in, or sets `LD_API_KEY` for automation (see `/laser-onboard`).
- `destructive operation requires --yes when not running on a terminal`: confirm with the user, then rerun with `--yes`.
- `N of M zones did not answer`: a workspace list is incomplete. Retry later, and name the missing zones from the JSON `zones` array.
- Transport errors or timeouts: check the network. Reads retry on their own. Creates and updates retry these failures only with `--idempotency-key`. Requests refused by the API key rate limit retry too, including writes. Check whether the resource changed before you run a failed write again.
- Data commands failing while the deployment is healthy: data access may be off or the proxy may refuse. See `/laser-credentials`.

## Platform headers

| Header | Direction | Meaning |
| --- | --- | --- |
| `ld-request` | response | The request id, 32 hex characters. The CLI prints it as `request_id`. Quote it when contacting support. |
| `idempotency-key` | request | Sent on creates and updates when `--idempotency-key` or `LD_IDEMPOTENCY_KEY` is set. The server keeps the first answer for 10 minutes per key and replays it for the same body. |
| `idempotent-replayed` | response | `true` when the answer came from the idempotency cache instead of running again. |
| `link` | response | RFC 8288 pagination links on paged lists. The CLI pages with `--page` and `--results`. |
| `retry-after` | response | Seconds to wait, sent with `429` and transient `5xx`. |

## Error envelope

API errors are `application/problem+json` (RFC 7807):

```json
{
  "type": "about:blank",
  "title": "Invalid Email",
  "code": "invalid_email",
  "reason": "Invalid email address",
  "instance": "8f4a2b6c9d1e4f3a8b5c7d9e0f1a2b3c",
  "field_issues": [{ "code": "invalid_email", "reason": "malformed address", "path": "email" }],
  "status": 400,
  "retryable": false
}
```

Branch on `code`, not on `title`. `field_issues` appears on validation errors. `retryable` is true for transient failures. Lead a summary with `code`, `reason` and the request id.

## Don't

- Don't paste keys, tokens or passwords into the chat, even from a log line.
- Don't leave a `tail -f` running. One read is enough.
- Don't suggest setting `LD_LOG_FILE=trace` permanently in a shell profile.

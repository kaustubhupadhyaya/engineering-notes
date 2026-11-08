---
title: OpenCode shared server in Herdr, database growth, and an upstream SSE connection leak
period: 2026-09-08..2026-09-11
status: architecture fix verified; upstream leak diagnosed, not fixable locally
applies_to: OpenCode CLI on Windows, Herdr terminal multiplexer
---

# OpenCode shared server in Herdr, database growth, and an upstream SSE connection leak

**Trigger.** Four debugging sessions over four days. Each started with the same report: a Herdr workspace group of `opencode` panes, six growing to seven and then eight, was slow or laggy again.

**Status.** The shared-server architecture held across all four rounds. Database-growth mitigations held, but the one-time prune was never actually run with `-Apply`. A persistent recurrence was traced to a confirmed upstream OpenCode SSE connection leak. No restart automation was built, deliberately.

## The original problem

Six panes each ran a full `opencode.exe` server:

| Pane working set |
|---|
| 406 MB |
| 908 MB |
| 84 MB |
| 184 MB |
| 285 MB |
| 593 MB |

About 2.55 GB total. Each server also ran a recursive file watcher over the entire user profile. The shared SQLite store had six-way WAL contention and had never been pruned: 131 MB, 29 sessions, 14,783 events.

## Fix: one server, N attach clients

A PATH-shadowing shim (`opencode.ps1`, `opencode.cmd`, and a bash twin) sits ahead of the npm-installed binary and routes bare `opencode` launches:

```
bare `opencode` -> known subcommand?
  yes -> exec the real opencode.exe with arguments unchanged
  no  -> ensure server on 127.0.0.1:4599 -> resolve --dir -> exec `opencode attach`
```

The server runs as `opencode serve --hostname 127.0.0.1 --port 4599`, rooted in an empty scratch directory so its own watcher has nothing broad to scan. A Task Scheduler logon trigger starts it directly, without a shell, so the first pane does not pay startup cost.

The shim also substitutes a per-pane scratch directory (`<data-dir>/panes/<PID>/`) whenever a session would otherwise root its watcher at the whole user profile.

`opencode.json` was hardened:

```json
{
  "snapshot": false,
  "watcher": {
    "ignore": [
      "**/node_modules/**",
      "**/.git/**",
      "**/AppData/**",
      "**/OneDrive/**",
      "**/My Drive/**",
      "**/dist/**",
      "**/build/**",
      "**/.venv/**",
      "**/__pycache__/**"
    ]
  }
}
```

A todo-sync plugin's `todo.updated` handler was changed from a synchronous `spawnSync(node ...)` to a detached fire-and-forget call. On a shared server, the synchronous version froze the entire event loop on every todo write, which stalled every attached pane.

Skill directories were de-duplicated: 13 of 14 directories under the OpenCode skills tree were byte-identical copies of entries in the Claude and agents skill trees. Removing them eliminated about 20 duplicate-skill-name warnings on every server boot and cut bootstrap `init` work.

## The leak

After hours of uptime, a new HTTP request to the server hung while already-attached panes kept working:

```text
curl -v http://127.0.0.1:4599/session
Operation timed out after 5010 milliseconds with 0 bytes received
```

Same result for `/`, `/config`, and `/session`.

`netstat -ano` on port 4599:

| State | Count |
|---|---|
| ESTABLISHED | 336 |
| CLOSE_WAIT | 20 |
| FIN_WAIT_2 | 20 |

Connections were spread 19 to 27 per client across all eight attached panes, so no single pane was the runaway. The server process RSS grew from 172 MB at start to roughly 700 to 843 MB after 3h08m to 3h36m of uptime. The server had written zero log lines in over three hours.

Root cause matched four open upstream issues:

- anomalyco/opencode #17628
- anomalyco/opencode #22198
- anomalyco/opencode #15645
- anomalyco/opencode #15149

Each independently describes the same symptom set: SSE connections leak on client reconnect, memory grows, and TCP connections accumulate.

## The wrong diagnosis that was caught

The first read was "the server is completely frozen, nothing is progressing." A restart was prepared. Before it ran, a direct SQLite query settled it:

```sql
SELECT max(time_updated) FROM message WHERE session_id = ...;
```

Timestamps came back 0 to 87 seconds old across six or more sessions at the exact moment HTTP endpoints were timing out. Messages were landing every few seconds throughout the window. The server was doing useful work; only new HTTP connections were stuck. The restart was withheld and live multi-subagent research continued.

This matters because HTTP reachability was actively misleading for this failure mode. The leak hits new connections first while existing streams and direct database writes keep working.

## Health check

A bare TCP port-open test sails straight through the leak. A real HTTP GET with a 2.5 second timeout was added, returning three states:

- **UP**: HTTP responds.
- **DEGRADED**: port accepts, HTTP times out.
- **DOWN**: port closed.

The first probe target was `/config`. That turned out to be the slowest endpoint even at full health and produced false DEGRADED readings. `/session` is lighter and closer to what an `attach` client actually needs.

The check is report-only. Two auto-restart designs were considered and rejected:

1. Timer-based restart after N hours.
2. Restart when the HTTP health check fails.

Both would have killed the exact productive-but-HTTP-degraded state described above. A monitor that measures liveness would have declared a working swarm dead.

## Timeline

| Round | Date | Finding | Action |
|---|---|---|---|
| 1 | 2026-09-08 | Six independent servers, ~2.55 GB RAM, each watching the whole profile; 131 MB unpruned database; 0.4 GB free RAM, 90.4% commit | PATH shim to one server plus attach clients; logon task; Herdr poll interval 100 ms to 500 ms; prune script built, dry-run only |
| 2 | 2026-09-10 | Round 1 held. Database still unpruned at 153 MB. Sync `spawnSync` in the todo plugin froze all panes on every write. 47 skill directories across three trees, about 20 duplicate warnings per boot. Missing `snapshot:false` and `watcher.ignore` | Config hardened; plugin made async; 13 duplicate skill dirs removed; shim redirects a home-rooted `--dir` to a per-pane scratch dir |
| 3 | 2026-09-11 early | Database grew to 199 MB despite the config fixes, because they stop new causes but do not shrink existing data. Pane count kept climbing 6, 7, 8. A burst of concurrent research subagents coincided with idle CPU rising from 1% to 4.7% | Probed server: all endpoints timed out; 336 established plus 40 zombie connections; zero log lines in 3 hours |
| 3 to 4 correction | 2026-09-11, about 20 minutes later | The "frozen" read was wrong. `max(message.time_updated)` showed messages every few seconds throughout | Restart withheld; root-caused to upstream SSE leak issues; both watchdog designs rejected |
| 4 | 2026-09-11 | Same untouched server partially self-recovered: established 336 to 32, CLOSE_WAIT 20 to 0, two of three endpoints back to HTTP 200; only `/config` still hung | Incident record written; health check rebuilt as report-only three-state on `/session`; server left running to a natural boundary |

Database growth across rounds: 131 MB (29 sessions, 14,783 events), 153 MB (31 sessions, 19,039 events), 199 MB (58 sessions, 29,588 events, 67.4 MB of `part` blobs).

## What generalizes

- A shared-server architecture trades N independent servers' memory for one failure domain. One upstream leak can now degrade every pane at once. Name that tradeoff, do not sell it as a strict improvement.
- "The server is not answering HTTP" and "the server has stopped doing useful work" are different claims. The only proof that mattered was a direct database query for forward progress.
- When corrected, re-verify with evidence. The fix was not an apology or a restatement; it was running the one query that could settle the question.
- Health-check target choice matters. Pick the endpoint an actual client uses, not the most obvious one.
- Config fixes do not undo accumulated debt. A prune that requires a manual `-Apply` step is a weaker fix than an automatic one, and this one stayed undone.
- Not every detected problem should trigger automation. A watchdog that cannot distinguish "degraded HTTP" from "productive work in progress" is a liability.

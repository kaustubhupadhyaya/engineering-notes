---
title: Claude Code Plan mode's auto-classifier gap, root-caused and fixed two ways
period: 2026-09
status: resolved on both machines, two different fixes
applies_to: Claude Code VS Code extension 2.1.272 on Windows
---

# Claude Code Plan mode's auto-classifier gap, root-caused and fixed two ways

**Symptom.** In Plan mode, every non-read-only Bash or PowerShell call prompted for permission instead of going through the auto-mode classifier. A session that had worked the previous night prompted on every call the next morning. Sessions running concurrently in other tabs hit the same behavior.

**Status.** Fixed on two machines with two different approaches, both valid for their constraints. The fix on the machine investigated here was verified live in production. The other machine's fix was verified mechanically but not behaviorally retested against a live Plan-mode session.

## Root cause

Plan mode only routes shell commands through the classifier when the session's classifier-arming latch survived the transition into Plan. Entering Plan from Auto keeps the latch armed. A direct switch into Plan did not.

A recent CLI update added a new interactive-terminal requirement to an implicit quiet path that some Plan-mode sessions had been relying on. Every VS Code session is non-interactive by construction (`isInteractive()` is set once at startup from `!process.stdout.isTTY`, `-p`, or `--sdk-url*`), so the new gate closed a path that had been covering a separate, older classifier-arming gap.

The regression was not visible by reading the current build alone. The current build made one classifier-arming code path look like the cause because it was real and easy to find. Only diffing the installed CLI against the previous build surfaced the actual change: the new interactive-terminal condition on a different path.

Two earlier explanations were formed and disproved on live evidence:

1. A hardcoded `CLAUDE_CODE_ENABLE_TASKS=0`. Real, but it governed a different subsystem.
2. A code-accurate story about when the auto-classifier gets armed. Correct about the code, but it described a pre-existing condition, not what changed. The tell was that the user kept reporting the same symptom after the explanation had been delivered.

## Two valid fixes

| | Extension patch | Settings change |
|---|---|---|
| **Approach** | Make every switch into Plan mode hop through Auto mode first | Set the session's starting permission mode to Auto |
| **What changes** | The extension's `set_permission_mode` handler | One VS Code user setting, `claudeCode.initialPermissionMode` |
| **Side effects** | None; the account's configured default mode is untouched | Every non-Plan session on that account also starts in Auto, not full-auto or no-prompts |
| **Survives extension update** | No; re-applied by the update task | Yes; it is a user setting, outside the versioned bundle |
| **Used here because** | The default starting mode on this machine had to remain exactly as configured | The other machine had no such constraint |

The settings fix does a surgical single-value text replace rather than a full JSON parse and reserialize, to avoid stripping comments the settings file may carry. It is idempotent and fails safe if the key already holds a different explicit value.

The extension fix inserts an Auto hop before applying Plan:

```js
try{/*claude-plan-auto-patch*/if(Q==="plan")try{await X.query.setPermissionMode("auto")}catch(__cpa){this.logger.warn(`[claude-plan-auto-patch] auto hop before plan refused: ${__cpa}`)}if(await X.query.setPermissionMode(Q),J)this.persistDefaultPermissionMode(Q);
```

Identifiers are re-minified every build and captured by the regex, not hardcoded. The patch script anchors on the handler's own `Refusing set_permission_mode on channel ...` warning string and the exact `query.setPermissionMode(...)` call site. It takes a timestamped backup, requires exactly one match, and exits 1 without writing if the handler's shape changes. A `-TargetJs` option lets a new regex run against a scratch copy and be byte-compared with a known-good hand patch before touching the live file.

The fix was added as a third patch in the existing update skill, alongside the no-split and fork-title patches. One run re-applies all three and prints one summary. The shared integrity check still applies: one contiguous changed span, under 4 KB, strict UTF-8, otherwise restore.

## The headless harness

Source reading could not settle whether a call was classified or prompted. A standalone Node harness drives the installed `claude.exe` over its own `stream-json` control protocol, the same protocol the VS Code extension uses. It sends a scripted mode-switch sequence, asks for one command that needs approval, and reports whether the request was resolved by the classifier or produced a prompt.

That gave a definitive PROMPTS versus CLASSIFIER verdict per run in about a minute. It also caught that a proposed binary patch did not change runtime behavior.

## The binary patch that looked right and did nothing

A same-length text edit removed the extra interactive-terminal condition from a scratch copy of `claude.exe`. The diff against an unpatched copy was exactly one byte. Driven through the harness, the patched copy still logged the unarmed-entry line and still prompted.

Two reasons not to do this:

1. The executable embeds precompiled Bun bytecode (`// @bun @bytecode` module headers, 1,720 of them). Editing the source text is silently ignored at runtime.
2. It is an Authenticode-signed vendor binary. Modifying it breaks the signature.

The binary route was rejected. The extension patch was the supported surface.

## Shared-repo reconciliation

This work lived in a repo synced across two machines. While the deeper investigation here was underway, a session on the other machine independently applied the simpler settings fix and pushed it. This machine's push was rejected as non-fast-forward.

The response was not to force-push. `git fetch`, then `git log HEAD..origin/master` and `git show origin/master:<path>` read the other side's actual content. It documented a different, independently valid fix, so the two accounts were merged into one record. A rejected push is a signal to read before acting.

## Evidence

- Harness driving the real installed CLI: switching directly into Plan logged an "entering Plan without the classifier armed" line and prompted for a PowerShell call. Switching into Plan through Auto first logged no such line, produced `classifier_request_started` and `classifier_request_finished` for the same kind of command, and the command ran with no prompt.
- Scratch binary diff: exactly one differing byte; behavior unchanged under the harness.
- `patch-plan-auto.ps1` validated against a backup copy of `extension.js`, byte-identical to a hand patch, then run against the live 2.1.272 install: marker present, exit 0.
- Live production check on the affected session, after the patch went active: zero unarmed-entry log lines and 10 of 10 `classifier_request_started`/`classifier_request_finished` pairs for consecutive Bash calls, with zero permission-prompt hook lines and zero manual approvals. Before the patch, that same session prompted on 100% of non-read-only calls.
- The initial log check produced a false contamination result because timestamps were compared as strings: `"23:06" > "04:39"` lexically even though 04:39 is later across midnight. Re-checking by line number instead of string-comparing timestamps caught it before it was reported.
- A scoped `PermissionRequest` hook relayed one session's pending approvals to a human-reviewed queue and decision file pair while the patch existed on disk but the running extension host had not reloaded it. It was limited to one session ID, had a 540-second wait and a hard wall-clock expiry, and was removed once live verification passed.
- The other machine's settings change: `claudeCode.initialPermissionMode` moved to `auto`, diffed against its backup with exactly one line changed; a repeat run reported a no-op. That run also surfaced that the no-split patch had been failing safe there for 10 days across three extension versions.
- Anthropic's docs were rewritten between 2026-09-11 and 2026-09-16 to describe the new behavior as intended, per a Wayback Machine diff. An independently reported identical regression on another Claude Code surface remained open with no maintainer reply.
- A five-agent verification workflow independently confirmed the interactive-terminal mechanism from the source diff, traced the non-interactive flag to its single call site, and cross-checked the behavior against the docs and the independent report.

## What generalizes

- A well-evidenced root cause can still be wrong. If it explains a condition that existed in the last known-good version, it cannot be what changed.
- Diff against the last known-good version, not just the current one.
- Real end-to-end testing settles arguments source reading cannot. The harness was cheaper than another round of source analysis and caught the dead binary patch.
- The fix has to satisfy the user's actual constraint, not the easiest one to implement. Two different correct answers to the same bug is not a contradiction.
- A rejected push is a signal to read, not an obstacle to route around.
- A scoped, expiring bridge is legitimate. An open-ended one becomes invisible standing behavior.
- Do not trust a zero-match result from string comparisons across a day boundary or midnight rollover. Compare by line number or parsed time.

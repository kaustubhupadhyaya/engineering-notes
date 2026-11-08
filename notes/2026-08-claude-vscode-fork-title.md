---
title: Patching the Claude Code VS Code "fork conversation loses all history" bug
period: 2026-08..2026-09
status: resolved, patch maintained across extension updates
applies_to: Claude Code VS Code extension 2.1.246 through 2.1.261 on Windows
---

# Patching the Claude Code VS Code "fork conversation loses all history" bug

**Symptom.** "Fork conversation from here" opened a new, empty tab. The forked conversation had no history, and only the newly typed prompt survived.

**Status.** Root cause found and patched. Reproduced three times before any change was made. The fix survived seven extension versions in nine days (2.1.246 through 2.1.261), including one fully unattended update.

## Root cause

`forkSession()` copies the parent's user, assistant, and attachment lines into a new `<id>.jsonl` file. It does not copy any title record: no `ai-title`, no `custom-title`, no `lastPrompt`, no `summary`.

The session-list scanner reads only the first and last 64 KB of each session file (`wQ=65536`) and looks for a title in four places, in order:

1. `customTitle`
2. `aiTitle`
3. `lastPrompt` from the tail
4. `summary` from the tail

If all four fail, it tries to parse the first user message in the head window. A session with no title from any branch is silently dropped from `listSessions()`.

`activateSessionFromServer()` resolves a `new_conversation_tab` target by looking for it in that list. On a miss it sends `update_panel_host_session {kind:"restore_declined"}` and creates a fresh empty session.

The trigger condition was specific: the parent conversation's opening message was a pasted image whose JSON line measured 280,617 bytes, over four times the 64 KB head window. The fork had no title of its own, and the scanner could not reach the first message to synthesize one. Forking a text-only conversation would not have hit the same ceiling.

## The patch

Insert one `ai-title` record synchronously after the fork's transcript is written and before `forkSession()` returns:

- Prefer the parent's title already in memory: `this.customTitles.get(parentId)`, populated moments earlier by the same function's forced `ensureSessionLoaded` reload.
- If the parent has none, use the first 100 characters of the fork's own first real user message.

The record lands before the webview's first list read, roughly 1.5 seconds later. The insertion has been a pure +453 or +454 byte addition on every version so far. Marker: `/*claude-fork-title-patch*/`.

The anchor regex uses `forkSession`, `loadedSessions.delete`, `ensureSessionLoaded`, `getTranscript`, `randomUUID`, `projectRoot`, and `promises.appendFile`, with `[\w$]+` capture groups and `[\s\S]*?` gaps. Two pre-write fixes came out of validation against every installed build:

- `\w` does not match `$`, and the minified bundle uses a bare `$` as an identifier.
- 2.1.258 inserted an `if(!WK($))throw Error("invalid session id")` guard at the top of `forkSession()`, which broke a tighter gap; `[\s\S]*?` absorbs it.

## Proving the fix, not just the absence of the symptom

"It stopped failing" can also mean the failure moved. The fix was confirmed with two independent structural fingerprints:

1. The next fork's injected title was byte-identical to the parent's actual title, not a generic fallback.
2. That record had no `timestamp` field, which only the patch's own object literal produces.

The record also sat immediately after the copied pre-fork content and before any new growth. The live extension-host log then showed the next `fork_conversation` event with no `restore_declined` follow-up.

An early lineage check produced false positives because many unrelated sessions shared identical `/model` command echo text. Restricting the match to the true first message at position 0, with non-trivial length, isolated the real fork family.

## The update-wipe trap

The extension updated from 2.1.246 to 2.1.250 partway through the investigation by replacing the whole versioned folder. 2.1.250's `forkSession` anchor was structurally identical and unpatched: same bug, new minified names.

The fix only still appeared to work because the open VS Code window's extension host was still running the old 2.1.246 code. The next window reload would have brought the bug back byte-for-byte. This happened independently of the no-split patch, and was caught only because the fix was re-verified against the installed file instead of assumed to still be active.

The bug then recurred on every update until the patch was added to the scheduled task that already maintained the no-split patch.

## The encoding incident

The first live run of the script returned exit 0 and left the marker present, yet corrupted 38 KB of unrelated strings. Windows PowerShell 5.1's `Get-Content -Raw` decoded the BOM-less UTF-8 bundle with the system codepage, and the UTF-8 write-back double-encoded it. The size delta was the tell: expected about +454 bytes, got +38,257.

The file was reverted from the script's own backup, the I/O was switched to `[System.IO.File]::ReadAllText` and `WriteAllText` with `UTF8Encoding($false)`, and the re-run produced a clean +454 byte single region. The generic integrity check in the shared entry point now catches this class for every patch: one contiguous span under 4 KB, strict UTF-8, otherwise restore and exit 2.

A permission classifier also refused a live edit to the extension host powering the session, through both Bash and PowerShell. That was correct: the response was to back up, validate the equivalent patch for the new build, and hand off a ready-to-run command, not to find another path around the refusal. The scheduled task runs outside any session, which is part of why putting the patch there was the durable answer.

## Evidence

- Three reproductions against the same parent session, each producing an orphaned fork file with zero `ai-title`, `custom-title`, `lastPrompt`, or `summary` records, confirmed by direct file inspection.
- Scanner constant `wQ=65536` and the four-branch fallback order read directly out of the minified bundle.
- Parent's opening message line measured at 280,617 bytes.
- Patch verified as +453 bytes with no BOM or line-ending changes; anchor matched exactly once before writing.
- Proof fork: `ai-title` matched the parent's real title byte-for-byte and was the only line in its file without a `timestamp` field.
- 2.1.250's anchor confirmed structurally identical and unpatched after the auto-update; 2.1.250 was never patched; 2.1.251 was hand-patched on 2026-08-31 (+453 bytes, one anchor match).
- Script validated against all four builds then installed (2.1.250, 2.1.251, 2.1.252, 2.1.258) before its first run.
- First script run on 2026-09-02 at 17:11: exit 0, marker present, +38,257 byte diff with mojibake in an unrelated locale table. Reverted from backup, fixed to `[System.IO.File]` I/O, re-run at 17:13 as a clean +454 byte single region.
- 2.1.259 on 2026-09-03 at 21:28:54: re-applied by the scheduled task with no human involved, in the same run as the no-split patch (`applied-now=[no-split,fork-title] exit=0`).
- 2.1.261 on 2026-09-05 at 17:19:54: applied on a hand run as one 454 byte span at offset 2,833,485, strict UTF-8, captured variables `$ / Y / W / r1 / U / O`, while the no-split patch failed safe on the same run and was fixed minutes later.
- Recurrence chain: 2.1.246 worked, 250 never patched, 251 hand-patched, 252 unpatched, 258 patched twice through the skill, 259 unattended, 261 hand run. Seven versions in nine days.

## What generalizes

- A scanner that reads a bounded window has a failure mode that stays invisible until a file crosses the threshold in the wrong place. Bugs gated on "content exceeds N bytes at position X" are easy to miss if test data never has oversized content at that position.
- "It stopped failing" is not proof. A byte-identical title plus an absent timestamp field pinned the fix to the exact code path that was written.
- Treat "forking is broken again" as the expected state after an extension update, not a surprise. Re-verify the installed file, not the running window's resident code.
- Exit 0 plus a marker is not a sufficient success check for a file rewrite. Check the diff shape and the encoding every time, in one shared place.
- `\w` does not match `$`, and minifiers use `$` as an identifier.
- A tool refusing to edit the process actively running it can be the correct answer. Validate and hand off; do not hunt for a bypass.

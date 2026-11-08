---
title: Patching and self-healing the Claude Code VS Code "new editor group" bug
period: 2026-08..2026-09
status: resolved, patch maintained across extension updates
applies_to: Claude Code VS Code extension 2.1.245 through 2.1.261 on Windows
---

# Patching and self-healing the Claude Code VS Code "new editor group" bug

**Symptom.** Clicking the Claude icon in the editor title bar opened a new editor group and split the window whenever no all-Claude tab group already existed. If one existed, the panel reused it and behaved correctly.

**Status.** Root cause found and patched. The fix survived eight extension versions over ten days (2.1.245 through 2.1.261), including one fully unattended update, and one structural change in 2.1.261 that the patch detected and refused to guess at until the matcher was widened.

## Root cause

The extension ships a webview panel. `createPanel()` decides which `ViewColumn` that panel opens in. Its fallback branch, used when no existing Claude tab group can be found, called `this.findUnusedColumn()` and set the "started in a new column" flag:

```js
if (existing && existing.viewColumn) column = existing.viewColumn;
else column = this.findUnusedColumn(), startedInNewColumn = true;
```

`findUnusedColumn()` returns a column that is not currently in use. VS Code has to create a new editor group to host it. That is the split.

The top-right icon runs the command `claude-vscode.editor.openLast` (confirmed from the extension's `package.json` `editor/title` menu contribution), which reaches `createPanel()` when the preferred location is not the sidebar.

## Finding it inside a 3 MB single-line bundle

`extension.js` is minified to roughly 941 lines, effectively one giant line, and 2.8 to 3.3 MB depending on build. Line-based search tools are useless on it. Locate byte offsets first, then read a window:

```bash
grep -aob "findUnusedColumn" extension.js
tail -c +<offset-150> extension.js | head -c 900
```

The same technique was used later to see what a new build had changed without loading the whole file into an editor.

## The patch

Before:

```js
if(K&&K.viewColumn)z=K.viewColumn;else z=this.findUnusedColumn(),Q=!0}
```

After:

```js
if(K&&K.viewColumn)z=K.viewColumn;else z=S4.ViewColumn.Active,Q=!1}/*claude-nosplit-patch*/
```

`S4` was that build's alias for the `vscode` namespace. Only the fallback changes: reuse the active column instead of allocating an unused one. The branch that reuses an existing Claude tab group is untouched. The marker comment makes subsequent runs idempotent.

Since 2.1.261 the guard reads `if(V)` because the tab-group lookup moved into a helper. The current matcher treats the `&&V.viewColumn` tail as optional.

## Making it survive auto-updates

The extension updates by replacing its whole versioned folder, so every hand patch is wiped. A Windows Scheduled Task runs a PowerShell entry point at logon and every six hours through a hidden `wscript` launcher. The task itself runs at limited rights; creating it needed one elevated registration on that machine, but every later behavior change was made by editing the scripts the task points at.

Per run, the patch script:

1. Picks the newest installed extension folder by numeric version, not string order.
2. Short-circuits if `/*claude-nosplit-patch*/` is already present.
3. Applies a structural regex that anchors on tokens minifiers preserve: `findUnusedColumn`, `viewColumn`, and the `"claudeVSCodePanel"` string literal. Variable names are captured, never hardcoded.
4. Requires exactly one match. Zero matches means the bundle was restructured and the script exits 1 without writing. More than one means ambiguity and also exits 1.
5. Takes a timestamped `.bak` before writing and writes UTF-8 through `System.IO.File`, preserving BOM state.

The matcher went through four versions:

| Version | Match strategy | Why it changed |
|---|---|---|
| v1 | Exact literal string | Broke on the first real update, 2.1.246, because the minifier renamed locals (`K` to `V`, `Q` to `Y`, namespace alias `S4` to `f0`) with no logic change |
| v2 | Regex anchored on method names, property names, and string literals, with capture groups for locals | Tolerated every later rename automatically; output verified byte-identical to a hand-derived patch |
| v3 | Added `[\w$]+` identifiers and UTF-8-safe I/O | The bundle uses a bare `$` as an identifier, which `\w` does not match |
| v4 | Made the `&&.viewColumn` guard an optional capture group | 2.1.261 hoisted the tab-group search into a helper and changed `if(V&&V.viewColumn)` to `if(V)` |

v3 is the version that mattered on 2026-09-05: 2.1.261 produced exactly one `0 matches, needs manual re-derivation` log line and touched nothing. v4 was validated against every build and backup on disk before being run: both `if` shapes matched once, and every already-patched file matched zero times.

## Write integrity belongs in the orchestrator

A second patch (the fork-title fix) once passed its own marker check and exit code while corrupting 38 KB of unrelated text through a PowerShell encoding mistake. The shared entry point now checks every write against the backup it just took:

- exactly one contiguous changed span
- that span under 4 KB
- the result strict-decodes as UTF-8

Anything else restores the backup and exits 2. The check sits in one place, so any patch added later inherits it.

## Evidence

- Bundle size: 2,980,939 bytes on 2.1.245; 3,256,530 bytes on 2.1.261. Reported as 941 lines, effectively one minified line.
- Run cost measured, not estimated: about 2.5 seconds wall clock and 105 MB peak working set for a single patch, dominated by PowerShell startup; 3.7 to 5.7 seconds for the combined two-patch entry point; zero resident footprint between runs.
- Real scheduled-task runs returned `LastTaskResult = 0` on 2026-08-26, 2026-08-27, and 2026-09-05.
- v1 logged its fail-safe message on five consecutive six-hour runs from 2026-08-26 15:08 through 2026-08-27 10:42 before v2 landed.
- v2's patch of a simulated fresh 2.1.246 file was byte-for-byte identical to the manual patch on the live install.
- Idempotency, fail-safe, and ambiguous paths were each re-tested against three disposable simulated version folders: already patched, fresh unpatched, structurally incompatible.
- 2.1.258: the combined entry point applied the no-split change as a 29 to 50 byte span at offset 3,120,195 (21 bytes added), strict UTF-8, in 5,702 ms.
- 2.1.259: healed fully unattended at 21:28 on 2026-09-03, `applied-now=[no-split,fork-title] exit=0`.
- 2.1.261: v3 failed safe at 17:19:53 on 2026-09-05. v4 applied at 17:21:13 as a 29 to 50 byte span at offset 3,174,376; file grew from 3,256,509 to 3,256,530 bytes; strict UTF-8; 3,697 ms.

## What generalizes

- Minifiers preserve method names, property names, and string literals. They freely rename local variables. Anchor on the stable tokens and capture the unstable ones.
- A structural change is the real test of "fails safe", and it eventually comes. The right response is the narrowest widening that covers the new shape, validated against every build on disk, not a looser anchor.
- "Fails safe" only works if it is designed in before the failure. A run that logs "cannot match, no changes made" is a success, not a silent outage.
- Split "create" from "run" for scheduled tasks. The task was registered once with elevation and then ran at limited rights forever. Later changes edited what the task executes, never the task itself.
- Automated persistence looks like persistence to security tooling, regardless of intent. A Startup-folder shortcut fallback was correctly blocked by the coding session's own classifier. The right move was to hand the prepared command to the user, not to route around the block.
- A GPU-composited Electron window is not reliably capturable with classic GDI `BitBlt`/`CopyFromScreen`; it returns solid black even when the window is enumerable. Extension-host logs plus a functional check were the usable verification path.

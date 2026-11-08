# Engineering notes

Field notes from debugging developer tooling on Windows. Each note records one failure: the symptom, the root cause, the patch or workaround, the versions tested, and the evidence that settled the conclusion.

The subjects are terminal multiplexers, coding agents, VS Code extension internals, and the Windows automation that keeps a fix alive after an auto-update. Notes are pinned to the versions in their frontmatter. These tools change; minified anchors, byte offsets, and internal handler shapes will move.

## Index

| Note | Area | Summary |
|---|---|---|
| [Patching the Claude Code VS Code "new editor group" bug](notes/2026-08-claude-vscode-new-editor-group.md) | VS Code extension | The panel fallback called `findUnusedColumn()`, forcing a split. A marker-based regex patch and a scheduled task carried the fix across eight extension versions. |
| [Patching the Claude Code VS Code "fork conversation loses all history" bug](notes/2026-08-claude-vscode-fork-title.md) | VS Code extension | `forkSession()` never wrote a title, and the list scanner reads only 64 KB of each session file. A synchronous title injection restored the fork, with a proof based on structural fingerprints rather than the absence of the symptom. |
| [Herdr Pi agent detection on Windows](notes/2026-09-herdr-pi-agent-detection.md) | Terminal multiplexer | Herdr matches executable names in the pane's process tree. npm installs `pi` as a `.cmd`/`.ps1` shim over generic `node.exe`, so a native `pi.exe` wrapper was built, and live sessions were reparented without a restart. |
| [OpenCode shared server in Herdr, database growth, and an upstream SSE connection leak](notes/2026-09-opencode-shared-server-sse-leak.md) | Coding agent | Six per-pane servers became one server plus attach clients. A later slowdown was a confirmed upstream SSE leak, not a frozen server; a database timestamp query prevented an unnecessary restart. |
| [Claude Code Plan mode's auto-classifier gap](notes/2026-09-claude-plan-mode-classifier.md) | VS Code extension | Plan mode prompted on every command because a new interactive-terminal gate closed a quiet path. Fixed by hopping through Auto mode on entry, or by changing the starting permission mode on a machine without that constraint. |
| [OpenCode subagent triggering and failure detection](notes/2026-11-opencode-subagent-triggering.md) | Coding agent | `opencode run --agent <subagent>` runs a primary agent instead. The note covers the two real trigger paths, the log fields that prove a child session, and how quota and proxy-ban failures were separated. |

## Conventions

- Paths use `%USERPROFILE%`, `%APPDATA%`, and `%LOCALAPPDATA%` placeholders. Account identifiers, hostnames, and credentials are removed.
- Times are IST.
- "Verified" means observed on the version listed. It is not a claim about later builds.
- Commands are Windows PowerShell unless marked otherwise.

## License

Prose is licensed under [Creative Commons Attribution 4.0 International](LICENSE). Code snippets are included for reference under the same terms.

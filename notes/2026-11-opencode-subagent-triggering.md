---
title: OpenCode subagent triggering and failure detection
period: 2026-11
status: resolved and verified
applies_to: OpenCode CLI, agent registry behavior observed in late 2026
---

# OpenCode subagent triggering and failure detection

**Symptom.** A wrapper script reported subagent failures, but no subagent session appeared in the chat or in the OpenCode log. The log showed `agent=build mode=primary`, not `agent=<subagent> mode=subagent`.

**Status.** Resolved. The wrapper was calling a command that cannot create a subagent. The real trigger paths and the evidence needed to prove a child session are below.

## `opencode run --agent <subagent>` is not a subagent

The CLI prints a warning and falls back:

```text
! agent "general" is a subagent, not a primary agent. Falling back to default agent
> build · mimo-v2.6-flash-free
```

`opencode run` launches primary agents only. A subagent type such as `general`, `explore`, or a custom `zen-test` is invoked through the `task` tool. Consequences:

- `--agent general` runs `build`, so the log line is `agent=build mode=primary`.
- A subagent session always has a `parentID`. A primary session does not.
- A subagent log line carries `mode=subagent`. A primary line carries `mode=primary`.

## The two real ways to trigger a subagent

### From the chat: the native `task` tool

The main agent calls `task` with a `subagent_type`:

```text
task(description="Visible subagent failure test", prompt="Reply with exactly OK", subagent_type="general")
```

This creates a child session with `parentID` set to the main session and `mode=subagent`. The chat shows the task call and its result. In the TUI, the subagent view opens with `ctrl+x down`.

A custom subagent type must exist in the live agent registry:

```json
{
  "agent": {
    "zen-test": {
      "description": "Temporary subagent for a quota failure test",
      "mode": "subagent",
      "model": "opencode/mimo-v2.6-flash-free"
    }
  }
}
```

The running session caches the agent registry at startup. `opencode debug agent zen-test` reads the fresh config, but the live `task` tool still reports `Unknown agent type: zen-test` until the session restarts. Changing an existing agent's model has the same problem: the CLI shows the new model, the live `task` tool keeps the old one.

### From the shell: a primary agent that calls `task`

Run a primary agent and instruct it to call `task`:

```powershell
opencode run --pure --agent build -m opencode-go/deepseek-v4.1-flash --format default `
  "Use the task tool with subagent_type=zen-test and the prompt 'Reply with exactly OK'. Report exactly what the subagent returned, including any error."
```

This produces a real child session. It is not visible as a chat subagent because it runs in a separate `opencode run` process. The log is the proof.

## Proving a subagent ran

Read the OpenCode log (`%USERPROFILE%\.local\share\opencode\log\opencode.log`) for the child session.

Creation line:

```text
message=created id=ses_<child>
  parentID=ses_<parent>
  title="Quota test (@zen-test subagent)"
  agent=zen-test
```

Model and mode line:

```text
message=stream providerID=opencode modelID=mimo-v2.6-flash-free
  session.id=ses_<child>
  agent=zen-test mode=subagent
```

Failure line:

```text
message="stream error" providerID=opencode modelID=mimo-v2.6-flash-free
  session.id=ses_<child>
  agent=zen-test mode=subagent
  error.error="AI_APICallError: Free usage limit reached. Free models work in OpenCode
  (https://opencode.ai); subscribe to OpenCode Go to keep going: https://opencode.ai/go"
```

The fields that matter:

- `parentID` set: the session is a child.
- `agent=<type>`: the subagent type that ran.
- `mode=subagent`: the runtime mode.
- `modelID`: the actual model.
- `error.error`: the real failure reason.

A primary run has `mode=primary` and no `parentID`. Do not accept a wrapper label as proof.

## What the fixed wrapper does

The wrapper now supports a real-subagent mode:

```powershell
subagent.ps1 zen-test -Subagent -NoClaude -Json -Models "opencode-go/deepseek-v4.1-flash" -TimeoutSec 90 "Reply with exactly OK"
```

With `-Subagent`, it runs `build` with a prompt that calls `task` with the requested subagent type. It also:

- passes a unique `--title` to correlate its own primary session in the log;
- captures the primary session ID from the `message=created` line carrying that title;
- captures the child session ID from the `message=created` line whose `parentID` is the primary;
- polls only that child `session.id` for errors, so another session's quota error cannot create a false positive;
- writes `subagentStarted` and `sessionId` into the receipt.

`subagentStarted: false` and `sessionId: null` mean the primary model failed before a child was created. That is a launch failure, not a subagent failure, and the wrapper reports it that way.

The Claude Code Sonnet fallback is a real Claude Code subagent, not an OpenCode child, and the receipt labels it `mode=claude-code`.

Failure kinds written to `%USERPROFILE%\.local\share\opencode\subagent-failures.jsonl`:

`quota`, `capacity`, `banned`, `model_not_found`, `agent_not_found`, `tool_denied`, `network`, `timeout`, `empty`, `output_mismatch`, `unknown`.

## Failure one: Zen free-model quota

Exact error:

```text
AI_APICallError: Free usage limit reached. Free models work in OpenCode
(https://opencode.ai); subscribe to OpenCode Go to keep going: https://opencode.ai/go
```

A provider quota limit, not a local config error. The free models stay unusable until the quota resets or the account moves to OpenCode Go. The paid model in this setup (`opencode-go/deepseek-v4.1-flash`) worked.

## Failure two: the proxy's account was banned

The proxy process was up and listening on port 8080, but its single linked Google account had been disabled by Google for a Terms of Service violation. Upstream returned 403 at 2026-11-05 22:43:45 UTC, which is 2026-11-06 04:13:45 IST:

```text
This service has been disabled in this account for violation of Terms of Service.
Please submit an appeal to continue using this product.
```

The proxy's account record had `isInvalid: true`, `invalidReason: "Account banned - Gemini disabled for Terms of Service violation"`, and `verifyUrl: null`. Every model through the proxy fails, not just Gemini, because all models share the one account. Retrying and re-authenticating do not help. The options are a Google appeal or linking a different account, with the same Terms risk.

The wrapper keeps the proxy as a last-resort fallback but probes it with a real model request and skips it while the account is invalid.

## Reproduction checklist

1. Confirm the fresh config:

```powershell
opencode debug agent general
opencode debug agent zen-test
```

2. Confirm the CLI warning:

```powershell
opencode run --pure --agent general -m opencode/mimo-v2.6-flash-free --format default "Reply with exactly OK"
```

3. Trigger a real subagent through a primary agent:

```powershell
opencode run --pure --agent build -m opencode-go/deepseek-v4.1-flash --format default `
  "Use the task tool with subagent_type=zen-test and the prompt 'Reply with exactly OK'. Report exactly what the subagent returned, including any error."
```

4. Prove the child session:

```powershell
Get-Content "$env:USERPROFILE\.local\share\opencode\log\opencode.log" |
  Select-String -Pattern "agent=zen-test mode=subagent"
```

5. For an in-chat subagent, restart OpenCode after adding the agent type, then call `task` with `subagent_type="zen-test"`. Without the restart, the cached registry rejects the custom type or uses the old model.

## Earlier mistakes

- The wrapper called `opencode run --agent general`, which silently ran `build`. Receipts said "general" but no subagent existed.
- The wrapper matched log errors from any session using the same model, so another session's quota error could look like its own failure. Title and child-session correlation fixed that.
- The running session's `task` tool kept the old model after the config changed, because the registry is cached. The fresh config is visible only through `opencode debug agent` or after a restart.
- A direct `opencode run --agent build` process did create a real child session, but that process is not part of the chat. Only the log proves it.

## Rules

- Use the native `task` tool when the subagent must appear in the chat.
- Use `opencode run --agent build` with an explicit `task` prompt when you need a real subagent from the shell.
- Never use `opencode run --agent <subagent-name>` and call it a subagent.
- Verify `mode=subagent`, `parentID`, and the child `session.id` before claiming a subagent ran.
- Treat `subagentStarted: false` as a launch failure, not a subagent failure.
- Restart OpenCode after changing agent models or adding agent types.
- Keep a banned proxy account out of the working fallback ladder.

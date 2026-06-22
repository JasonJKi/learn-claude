---
title: "Automating guardrails with Claude Code hooks"
slug: "hooks-and-automations"
date: "2026-06-22"
summary: "Learn how to use Claude Code hooks to enforce conventions, log activity, and block unsafe tool calls at the event level."
---

## What you'll learn

- What Claude Code hooks are and which four events they respond to
- How to configure hooks in `.claude/settings.json`
- How to write an audit log hook and an auto-format hook
- How to block a tool call from a hook using exit codes

## Background

Claude Code takes actions by calling tools — running bash commands, writing files, reading directories. Most of the time that's exactly what you want. But sometimes you need something extra to happen at the moment a tool fires: log the command for compliance, auto-format the file that was just written, or refuse a destructive operation before it runs.

You could add those requirements to your CLAUDE.md, but that relies on Claude choosing to follow them in every case. Hooks are different. They are shell commands that the Claude Code harness itself executes in response to tool events, regardless of what Claude decided. Claude doesn't see them unless they produce output that feeds back into the session.

This makes hooks the right tool for anything you'd call enforcement rather than guidance: audit trails, automated side effects, and hard blocks on dangerous operations.

## Core concept

Hooks are configured in `.claude/settings.json` (project scope) or `~/.claude/settings.json` (global scope) under a top-level `"hooks"` key. Each hook entry binds a shell command to a specific event and tool matcher.

Four events are available:

- `PreToolUse` — runs before a tool executes; exit code 2 blocks the tool call
- `PostToolUse` — runs after a tool completes; receives the tool's response
- `Stop` — runs when Claude finishes its turn
- `Notification` — runs when Claude emits a notification

Every hook command receives a JSON object on stdin. For `PreToolUse` and `PostToolUse`, that object contains `tool_name` (a string) and `tool_input` (an object whose fields match the tool's parameter schema). `PostToolUse` additionally includes `tool_response` and `tool_error`.

The `matcher` field is a regex matched against the tool name. A matcher of `"Bash"` hits only the Bash tool; `"Write|Edit"` hits either.

## Example

Here is a `.claude/settings.json` that adds two automations: an audit log for every bash command, and a Prettier pass after any file write or edit.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '\"[bash] \" + .tool_input.command' >> ~/claude-audit.log"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "file=$(jq -r '.tool_input.file_path // empty'); [ -n \"$file\" ] && npx prettier --write \"$file\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

To try it, save this as `.claude/settings.json` in your project root, then start a Claude Code session and ask Claude to create a file or run a command.

## How it works

**The `"hooks"` structure** — The top-level key maps event names to arrays of matcher objects. Each matcher object has a `"matcher"` regex string and a `"hooks"` array of commands to run when the matcher fires. Multiple matchers under the same event all run if they match.

**`"type": "command"`** — This is the only hook type currently supported. The value of `"command"` is passed to your shell as-is, with the tool event data available on stdin.

**Audit log command** — `jq` reads stdin, constructs a string prefixed with `[bash]`, and appends it to `~/claude-audit.log`. Because this is a `PreToolUse` hook, every bash command is logged before it executes — including commands that will fail.

**Auto-format command** — After a `Write` or `Edit` call, the hook extracts `.tool_input.file_path` from stdin and runs Prettier on it. The `// empty` in jq means "produce no output if the field is null", which causes `$file` to be empty and the Prettier call to be skipped safely. The `|| true` ensures a Prettier parse error doesn't surface as a hook failure.

**Exit codes** — A `PreToolUse` hook that exits with code 2 cancels the tool call and sends its stdout back to Claude as an error message. Exit codes 0 and 1 both allow the tool call to proceed; the difference is that 1 is noted in the log. `PostToolUse` hooks cannot block execution regardless of exit code.

## Common mistakes

**Not reading stdin.** Hook commands receive tool data on stdin as a JSON object. If your command ignores stdin, the data is silently discarded — but your command still runs. Use `jq`, `python3 -c 'import sys,json; d=json.load(sys.stdin)'`, or another JSON parser; don't assume you can get the data from environment variables.

**Matching too broadly.** A matcher of `".*"` runs your hook before and after every tool call in the session. If your hook does anything non-trivial, this stalls Claude noticeably. Match specific tool names whenever possible.

**Trying to block from PostToolUse.** Exit code 2 only cancels tool calls from `PreToolUse` hooks. In `PostToolUse`, a non-zero exit is logged but ignored. Side effects belong in `PostToolUse`; enforcement belongs in `PreToolUse`.

**Writing hook output expecting Claude to see it.** A hook's stdout goes to the Claude Code internal log, not to Claude's context. The only way hook output reaches Claude is via exit code 2 in `PreToolUse` — the hook's stdout becomes the error message Claude receives.

**Running slow commands in PreToolUse.** Every `PreToolUse` hook adds latency before the tool executes. Keep pre-hooks under 100ms. Move anything heavier — uploading logs, running full test suites — to `PostToolUse` or `Stop`.

## Next steps

- Add a `Stop` hook that sends a desktop notification when Claude finishes a long task: `osascript -e 'display notification "Claude is done" with title "Claude Code"'` on macOS, or `notify-send "Claude Code" "Done"` on Linux
- Block dangerous commands with a `PreToolUse` hook that exits 2 when the bash command matches a pattern: `jq -e '.tool_input.command | test("rm -rf /")' && echo "Blocked: destructive path" && exit 2`
- Combine hooks with [CLAUDE.md](/posts/understanding-claude-md-files) — hooks enforce at the harness level what CLAUDE.md states as intent
- Explore the official Claude Code hooks documentation for the full list of environment variables and advanced hook composition patterns

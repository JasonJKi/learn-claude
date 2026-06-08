---
title: "Hooks and automations in Claude Code"
slug: "hooks-and-automations"
date: "2026-06-08"
summary: "Learn how to use Claude Code hooks to run shell commands automatically on lifecycle events like tool calls, session stops, and notifications."
---

## What you'll learn

- What hooks are and which lifecycle events trigger them
- How to configure hooks in `.claude/settings.json`
- How to write a hook that blocks a dangerous tool call before it runs
- How to use a PostToolUse hook to automatically run tests after file edits

## Background

Claude Code can do a lot on your behalf — edit files, run commands, call external APIs. That power is useful, but it also means you're trusting Claude to stay in bounds. CLAUDE.md instructions help, but they only shape Claude's intent; they don't enforce it at the execution layer.

Hooks close that gap. A hook is a shell command that Claude Code runs automatically when a lifecycle event fires — before a tool call executes, after it completes, when the session ends, or when Claude sends a notification. Your shell command gets full context about the event via stdin, and its exit code can block or allow the action.

You'll reach for hooks when you want something that runs reliably regardless of what Claude decides to do: enforcing that tests pass after every file write, logging all Bash commands to a file for audit, blocking writes to a protected directory, or posting a desktop notification when a long task finishes.

## Core concept

Hooks are configured in `hooks` keys inside a Claude Code settings file — either `.claude/settings.json` for a project or `~/.claude/settings.json` globally. Each hook entry declares which event it listens to and a shell command to run.

Claude Code supports six lifecycle events:

- **PreToolUse** — fires before a tool call executes; a non-zero exit code blocks the call and surfaces your stderr as an error message
- **PostToolUse** — fires after a tool call completes; non-zero exit is logged but does not undo the action
- **Notification** — fires when Claude emits a user-facing notification (e.g. task complete)
- **Stop** — fires when Claude finishes its turn
- **SubagentStop** — fires when a subagent finishes its turn
- **PreCompact** — fires before the conversation is compacted to save context

When a hook runs, Claude Code passes a JSON object to its stdin describing the event. For `PreToolUse` and `PostToolUse` this includes the tool name and its input arguments. Your script reads that JSON and decides whether to proceed.

You can also filter which tool calls a hook applies to using a `matcher` field — a glob pattern matched against the tool name. This lets you write a hook that only fires for `Bash` calls, or only for `Write` calls.

## Example

This settings file configures two hooks: one that blocks any `Bash` command containing `git push --force`, and one that runs `npm test` after every file write.

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 ~/.claude/hooks/block-force-push.py"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash ~/.claude/hooks/run-tests-after-write.sh"
          }
        ]
      }
    ]
  }
}
```

The `block-force-push.py` hook reads the Bash command from stdin and exits non-zero if it sees a force-push:

```python
#!/usr/bin/env python3
import json, sys

event = json.load(sys.stdin)
command = event.get("tool_input", {}).get("command", "")

if "git push" in command and "--force" in command:
    print("Blocked: force-push is not allowed. Use --force-with-lease if you must.", file=sys.stderr)
    sys.exit(1)

sys.exit(0)
```

The `run-tests-after-write.sh` hook re-runs the test suite in the background and logs the result:

```bash
#!/usr/bin/env bash
set -euo pipefail

EVENT=$(cat)
FILE=$(echo "$EVENT" | python3 -c "import json,sys; print(json.load(sys.stdin)['tool_input']['file_path'])")

# Only run tests for source files, not generated or config files
if [[ "$FILE" == src/* && "$FILE" == *.ts ]]; then
  npm test --silent >> ~/.claude/test-log.txt 2>&1 || true
fi
```

## How it works

**Settings structure** — The top-level `hooks` object maps event names to arrays of matchers. Each matcher has a `matcher` glob (matched against the tool name) and its own `hooks` array with one or more `command` entries. This two-level array lets you attach multiple independent scripts to the same event.

**JSON on stdin** — Both hooks read `sys.stdin` / `$(cat)` to get the event payload. For `PreToolUse`, the payload always contains `tool_name` and `tool_input`; for `Bash` calls, `tool_input.command` is the shell string Claude is about to run. For `Write` calls, `tool_input.file_path` is the destination path.

**Exit code contract** — The force-push blocker exits 1 when it detects a forbidden pattern. Claude Code intercepts this, cancels the tool call, and shows the stderr message to Claude as an error. Claude then decides what to do next (typically it apologises and tries a safe alternative). Exiting 0 means "proceed normally."

**PostToolUse is advisory** — The test runner uses `PostToolUse`, which cannot undo the file write that already happened. A non-zero exit here is logged but doesn't block anything. The goal is visibility: you can tail `~/.claude/test-log.txt` while Claude works and see whether each edit kept the suite green.

**Matcher glob** — The `"Bash"` and `"Write"` matchers are exact tool name strings here, but you can use `"*"` to match all tools, or patterns like `"mcp__*"` to match all MCP tool calls.

## Common mistakes

**Not making hook scripts executable.** If your script file lacks the execute bit, the shell will refuse to run it with a "Permission denied" error. Run `chmod +x ~/.claude/hooks/your-hook.py` after creating any hook script.

**Forgetting that PreToolUse blocks the whole call.** A non-zero exit from a PreToolUse hook cancels the tool call entirely — it doesn't just warn Claude. If your logic is wrong (say, it matches too broadly), you'll find Claude unable to run any Bash command at all. Test your hook script manually with sample JSON before wiring it up.

**Printing to stdout instead of stderr for block messages.** Claude Code reads your hook's stderr as the error message it surfaces to Claude. Content written to stdout is ignored. When you want to explain why you blocked a call, always write to stderr.

**Running expensive commands unconditionally in PostToolUse.** If you run the full test suite after every single `Write` call, Claude will slow to a crawl during any multi-file refactor. Use the file path from the event payload to filter: only run tests for source files, only trigger linting for `.ts` or `.py` files, and so on.

**Putting hooks in the wrong settings file.** Project hooks live in `.claude/settings.json` (relative to the repo root). User-level hooks live in `~/.claude/settings.json`. A common mistake is adding a hook to the wrong file and wondering why it never fires. Project hooks run in addition to user hooks — they don't replace them.

## Next steps

- Add a `Stop` hook that plays a terminal bell (`printf '\a'`) so you know when Claude has finished a long task
- Explore [custom slash commands](/posts/custom-slash-commands) to trigger complex workflows on demand rather than on every tool call
- Combine hooks with [CLAUDE.md instructions](/posts/understanding-claude-md-files) — CLAUDE.md shapes intent, hooks enforce it
- Read the official [hooks documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) for the full event payload schema and advanced matcher syntax

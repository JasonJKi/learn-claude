---
title: "Hooks and automations in Claude Code"
slug: "hooks-and-automations"
date: "2026-06-29"
summary: "Learn how to use Claude Code hooks to run shell commands at lifecycle events, automating tasks like formatting, testing, and safety guardrails."
---

## What you'll learn

- What hooks are and which lifecycle events they can attach to
- How to configure hooks in `.claude/settings.json`
- How to use hook exit codes to block or allow tool calls
- How to build a practical guardrail that prevents Claude from committing directly to `main`

## Background

Claude Code is a capable agent, but capability without guardrails can cause problems. Claude will run the commands you ask it to run — including ones that format code, commit to git, or deploy to production. On a good day that's exactly what you want. On a bad day, it means a stray `git push --force` lands on `main` before you notice.

Hooks are the mechanism Claude Code provides to enforce your own rules at the tool level. Rather than relying on instructions in CLAUDE.md that Claude might interpret loosely, hooks are shell commands that the harness executes unconditionally — before or after every tool call, or when the session reaches specific milestones.

You might use hooks to auto-run a linter after every file write, log every bash command Claude runs, notify you when a long task finishes, or hard-block certain git operations regardless of what Claude decides.

## Core concept

A hook is a shell command bound to a lifecycle event. The Claude Code harness calls it at the right moment and passes context via stdin as a JSON object describing what's happening (which tool was invoked, what arguments it received, and so on).

Claude Code currently supports four hook events:

- **`PreToolUse`** — fires before a tool call executes. Your script can inspect the arguments and exit with code `2` to block the call entirely, or exit `0` to allow it through.
- **`PostToolUse`** — fires after a tool call completes. The hook receives the tool's output. Useful for side effects like running a formatter after every file write.
- **`Notification`** — fires when Claude sends a desktop notification. Use it to add custom alerting (a sound, a phone ping) on top of the built-in notification.
- **`Stop`** — fires when Claude finishes responding. Useful for cleanup, summaries, or alerting that a long-running task completed.

Hooks are configured per event in `.claude/settings.json` (project-level) or `~/.claude/settings.json` (global). Each hook entry specifies a `matcher` — a glob or regex that filters which tool names trigger the hook — and a `command` that the harness runs as a shell command.

The hook receives a JSON payload on stdin. A `PreToolUse` hook for the `Bash` tool sees the exact command string Claude is about to run. A `PostToolUse` hook for `Write` sees the file path and the content that was written. Your script reads stdin, decides what to do, and exits.

Exit codes carry meaning for `PreToolUse` hooks:

- `0` — allow the tool call to proceed
- `2` — block the tool call; Claude sees an error and can try a different approach
- Anything else — allow the call, but Claude is shown a warning with your script's stderr output

## Example

This configuration blocks any `git push` that targets the `main` or `master` branch, with a clear error message explaining why.

Create `.claude/settings.json` in your project:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "command": "python3 .claude/hooks/block_main_push.py"
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write",
        "command": "bash .claude/hooks/autoformat.sh"
      }
    ]
  }
}
```

Create `.claude/hooks/block_main_push.py`:

```python
import json
import sys

payload = json.load(sys.stdin)
command = payload.get("tool_input", {}).get("command", "")

if "git push" in command and any(branch in command for branch in ["main", "master"]):
    print("Blocked: direct push to main/master is not allowed.", file=sys.stderr)
    print("Create a branch and open a PR instead.", file=sys.stderr)
    sys.exit(2)

sys.exit(0)
```

Create `.claude/hooks/autoformat.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Read the PostToolUse payload from stdin
payload=$(cat)
file_path=$(echo "$payload" | python3 -c "import json,sys; print(json.load(sys.stdin)['tool_input']['file_path'])")

# Only format files that prettier knows about
case "$file_path" in
  *.ts|*.tsx|*.js|*.jsx|*.json|*.css|*.md)
    npx prettier --write "$file_path" --log-level silent
    ;;
esac

exit 0
```

Make the shell script executable:

```bash
chmod +x .claude/hooks/autoformat.sh
```

## How it works

**The settings structure** — `hooks` is a top-level key in `settings.json`. Each value is a list of hook objects. Having multiple objects under `PreToolUse` means all of them run for every matching tool call, in order.

**The `matcher` field** — `"Bash"` matches the built-in Bash tool by name. You can use `"Write"`, `"Edit"`, `"Read"`, or any MCP tool name. The matcher does a simple string comparison against the tool name, so `"Bash"` won't accidentally match a tool called `"BashHelper"`.

**Reading stdin in the Python script** — `json.load(sys.stdin)` parses the payload the harness sends. For `PreToolUse` on the Bash tool, `tool_input.command` holds the exact string Claude is about to pass to the shell. The script checks for the substrings `git push` and the branch name to decide whether to block.

**Exit code 2 as a veto** — When the Python script exits with `2`, the harness does not run the Bash command. Claude receives an error message containing the script's stderr output. Claude typically acknowledges the block and tries a different approach — in this case, it would create a branch first.

**The autoformat hook** — This runs after every `Write` tool call. It reads the file path from the payload and passes it to `prettier`. Because it exits `0` regardless of whether prettier ran, it never blocks Claude's work — it just silently fixes formatting on the side.

**Why separate files instead of inline commands** — The `command` field supports inline shell one-liners, but scripts longer than a few words belong in files. It keeps `settings.json` readable and lets you version-control your hooks logic separately.

## Common mistakes

**Forgetting to exit 0 in non-blocking hooks.** If your `PostToolUse` script crashes or exits non-zero, Claude sees a warning. For side-effect-only hooks (like the formatter), always end with an explicit `exit 0` so a prettier error doesn't surface as a Claude warning.

**Blocking too broadly with the matcher.** Using `"*"` as a matcher runs your hook before every tool call — including `Read` calls that are completely harmless. Match as specifically as possible to keep hook overhead low and avoid blocking unintended tools.

**Parsing the command string with regex instead of checking intent.** The `block_main_push.py` example does substring matching, which is simple and correct for `git push`. Avoid complex regex on command strings — they break on quoting variations. For nuanced cases, parse structured data (file paths, tool names) from the JSON payload rather than trying to parse shell syntax.

**Putting hooks only in the global settings.** A hook that enforces project-specific rules (like the `main` push block) belongs in `.claude/settings.json` at the project root, checked into version control, so every team member and every CI run gets the same guardrails. Your `~/.claude/settings.json` is the right place for personal preferences (custom notifications, personal logging) that shouldn't affect teammates.

**Not testing hooks before relying on them.** Hooks run as shell commands. A typo in the path or a missing `chmod +x` makes the hook silently fail (or noisily crash). Test each hook manually by running the command with sample stdin before trusting it to protect you in a session: `echo '{"tool_input":{"command":"git push origin main"}}' | python3 .claude/hooks/block_main_push.py`.

## Next steps

- Learn about [subagents](/posts/subagents) to understand how Claude can delegate work to specialised agents, and how hooks interact with multi-agent sessions
- Read the [official hooks documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) for the full payload schema for each event type
- Explore [custom slash commands](/posts/custom-slash-commands) for repeatable task workflows that complement hook-level automations
- Add a `Stop` hook to send yourself a phone notification when Claude finishes a long task — combine it with the `Notification` hook for finer-grained alerting

---
title: "Automating Claude Code with hooks"
slug: "hooks-and-automations"
date: "2026-07-06"
summary: "Learn how to wire Claude Code's tool calls into your existing toolchain using lifecycle hooks that run automatically on every write, edit, or bash command."
---

## What you'll learn

- The four hook event types Claude Code exposes and when each fires
- How to configure hooks in `.claude/settings.json`
- How to build a PostToolUse hook that gives Claude immediate lint feedback after every file edit
- How to use a PreToolUse hook to block writes to protected paths

## Background

Claude Code acts fast. It edits files, runs commands, and installs packages without waiting for permission on every step. That speed is valuable, but it also means problems surface late — Claude finishes a round of edits, you run the linter, and now there are fifteen errors to fix in a second pass.

Hooks close that feedback loop. They let you attach shell commands to Claude's tool lifecycle so your toolchain runs automatically at the right moment. Claude sees the output as part of the same turn, which means it can fix problems immediately instead of waiting for you to report them.

You can also use hooks defensively. A PreToolUse hook that exits non-zero cancels the tool call before it happens, making certain actions physically impossible rather than merely discouraged by a prompt.

## Core concept

A hook is a shell command that Claude Code runs when a lifecycle event fires. The four event types are:

- **PreToolUse** — runs before a tool executes; non-zero exit cancels the tool call and shows the output to Claude as an error
- **PostToolUse** — runs after a tool executes; output is shown to Claude in the same turn
- **Notification** — runs when Claude emits a desktop notification
- **Stop** — runs when Claude finishes generating a response

Each hook command receives the event's data as JSON on stdin. For PreToolUse and PostToolUse, that JSON includes `tool_name` and `tool_input` (and for PostToolUse, `tool_response` as well). Your hook reads those fields to make decisions — blocking specific file paths, skipping the linter for test files, or writing to an audit log.

Hooks are configured in `.claude/settings.json` under a `"hooks"` key. Each event type maps to a list of rules. Each rule has a `matcher` (a regex tested against the tool name) and a list of commands to run.

## Example

This configuration adds two hooks: a PostToolUse hook that auto-lints after every file write or edit, and a PreToolUse hook that blocks writes to protected paths.

`.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "npm run lint 2>&1 | tail -20"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/guard-paths.sh"
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/guard-paths.sh`:

```bash
#!/usr/bin/env bash
# Blocks writes to protected directories.
file_path=$(python3 -c "
import sys, json
data = json.load(sys.stdin)
print(data.get('tool_input', {}).get('file_path', ''))
")

protected=("infra/" ".env" "secrets/")

for prefix in "${protected[@]}"; do
  if [[ "$file_path" == "$prefix"* ]]; then
    echo "Blocked: writes to $file_path are not allowed" >&2
    exit 2
  fi
done
```

Make the script executable before using it:

```bash
chmod +x .claude/hooks/guard-paths.sh
```

## How it works

**The PostToolUse lint rule** matches any call to the `Write` or `Edit` tools using the `"Write|Edit"` regex. After Claude writes or edits a file, `npm run lint` runs automatically and its output is trimmed to the last 20 lines. If there are lint errors, Claude sees them in the same turn and fixes them before moving on. Substitute your project's lint command — `pnpm lint`, `cargo clippy`, `ruff check .` — or replace this entire command with whatever fast feedback matters most.

**The PreToolUse guard rule** uses the same matcher but fires before the write happens. It delegates to a shell script so the logic stays readable. The script reads the JSON event from stdin, extracts `tool_input.file_path`, and exits 2 if the path starts with any protected prefix. A non-zero exit from a PreToolUse hook cancels the tool call — Claude sees the stderr message and knows not to retry.

**Why a separate script file?** Inline commands grow unreadable quickly. Keeping logic in `.claude/hooks/` lets you test the script directly (`echo '{"tool_input":{"file_path":"infra/main.tf"}}' | bash .claude/hooks/guard-paths.sh`), version it with the rest of your project, and share it with your team without duplicating JSON escaping.

**Why `exit 2` instead of `exit 1`?** Both cancel the tool call. Exit code 2 is the conventional signal for a usage or input error, which is more accurate when the block is intentional — it distinguishes "something went wrong in the hook" from "this action is not permitted."

## Common mistakes

**Not reading from stdin.** The hook receives event data via stdin, not environment variables. If your hook ignores stdin entirely, it still runs — but it cannot inspect the tool's inputs, which makes conditional logic impossible. Always pipe stdin into your parser before making any decision.

**Writing slow hooks.** Every hook runs synchronously; Claude waits for it before continuing. Running a full test suite as a PostToolUse hook on every file write will make Claude feel stuck. Use `--silent` flags, pipe through `tail`, or scope expensive hooks to matchers that fire infrequently rather than on every edit.

**Matching too broadly.** A matcher of `.*` fires on every tool call, including `Read`, `Glob`, and `Grep`. Your linter then runs even when Claude is just reading a file. Use precise matchers like `"Write|Edit"` so hooks only fire when they're relevant.

**Using absolute paths in hook commands.** A path like `/home/alice/.claude/hooks/guard-paths.sh` breaks for every other developer on the team. Claude Code runs hook commands from the project's working directory, so relative paths like `bash .claude/hooks/guard-paths.sh` work for everyone who checks out the repo.

**Treating hooks as a substitute for CLAUDE.md constraints.** A CLAUDE.md instruction like "don't touch infra/" is advice Claude can overlook under pressure from a later prompt. A PreToolUse hook that exits non-zero is a hard block that no prompt can override. Use CLAUDE.md for guidance and context; use PreToolUse hooks when a constraint is non-negotiable.

## Next steps

- Add a `Stop` hook that plays a terminal bell (`tput bel`) so you know when Claude finishes a long-running task
- Explore [custom slash commands](/posts/custom-slash-commands) for repeatable multi-step workflows that go beyond what hooks can express
- Read the [official hooks documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) for the full event schema, all available hook types, and how hooks interact with permission modes
- Try pairing hooks with [subagents](/posts/subagents) so each agent's PostToolUse output triggers the next step in a pipeline automatically

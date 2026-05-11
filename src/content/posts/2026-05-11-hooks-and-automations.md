---
title: "Hooks and automations in Claude Code"
slug: "hooks-and-automations"
date: "2026-05-11"
summary: "Learn how to use Claude Code hooks to automatically run shell commands before or after tool calls, enforcing code quality and guarding sensitive paths."
---

## What you'll learn

- How to configure hooks in `.claude/settings.json`
- The difference between `PreToolUse` and `PostToolUse` hooks
- How to auto-format files every time Claude edits them
- How to block tool calls on protected paths using a hook exit code

## Background

Claude Code can write files, run shell commands, and make dozens of tool calls in a single session. That's powerful, but it creates a gap: Claude might edit a TypeScript file without running the formatter, or write to a generated directory you've already told it to avoid.

You can catch some of this with CLAUDE.md instructions, but instructions are advisory — Claude interprets them, and under pressure it can skip steps. Hooks are enforced at the tool level. The shell command runs regardless of what Claude decided.

Hooks are also useful for work that has nothing to do with Claude's reasoning: sending a desktop notification when a long task finishes, writing an audit log of every file touched, or uploading a coverage report after tests pass.

## Core concept

A hook is a shell command that Claude Code runs automatically at a named lifecycle event. You configure hooks in `.claude/settings.json` (project-level) or `~/.claude/settings.json` (user-level). Each event type accepts a list of matchers, and each matcher fires its command only when the tool name matches a regex.

The four most useful event types are:

- **`PreToolUse`** — runs before the tool call. Exit non-zero to block it entirely.
- **`PostToolUse`** — runs after the tool call completes. Useful for formatting, linting, or logging.
- **`SessionStart`** — runs once when a session begins. Useful for environment setup.
- **`Stop`** — runs when Claude finishes its turn. Useful for notifications.

Claude Code passes a JSON object to the hook's stdin describing the event: the session ID, tool name, and tool input (and, for `PostToolUse`, the tool response). Your hook reads that JSON to decide what to do.

For `PreToolUse` hooks, exit code `0` allows the call to proceed. Any non-zero exit code blocks it. Whatever the hook writes to stdout is shown to Claude as the reason, so it can course-correct.

## Example

This configuration adds two hooks to a TypeScript project. The first runs Prettier on any JS or TS file Claude writes or edits. The second blocks all writes inside `src/generated/`, which contains auto-generated code.

**.claude/settings.json**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/format-on-edit.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Write|Edit|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/guard-generated.sh"
          }
        ]
      }
    ]
  }
}
```

**.claude/hooks/format-on-edit.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

input=$(cat)
file=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [[ -z "$file" ]]; then
  exit 0
fi

if [[ "$file" =~ \.(js|jsx|ts|tsx)$ ]] && [[ -f "$file" ]]; then
  npx prettier --write "$file" --log-level silent
fi
```

**.claude/hooks/guard-generated.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

input=$(cat)
file=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [[ "$file" == src/generated/* ]]; then
  echo "src/generated/ is auto-generated. Edit the GraphQL schema in src/schema/ instead."
  exit 1
fi
```

## How it works

**`format-on-edit.sh`** reads the full event JSON from stdin, then uses `jq` to extract `tool_input.file_path`. If the path is a JS or TS file and actually exists on disk, it runs Prettier in silent mode. Exiting `0` either way tells Claude Code the hook succeeded — formatting is a side effect, not a gate.

**`guard-generated.sh`** does the same path extraction, but for a `PreToolUse` event. If the target file is inside `src/generated/`, the script prints a human-readable message to stdout and exits `1`. Claude Code sees the non-zero exit, cancels the tool call, and passes the stdout message back to Claude. Claude reads it, understands why the write was blocked, and can try a different approach.

The `matcher` regex `"Write|Edit|MultiEdit"` catches all three file-writing tools. A bare `"Bash"` matcher would catch all shell commands instead.

Both scripts use `// empty` in the `jq` expression as a safe fallback: if the field is missing (for example, on a tool that doesn't take a file path), the variable is empty and the script exits cleanly rather than crashing.

## Common mistakes

**Matching too broadly.** Omitting `matcher` or using `.` runs your hook on every single tool call, including `Read` and `Bash`. A formatting hook on `Read` is harmless but wasteful; a blocking hook on all tools will prevent Claude from doing almost anything. Always scope your matcher to the tool names you care about.

**Writing slow hooks.** A `PostToolUse` hook that takes two seconds runs after every matched tool call. If Claude makes twenty edits in one task, that's forty extra seconds. Keep hooks fast. If you need to run something slow (a full test suite, a type-check), trigger it from `Stop` instead, where it runs once at the end of the turn.

**Forgetting `set -euo pipefail`.** Without it, a failed `jq` command silently produces an empty string, your `if` condition evaluates unexpectedly, and the hook behaves incorrectly with no error message. Always add it to bash hooks.

**Using a hook for what CLAUDE.md handles better.** Hooks are for enforcement and automation, not for context. "Always use single quotes" belongs in CLAUDE.md. "Reformat after every edit" belongs in a hook. If the instruction requires no shell command, it doesn't need a hook.

**Not making the scripts executable.** If Claude Code can't execute the hook script, the tool call is blocked with a cryptic permission error. Run `chmod +x .claude/hooks/*.sh` and commit the mode change.

## Next steps

- Add a `SessionStart` hook to run `npm install` or activate a virtual environment at the start of every session, so Claude never hits a missing-dependency error mid-task
- Use a `Stop` hook to send a desktop notification (`osascript` on macOS, `notify-send` on Linux) when a long agentic task finishes
- Combine hooks with subagents: a `PostToolUse` hook can invoke a second Claude process to review each file edit before the main session continues
- Read the [official hooks documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) for the full list of event types and the complete stdin JSON schema

---
title: "Hooks and automations in Claude Code"
slug: "hooks-and-automations"
date: "2026-05-04"
summary: "Learn how to wire shell scripts to Claude Code lifecycle events so that linting, testing, and guardrails run automatically without you having to ask."
---

## What you'll learn

- What hooks are and which lifecycle events you can attach them to
- How to configure hooks in `.claude/settings.json`
- How to use a `PreToolCall` hook to block unsafe operations before they run
- How to use a `PostToolCall` hook to enforce code quality after every file edit

## Background

Claude Code will happily run commands, edit files, and call external tools on your behalf. That autonomy is the point — but it means quality gates that used to be enforced by developer habit now need to be enforced at the tool level. If Claude edits a file and you don't remind it to lint, it may not.

Hooks solve this by letting you attach shell scripts to specific moments in the Claude Code lifecycle. A `PreToolCall` hook runs before Claude executes a tool — giving you a chance to inspect and potentially block the call. A `PostToolCall` hook runs after — giving you a chance to react to what just happened. Neither requires Claude to remember to do anything; the hooks fire automatically every time.

This is especially valuable in CI-style workflows or shared projects where conventions must be enforced consistently, not just when someone remembers to ask.

## Core concept

Hooks are shell commands registered in `.claude/settings.json` under a `hooks` key. Each hook is associated with a lifecycle event and an optional matcher that limits which tool calls trigger it.

Claude Code currently supports four events:

- **`PreToolCall`** — fires before a tool is executed; a non-zero exit code cancels the tool call entirely
- **`PostToolCall`** — fires after a tool completes successfully
- **`SessionStart`** — fires once when a session begins
- **`Stop`** — fires when Claude finishes its turn

At runtime, Claude Code passes a JSON object to the hook script via stdin. The object contains the event name, the tool name, and the tool's input arguments. Your script can read this JSON to decide what to do.

A `PreToolCall` hook that exits with code `0` lets the tool call proceed. Any other exit code blocks the call, and whatever the script wrote to stdout is shown to Claude as the reason for the block — Claude then sees the message and can adjust its approach.

## Example

Here is a `.claude/settings.json` that registers two hooks: one that blocks `git push --force` before it runs, and one that runs ESLint after every file write.

```json
{
  "hooks": {
    "PreToolCall": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 .claude/hooks/block_force_push.py"
          }
        ]
      }
    ],
    "PostToolCall": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/lint_after_write.sh"
          }
        ]
      }
    ]
  }
}
```

The `PreToolCall` hook script at `.claude/hooks/block_force_push.py`:

```python
import json
import sys

event = json.load(sys.stdin)
command = event.get("tool_input", {}).get("command", "")

if "--force" in command and "git push" in command:
    print("Blocked: force-pushing is not allowed. Use a pull request instead.")
    sys.exit(1)

sys.exit(0)
```

The `PostToolCall` hook script at `.claude/hooks/lint_after_write.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

event=$(cat)
file_path=$(echo "$event" | python3 -c "import json,sys; print(json.load(sys.stdin).get('tool_input', {}).get('file_path', ''))")

if [[ "$file_path" == *.ts || "$file_path" == *.tsx ]]; then
  npx eslint --fix "$file_path"
fi
```

Make both scripts executable:

```bash
chmod +x .claude/hooks/block_force_push.py
chmod +x .claude/hooks/lint_after_write.sh
```

## How it works

**`PreToolCall` with a `Bash` matcher** — The `matcher` field limits the hook to a specific tool. Here it only fires when Claude calls the `Bash` tool. If Claude tries a different tool (like `Write`), this hook is skipped.

**Reading stdin in the Python script** — `json.load(sys.stdin)` deserializes the event payload that Claude Code pipes in. The `tool_input` key holds the arguments Claude was about to pass to the tool — in this case, the shell `command` string.

**Blocking with a non-zero exit** — When the script detects `--force` combined with `git push`, it prints a human-readable reason and calls `sys.exit(1)`. Claude receives the printed message and understands why the call was blocked. It can then try an alternative (like creating a PR instead).

**`PostToolCall` with a `Write` matcher** — This hook fires after every successful `Write` tool call. The `tool_input` for `Write` includes a `file_path` key, which the shell script extracts with a one-line Python invocation.

**Conditional linting** — The bash script only runs ESLint for `.ts` and `.tsx` files, so editing a Markdown file doesn't trigger an unnecessary lint run. ESLint's `--fix` flag applies auto-fixable style corrections immediately, before Claude moves on to the next step.

## Common mistakes

**Forgetting that `PreToolCall` only blocks the tool, not Claude.** If your hook blocks a `Bash` call, Claude sees the rejection message and may try again differently — for example, writing a script file and running it instead. Hooks enforce a specific tool call; they don't teach Claude a rule. Pair hooks with a CLAUDE.md entry that explains the constraint in plain language so Claude understands the intent.

**Writing hooks that fail silently.** If your hook script crashes with an unhandled exception, it may exit with code `1` and block the tool call without printing a useful message. Always wrap your script's main logic in a try/except (Python) or use `set -euo pipefail` plus explicit error messages (Bash) so failures are diagnosable.

**Using PostToolCall for validation that should be PreToolCall.** A `PostToolCall` hook can't undo the tool call that already happened. Use `PostToolCall` for reactions (linting, logging, notifications) and `PreToolCall` for enforcement (blocking destructive commands, checking preconditions).

**Matching too broadly.** A `PostToolCall` hook with no `matcher` fires after every tool call — including reads, searches, and anything else Claude does. That means your lint script runs after every `Read` too. Always specify a `matcher` to limit the hook to the tool that actually needs the reaction.

**Making hooks slow.** Hooks run synchronously in Claude's turn. A hook that takes 10 seconds to run will make every matched tool call feel sluggish. Keep hooks fast: lint one file, not the whole project. For expensive operations (full test suites), trigger them asynchronously and report results in a follow-up prompt.

## Next steps

- Add a `SessionStart` hook that prints a checklist of open TODOs from your issue tracker so Claude has context at the start of every session
- Combine hooks with [CLAUDE.md files](/posts/understanding-claude-md-files) — use CLAUDE.md to explain your rules in prose and hooks to enforce them mechanically
- Explore the full `settings.json` schema in the [Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code/settings) to see all available hook events and matcher options
- Try a `Stop` hook that posts a summary of what changed (via `git diff --stat`) to a Slack webhook so your team sees what Claude did each session

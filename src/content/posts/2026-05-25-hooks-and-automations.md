---
title: "Automating your workflow with Claude Code hooks"
slug: "hooks-and-automations"
date: "2026-05-25"
summary: "Learn how to use Claude Code hooks to run shell commands automatically before or after tool calls, so code quality and safety rules are enforced without manual prompting."
---

## What you'll learn

- What Claude Code hooks are and the four events they respond to
- How to configure hooks in `.claude/settings.json`
- How to auto-format files every time Claude edits one
- How to block unsafe tool calls before they happen

## Background

When Claude Code edits a file, it doesn't automatically run your linter. When it executes a shell command, nothing stops it from touching files you've declared off-limits in CLAUDE.md — unless Claude happens to re-read those instructions. Relying on Claude to remember and follow every rule every time is fragile. Instructions get compressed out of context. Edge cases get missed.

Hooks solve this by moving enforcement out of Claude's context and into the shell. A hook is a shell command that Claude Code runs automatically at a specific event — before a tool fires, after it completes, when Claude finishes a turn, or when the session ends. The hook runs regardless of what Claude "remembers," and a `PreToolUse` hook can hard-block a tool call if the command exits with a non-zero code.

This matters most in team settings where multiple people run Claude against a shared codebase, or in any project where mistakes are expensive. A hook that runs `prettier` after every file edit is cheaper than a code-review comment pointing out formatting violations.

## Core concept

Claude Code defines four hook events:

- **`PreToolUse`** — fires before a tool executes. If the hook exits non-zero, the tool call is cancelled and Claude sees the hook's stdout as the reason.
- **`PostToolUse`** — fires after a tool executes successfully. Useful for side effects like formatting, logging, or running tests on changed files.
- **`Notification`** — fires when Claude would show a desktop notification. Lets you route alerts to your own system.
- **`Stop`** — fires when Claude finishes its turn. Useful for summaries, metrics, or cleanup.

Hooks are configured in `.claude/settings.json` (project-scoped) or `~/.claude/settings.json` (global). Each hook entry specifies which tools it matches and the shell command to run. The hook command receives a JSON payload on stdin describing the event — the tool name, its input parameters, and (for `PostToolUse`) its response.

## Example

This example adds two hooks to a project: one that auto-formats JavaScript and TypeScript files after every edit, and one that blocks writes to a `generated/` directory.

First, create the hook scripts:

```bash
# .claude/hooks/format-on-save.sh
#!/bin/bash
input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [ -z "$file_path" ]; then
  exit 0
fi

if [[ "$file_path" =~ \.(js|ts|jsx|tsx)$ ]]; then
  npx prettier --write "$file_path" 2>/dev/null
fi
```

```bash
# .claude/hooks/block-generated.sh
#!/bin/bash
input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [[ "$file_path" == *"/generated/"* ]]; then
  echo "Writes to generated/ are not allowed. Edit the source schema instead."
  exit 1
fi
```

Make them executable:

```bash
chmod +x .claude/hooks/format-on-save.sh
chmod +x .claude/hooks/block-generated.sh
```

Then wire them up in `.claude/settings.json`:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/format-on-save.sh"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": ".claude/hooks/block-generated.sh"
          }
        ]
      }
    ]
  }
}
```

## How it works

**`format-on-save.sh`** reads the JSON payload from stdin and extracts the `file_path` field from the tool's input using `jq`. If the path ends in a JS or TS extension, it runs `prettier --write` on the file. Because this is a `PostToolUse` hook, the file already exists on disk when the script runs — `prettier` sees the content Claude just wrote and reformats it in place. The hook's exit code doesn't matter to Claude; it's purely a side effect.

**`block-generated.sh`** is a `PreToolUse` hook that runs *before* the write happens. It checks whether the target path contains `/generated/`. If it does, it prints a human-readable reason to stdout and exits with code `1`. Claude Code sees this non-zero exit, cancels the tool call, and shows Claude the printed reason as context. Claude can then explain to you what happened and suggest an alternative approach.

**The `matcher` field** uses a pipe-separated list of tool names. `"Edit|Write"` matches both the `Edit` tool (modifying existing files) and the `Write` tool (creating new files). You can match a single tool with just its name, or use `"*"` to match every tool.

**Settings.json scoping** — `.claude/settings.json` in your repo root applies to everyone who runs Claude Code in that project. `~/.claude/settings.json` applies only to you, across all projects. Team-wide hooks like `block-generated.sh` belong in the project file. Personal hooks (notification routing, personal logging) belong in the global file.

## Common mistakes

**Forgetting to make scripts executable.** A hook command that fails with `Permission denied` is treated as a non-zero exit — which means a `PostToolUse` hook that can't run silently does nothing, and a `PreToolUse` hook that can't run blocks every matching tool call. Run `chmod +x` on every hook script.

**Not handling missing fields in the JSON payload.** Not every tool passes a `file_path`. The `Bash` tool's input has a `command` field, not a `file_path`. If your script does `jq -r '.tool_input.file_path'` without a fallback, it will get `null` as a string and behave unpredictably. Always use `// empty` or `// ""` as a default in your `jq` expressions.

**Using `PreToolUse` hooks for post-hoc validation.** A `PreToolUse` hook sees the *intended* tool input, not the result. You can block a write to a protected path, but you can't check whether the written content passes a lint rule — the file doesn't exist yet. Use `PostToolUse` for content-based checks after the fact, or structure `PreToolUse` checks around input metadata only.

**Writing hooks that are slow.** Hooks run synchronously in Claude's tool-call loop. A `PostToolUse` hook that takes 10 seconds delays every file edit by 10 seconds. Keep hooks fast: run only what's needed, scope the matcher to specific tools, and avoid expensive operations like full test suite runs in hooks (run those in a `Stop` hook or a separate process instead).

**Putting secrets in the command string.** Hook commands appear in `.claude/settings.json`, which is typically checked into source control. Don't hardcode API keys or tokens in the command field. Read them from environment variables or a secrets file excluded by `.gitignore`.

## Next steps

- Explore the `Stop` hook to post a Slack message or write a session summary whenever Claude finishes a turn
- Combine hooks with [CLAUDE.md](/posts/understanding-claude-md-files) — use CLAUDE.md to describe conventions in prose, and hooks to enforce them mechanically
- Read about [custom slash commands](/posts/custom-slash-commands) to pair hooks with repeatable prompt workflows
- Check the Claude Code settings documentation for the full list of hook events and available fields in the stdin payload

---
title: "Hooks and automations in Claude Code"
slug: "hooks-and-automations"
date: "2026-05-18"
summary: "Learn how to attach shell commands to Claude Code lifecycle events so linting, formatting, and validation run automatically every time Claude touches a file."
---

## What you'll learn

- What hooks are and which lifecycle events Claude Code exposes
- How to configure hooks in `.claude/settings.json`
- How to write a hook script that reads tool input from stdin and acts on it
- How to block a tool call or surface errors back to Claude using exit codes

## Background

Claude can write and edit files quickly, but it doesn't automatically run your project's linter, formatter, or test suite after each change. If you're not paying close attention, you can end up with a pile of edits that all have the same fixable ESLint error — and you only find out when you go to commit.

The naive fix is to manually run checks after each Claude session. That works, but it breaks flow and is easy to skip. A better fix is to teach Claude Code itself to run those checks automatically.

Hooks are shell commands that Claude Code fires in response to specific lifecycle events — before or after a tool call, or when Claude stops generating. They run every time the matching event fires, regardless of what Claude is doing, which makes them ideal for enforcement: conventions can't slip through if the check runs automatically.

## Core concept

A hook is a shell command (or script) that Claude Code runs at a defined point in its execution cycle. You register hooks in `.claude/settings.json` under a `"hooks"` key, keyed by event name.

The three most useful events are:

- **`PreToolUse`** — fires before a tool call executes. A non-zero exit code cancels the call; Claude receives the stderr output and can explain the block to you or try a different approach.
- **`PostToolUse`** — fires after a tool call completes. A non-zero exit code signals failure; Claude sees the stderr output and can decide to fix the problem.
- **`Stop`** — fires when Claude finishes its turn. Useful for summaries or notifications, not for blocking.

Each hook entry specifies a `matcher` — a regular expression tested against the tool name — and a list of `hooks` objects, each with a `type` of `"command"` and a `command` string to run.

The hook process receives the full tool call as JSON on stdin. For a `Write` tool call, that JSON includes `tool_name` and `tool_input`, where `tool_input` contains `file_path` and `content`. Your script reads stdin, extracts what it needs, does its work, and exits. Exit 0 means success. Any other exit code means failure.

## Example

This example runs ESLint on every JavaScript or TypeScript file that Claude writes or edits. Create the hook script and the settings file, then make the script executable.

```bash
# .claude/hooks/lint-on-write.sh

#!/usr/bin/env bash
set -euo pipefail

input=$(cat)
file_path=$(echo "$input" | jq -r '.tool_input.file_path // empty')

if [[ -z "$file_path" ]]; then
  exit 0
fi

if [[ "$file_path" =~ \.(js|jsx|ts|tsx)$ ]]; then
  npx eslint --max-warnings=0 "$file_path" 1>&2
fi
```

```bash
chmod +x .claude/hooks/lint-on-write.sh
```

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "bash .claude/hooks/lint-on-write.sh"
          }
        ]
      }
    ]
  }
}
```

## How it works

**Reading stdin** — `input=$(cat)` captures the full JSON payload that Claude Code pipes to the hook process. The payload always contains `tool_name` and `tool_input`; the shape of `tool_input` varies by tool. For `Write`, it has `file_path` and `content`. For `Edit`, it has `file_path`, `old_string`, and `new_string`.

**Extracting the file path** — `jq -r '.tool_input.file_path // empty'` pulls the path out of the JSON. The `// empty` fallback means the variable stays empty (rather than the string `"null"`) if the key isn't present, so the early-exit guard on the next line works correctly.

**Filtering by extension** — The regex `\.(js|jsx|ts|tsx)$` restricts linting to JavaScript and TypeScript files. Without this filter, the hook would try to lint Markdown files, JSON, and anything else Claude writes, causing spurious failures.

**Redirecting ESLint output** — `1>&2` sends ESLint's stdout to stderr. Claude Code captures stderr from hook processes and surfaces it as context when the exit code is non-zero. If ESLint finds errors, Claude will see the exact error messages and can attempt a fix.

**The matcher regex** — `"Write|Edit"` matches both tool names so the hook fires whether Claude is creating a new file or modifying an existing one. Matcher strings are tested as regular expressions, so `|` is alternation, not a shell pipe.

**Settings file location** — `.claude/settings.json` at the repo root applies hooks to everyone who uses Claude Code on this project. To register hooks only for yourself without committing them, use `.claude/settings.local.json` instead (add it to `.gitignore`).

## Common mistakes

**Running slow hooks on every tool call.** Hooks block Claude Code while they run. A hook that takes five seconds will make every file write feel sluggish. Keep hooks fast: lint one file, not the whole project. If you need a full project check, put it in a `Stop` hook so it runs once per turn, not once per tool call.

**Forgetting to filter by tool or file type.** Without a file-extension check, a lint hook fires on every `Write` and `Edit` regardless of what was written. Trying to run ESLint on a `.md` file will always fail. Filter early and exit 0 for files you don't care about.

**Using hardcoded absolute paths.** A hook command like `/Users/alice/project/.claude/hooks/lint.sh` breaks the moment anyone else clones the repo. Use relative paths (`bash .claude/hooks/lint.sh`) or paths relative to `$PWD`, which Claude Code sets to the project root before running hooks.

**Not testing the hook script independently.** Because hooks receive stdin, you can test them directly from the terminal before wiring them up: `echo '{"tool_name":"Write","tool_input":{"file_path":"src/index.ts"}}' | bash .claude/hooks/lint-on-write.sh`. If the script exits correctly in isolation, it will work inside Claude Code.

**Blocking with PreToolUse when you only want to warn.** `PreToolUse` cancels the tool call on non-zero exit. That's appropriate for enforcing hard rules ("never write to `src/generated/`"), but for softer checks like linting, `PostToolUse` is the better fit — Claude can still see and react to failures without the tool call being outright rejected.

## Next steps

- Try a `Stop` hook that runs your full test suite at the end of each Claude turn, so you always know if the session left tests passing
- Read about [custom slash commands](/posts/custom-slash-commands) to pair hooks with repeatable prompt workflows
- Explore the `PreToolUse` event to enforce access boundaries — for example, blocking any `Bash` command that contains `rm -rf`
- Use `.claude/settings.local.json` to experiment with personal hooks before proposing them to your team as shared project hooks

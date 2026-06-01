---
title: "Hooks and automations in Claude Code"
slug: "hooks-and-automations"
date: "2026-06-01"
summary: "Learn how to use Claude Code hooks to run shell commands automatically before and after tool calls, enforcing conventions without relying on Claude's memory."
---

## What you'll learn

- What hooks are and which lifecycle events you can attach to
- How to configure hooks in `.claude/settings.json`
- How to build a working hook that auto-formats files after every edit
- How hooks differ from CLAUDE.md instructions and when to use each

## Background

CLAUDE.md lets you tell Claude what you want. Hooks let you enforce it. There's an important gap between those two things: an instruction in CLAUDE.md can be followed, misunderstood, or simply overridden when Claude is confident it knows better. A hook runs a real shell command at a real point in the workflow, unconditionally.

The practical difference shows up fast. You write "always run prettier before committing" in CLAUDE.md. Claude reads it, agrees, and then edits five files and creates a commit — sometimes remembering to run prettier, sometimes not. With a hook, the formatter runs automatically after every file write. No trust required.

Hooks also give you observability. You can log every bash command Claude runs, capture every file it edits, or post a notification when a session ends. That audit trail is useful both for debugging sessions that went wrong and for building confidence that automated Claude Code runs are doing what you expect.

## Core concept

A hook is a shell command (or list of commands) that Claude Code executes when a specific lifecycle event fires. You define hooks in the `hooks` section of `.claude/settings.json` (project-level) or `~/.claude/settings.json` (global).

Each hook entry has two fields:
- **`matcher`** — a string that filters which tool calls trigger the hook. For `PreToolUse` and `PostToolUse` events, the matcher is compared against the tool name (e.g. `"Edit"`, `"Bash"`, `"Write"`). An empty string matches everything.
- **`hooks`** — an array of hook objects, each with a `type` of `"command"` and a `command` string to execute.

The lifecycle events are:

| Event | When it fires |
|---|---|
| `PreToolUse` | Before Claude calls any tool |
| `PostToolUse` | After a tool call completes |
| `Stop` | When Claude finishes its turn |
| `SessionStart` | When the session initialises |

Claude Code passes context to hooks via environment variables and, for `PreToolUse`/`PostToolUse`, via a JSON payload on stdin. A `PreToolUse` hook can block the tool call by exiting with a non-zero status — this is how you enforce "never run `git push` without confirmation" at the infrastructure level, not just in prose.

## Example

This settings file adds two hooks: one that runs `prettier` on every file that Claude writes or edits, and one that prints a summary line to a log file whenever Claude runs a bash command.

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path // empty' | xargs -I{} npx prettier --write {} 2>/dev/null || true"
          }
        ]
      },
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '\"[\" + (now | strftime(\"%H:%M:%S\")) + \"] \" + .tool_input.command' >> /tmp/claude-bash.log"
          }
        ]
      }
    ]
  }
}
```

Save this as `.claude/settings.json` in your project root. You'll need `prettier` installed (`npm install --save-dev prettier`) and `jq` available on your PATH.

## How it works

**The matcher** `"Write|Edit"` is a substring match — Claude Code checks whether the tool name contains `Write` or `Edit`. This catches both the `Write` and `Edit` tools with one rule. The `"Bash"` matcher catches every `Bash` tool call.

**The JSON payload on stdin** — Claude Code writes a JSON object to the hook's stdin before running it. For `PostToolUse`, this object includes `tool_name`, `tool_input` (the parameters Claude passed to the tool), and `tool_response`. The prettier hook pipes stdin through `jq` to extract `.tool_input.file_path`, then passes that path to `prettier --write`.

**The `|| true` at the end** — If prettier exits non-zero (e.g., because the file is a `.json` that prettier can't parse), the hook would otherwise report failure. `|| true` keeps the hook from surfacing an error for every unsupported file type. Remove it if you want strict enforcement.

**The bash log hook** — `jq` reads the same stdin payload and formats a timestamped line using `strftime`. The `>>` appends to the log without truncating it between tool calls. This gives you a running record of every shell command Claude ran in the session.

**Hooks run synchronously** — Claude Code waits for the hook to finish before proceeding. Keep hooks fast. A hook that takes two seconds on every file edit will noticeably slow down sessions where Claude touches many files.

## Common mistakes

**Using `PreToolUse` when you mean `PostToolUse`.** If you want to format a file after it's written, use `PostToolUse`. A `PreToolUse` hook fires before the write happens — the file doesn't exist yet or still has its old contents.

**Forgetting that hooks receive stdin.** The JSON payload comes in on stdin, not as arguments or environment variables. If your hook command doesn't read stdin (e.g., a plain `echo "done"`), that's fine — unread stdin is harmless. But if you need the tool input, you must read and parse it from stdin, typically with `jq`.

**Writing hooks that fail noisily on unrelated tools.** A matcher of `""` matches every tool call. If your hook calls `jq -r '.tool_input.file_path'` on every tool, it will produce errors for tools that don't have a `file_path` input (like `Bash`). Use specific matchers and add `// empty` or `2>/dev/null` guards where needed.

**Putting secrets in the `command` string.** Hook commands are stored in `settings.json`, which is committed to the repo. Don't inline API keys or tokens. Load them from environment variables instead: `"command": "curl -H \"Authorization: Bearer $MY_TOKEN\" ..."`.

**Blocking yourself with `PreToolUse`.** A `PreToolUse` hook that exits non-zero cancels the tool call. This is powerful but easy to misconfigure. If your block condition is too broad (e.g., blocking all `Bash` calls), Claude will be unable to run any shell commands and the session will stall. Test block conditions carefully, and prefer `PostToolUse` logging over `PreToolUse` blocking until you're confident in the logic.

## Next steps

- Add a `SessionStart` hook to run `git fetch` and print a status summary so Claude always starts a session with fresh remote state
- Explore `PreToolUse` blocking to enforce a no-`git push` rule on specific branches, exiting non-zero when the branch name matches `main` or `master`
- Combine hooks with [custom slash commands](/posts/custom-slash-commands) — a slash command can set a flag in a temp file, and hooks can read that flag to change their behavior mid-session
- Check the [Claude Code hooks reference](https://docs.anthropic.com/en/docs/claude-code/hooks) for the full list of events and the complete stdin payload schema

---
title: "Hooks and automations in Claude Code"
slug: "hooks-and-automations"
date: "2026-06-15"
summary: "Learn how to use Claude Code hooks to run shell commands automatically before and after Claude's tool calls, enforcing conventions without relying on Claude's judgment."
---

## What you'll learn

- What hooks are and how they differ from CLAUDE.md instructions
- The four hook events and when each fires
- How to write a hook that lints, formats, or validates Claude's output
- How to block a tool call from a hook using exit codes

## Background

CLAUDE.md gives Claude instructions it can choose to follow. Most of the time that's enough — Claude is good at following rules written in plain English. But sometimes you need a guarantee, not a best effort. If your project requires that every edited file passes a linter before it's saved, you don't want to depend on Claude remembering that rule in the middle of a long session.

Hooks are the answer. They are shell commands that Claude Code executes automatically at specific points in its tool-use loop. A hook runs outside Claude's context window — it doesn't matter whether Claude forgot a rule or misread an instruction, the hook runs regardless. This makes hooks the right tool for enforcement: formatting, linting, secret scanning, audit logging, or anything else that must happen every time without exception.

You'll reach for hooks when you find yourself writing things like "always run the linter after editing" in CLAUDE.md and then catching Claude skipping it anyway, or when you want to log every file Claude writes to a ledger your team can audit.

## Core concept

Claude Code supports four hook events, each tied to a point in the tool-use lifecycle:

- **PreToolUse** — fires before Claude calls a tool. You can inspect the tool name and input, then block the call by exiting non-zero.
- **PostToolUse** — fires after the tool completes. You see the tool name, input, and output. Ideal for side effects like running a formatter on an edited file.
- **Notification** — fires when Claude would send the user a message. Useful for logging or alerting.
- **Stop** — fires when Claude finishes its turn entirely.

Hooks are configured in `.claude/settings.json` (project-scoped) or `~/.claude/settings.json` (global). Each hook entry specifies which event triggers it, an optional matcher on the tool name, and the shell command to run.

Claude Code passes context to each hook as a JSON blob on stdin. Your hook reads that JSON, does its work, and communicates back through its exit code and stdout. An exit zero means "proceed". A non-zero exit on a PreToolUse hook means "block this tool call and surface my stdout as the reason."

## Example

This configuration runs ESLint on every JavaScript or TypeScript file that Claude edits, and blocks the edit if linting fails:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'INPUT=$(cat); FILE=$(echo \"$INPUT\" | jq -r \".tool_input.file_path // .tool_input.path // empty\"); if [[ \"$FILE\" =~ \\.(js|ts|jsx|tsx)$ ]]; then npx eslint --max-warnings=0 \"$FILE\"; fi'"
          }
        ]
      }
    ]
  }
}
```

To add a PreToolUse hook that blocks Claude from deleting files outside the `src/` directory:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'INPUT=$(cat); CMD=$(echo \"$INPUT\" | jq -r \".tool_input.command\"); if echo \"$CMD\" | grep -qE \"rm |unlink \"; then if ! echo \"$CMD\" | grep -qE \"src/\"; then echo \"Blocked: rm commands must target files under src/\"; exit 1; fi; fi'"
          }
        ]
      }
    ]
  }
}
```

Both hooks read from stdin, parse the JSON with `jq`, and act on the relevant fields.

## How it works

**Event and matcher** — The top-level key (`PostToolUse`, `PreToolUse`) selects when the hook fires. The `matcher` is a regular expression tested against the tool name. `"Edit|Write|MultiEdit"` matches any of those three tools; an empty string matches every tool.

**stdin JSON** — Claude Code writes a JSON object to the hook's stdin before running it. For tool hooks it includes `tool_name` and `tool_input` (the arguments Claude passed) plus, for PostToolUse, `tool_response`. Your hook reads this with `INPUT=$(cat)` and extracts fields with `jq`.

**The ESLint hook** — After each edit, the hook extracts the file path from `tool_input.file_path` (the field name varies slightly by tool, so the `//` fallback chain handles both). It checks whether the extension is JS/TS, then runs ESLint. If ESLint exits non-zero, PostToolUse hooks do not block the action already taken (the file is already written), but Claude sees the hook's stdout and typically offers to fix the lint errors automatically.

**The rm guard hook** — PreToolUse hooks run before the tool executes, so an exit code of 1 here stops the Bash command from running at all. The hook inspects the command string, looks for `rm` or `unlink`, and then requires that `src/` appears somewhere in the command. If not, it prints a reason and exits 1. Claude receives that message and tells you it was blocked.

**Exit codes** — Exit 0 always means "ok, proceed." Exit non-zero on PreToolUse blocks the tool call and passes your stdout to Claude as context. Exit non-zero on PostToolUse is reported to Claude but doesn't undo what the tool already did.

## Common mistakes

**Using hooks for things CLAUDE.md handles fine.** Hooks add operational complexity — they can fail, time out, or produce confusing output. Use them only when you need a hard guarantee that can't be expressed as an instruction. Stylistic preferences ("prefer named exports") belong in CLAUDE.md, not hooks.

**Forgetting that PostToolUse can't undo the tool.** If you run a failing test in a PostToolUse hook and exit 1, the file has already been written. PostToolUse is for side effects and feedback, not for preventing damage. Use PreToolUse if you need to block something before it happens.

**Piping untrusted input directly to the shell.** Hook stdin comes from Claude's tool arguments, which ultimately derive from user prompts. Never construct shell commands by interpolating `$FILE` or `$CMD` directly into an unquoted string — always quote variables and prefer `jq` to extract values before using them.

**Writing hooks that are slow.** Every matched tool call waits for your hook to finish. A hook that takes three seconds per file edit will make Claude feel sluggish. Keep hooks fast: lint a single file, not the whole project; skip files by extension early; avoid network calls.

**Not testing hooks independently.** Before wiring a hook into Claude Code, test it by piping sample JSON to it directly: `echo '{"tool_name":"Edit","tool_input":{"file_path":"src/index.ts"}}' | bash your-hook.sh`. Hooks that crash silently or produce unexpected output are hard to debug once they're inside the Claude loop.

## Next steps

- Read the [official hooks documentation](https://docs.anthropic.com/en/docs/claude-code/hooks) for the full stdin/stdout schema and all supported hook types
- Combine hooks with [custom slash commands](/posts/custom-slash-commands) — a slash command can trigger a workflow that hooks then guard
- Try a Stop hook that posts a summary of what Claude changed to a Slack channel or writes to a log file your team monitors
- Explore [MCP servers](/posts/working-with-mcp-servers) for cases where hooks aren't enough and you need Claude to have access to new tools entirely

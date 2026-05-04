---
title: "Understanding CLAUDE.md files"
slug: "understanding-claude-md-files"
date: "2026-04-27"
summary: "Learn how to use CLAUDE.md to give Claude Code persistent, project-specific context that shapes every session without manual prompting."
---

## What you'll learn

- What a CLAUDE.md file is and where Claude Code looks for it
- How to write instructions that survive context compression
- How to layer project-level and directory-level CLAUDE.md files
- When to put something in CLAUDE.md versus a one-off prompt

## Background

Every time you start a Claude Code session, Claude starts fresh. It doesn't remember what you told it last week, which commands to avoid, or that your project uses tabs instead of spaces. Without a way to persist that context, you end up repeating yourself constantly — or worse, Claude makes the same wrong assumption on every run.

CLAUDE.md is the solution. It's a plain Markdown file that Claude Code reads automatically at the start of each session and injects into its context. Anything you write there becomes standing instructions for every interaction in that project, no copy-pasting required.

This matters most on real projects: multi-person repos where conventions need to be shared, long-running codebases where important constraints aren't obvious from the code, or any project where you find yourself writing the same prefixes to every prompt.

## Core concept

CLAUDE.md is just a Markdown file — Claude doesn't execute it, it reads it. The content becomes part of Claude's context before your first message, so it shapes every response in the session.

Claude Code looks for CLAUDE.md in a hierarchy of locations:

1. **`~/.claude/CLAUDE.md`** — your global personal preferences, applied to every project
2. **`CLAUDE.md` at the repo root** — project-wide conventions shared with your team
3. **`CLAUDE.md` inside a subdirectory** — scoped instructions that only apply when Claude is working in that subtree

All three layers are merged when Claude starts. A subdirectory's CLAUDE.md doesn't replace the root one — it adds to it.

What belongs in CLAUDE.md? Think of it as the onboarding document you'd write for a new engineer who already knows how to code. You want to cover things that aren't discoverable from reading the code: which commands to run before committing, which directories are off-limits, naming conventions, external systems Claude shouldn't touch, and anything that would surprise someone unfamiliar with the project.

## Example

Here is a CLAUDE.md for a TypeScript monorepo with a few real constraints:

```markdown
# my-monorepo

A TypeScript monorepo containing a Next.js frontend (`apps/web`) and an Express API (`apps/api`).

## Commands

- Install: `pnpm install` (never `npm install` or `yarn`)
- Test: `pnpm test` from the repo root runs all packages
- Type-check: `pnpm tsc --noEmit`
- Lint: `pnpm lint` — fix errors before committing

## Conventions

- All files use 2-space indentation and single quotes
- React components go in `apps/web/src/components/`, one component per file
- API route handlers live in `apps/api/src/routes/`
- Use `zod` for all runtime validation — never `joi` or `yup`

## Do not touch

- `apps/web/src/generated/` — auto-generated from GraphQL schema, edit the schema instead
- `.env.local` — contains real secrets, never read or log its contents
- `infra/` — Terraform, requires separate credentials, ask before modifying

## Testing

- Unit tests use Vitest, not Jest — import from `vitest`, not `jest`
- Test files are co-located with source files: `foo.ts` → `foo.test.ts`
- Run a single test file: `pnpm vitest run apps/web/src/components/Button.test.ts`
```

## How it works

**Header and description** — The first line names the project so Claude immediately knows what codebase it's in. The one-line description rules out confusion between similarly named projects.

**Commands section** — Claude will run commands on your behalf. Listing the correct ones here prevents Claude from defaulting to whatever it's seen most in training data. The `never npm install` line is the kind of specific constraint that saves you from a corrupted lockfile.

**Conventions section** — These are things Claude can't infer from the code unless it reads every file. Specifying `zod` avoids Claude importing a different validation library mid-feature because it looked plausible.

**Do not touch section** — This is the most important section for safety. Claude is eager to help, which means it will happily modify generated files or read secrets if you don't say otherwise. Explicit off-limits boundaries prevent hard-to-reverse mistakes.

**Testing section** — Test tooling is an area where training data is noisy (Jest is far more common than Vitest). Calling it out here means Claude writes correct test imports on the first try instead of the third.

## Common mistakes

**Writing vague instructions.** "Be careful with the database" does nothing. "Never run `DROP` statements or destructive migrations without confirmation" does. Be specific about the action you want to allow or prevent.

**Putting everything at the root level.** If you have a `backend/` directory with completely different conventions from `frontend/`, put separate CLAUDE.md files in each subdirectory. The root CLAUDE.md should only contain things that are universally true.

**Forgetting to keep it up to date.** CLAUDE.md rots like any documentation. When you migrate from Jest to Vitest, update the file the same day. Stale instructions are worse than none because they actively mislead.

**Using it as a prompt template.** CLAUDE.md is for standing context, not for task instructions. Don't write "When I ask you to write a component, always start with a test." Write task instructions in your actual prompts, or use slash commands for repeatable workflows.

**Making it too long.** Claude has a context window. A 2,000-line CLAUDE.md crowds out the code Claude is trying to read. Aim for under 100 lines. If a section is growing large, that's a sign the project needs better code-level conventions instead.

## Next steps

- Explore `~/.claude/CLAUDE.md` for global preferences you want in every project (editor style, personal tone preferences, common tool paths)
- Learn about [custom slash commands](/posts/custom-slash-commands) to handle repeatable tasks that are too dynamic for static CLAUDE.md instructions
- Try [hooks and automations](/posts/hooks-and-automations) to enforce CLAUDE.md conventions automatically at the tool-call level
- Read the [official Claude Code documentation](https://docs.anthropic.com/en/docs/claude-code) for the full CLAUDE.md specification

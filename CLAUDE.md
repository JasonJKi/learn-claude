# learn-claude

A blog about learning Claude Code — tips, patterns, and working examples.

## Project Structure

```
src/
  content/
    posts/       # Blog posts in Markdown
.claude/
  agents/        # Subagent instruction files
```

## Writing Posts

- All posts live in `src/content/posts/`
- Filename format: `YYYY-MM-DD-slug.md`
- Each post requires YAML frontmatter: `title`, `slug`, `date`, `summary`
- Follow the structure in `.claude/agents/content-writer.md`

## Conventions

- Use sentence case for headings
- Code examples must be working and self-contained
- No placeholder or lorem ipsum content
- Commit messages follow Conventional Commits: `docs: add post on <topic>`
- Branch names: `docs/blog-YYYY-MM-DD-slug`

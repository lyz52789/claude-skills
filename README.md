# Claude Skills

Personal Claude Code skills collection for reliable agentic workflows.

## Installation

### Option 1: Claude Code (Recommended)

```bash
# Clone to your skills directory
git clone https://github.com/lyz52789/claude-skills.git ~/.claude/skills/custom

# Or symlink an existing checkout
ln -s /path/to/claude-skills ~/.claude/skills/custom
```

### Option 2: Per-Project

Copy individual `SKILL.md` files into your project's `.claude/skills/` directory.

## Skill Index

| Skill | Trigger | Purpose |
|-------|---------|---------|
| [checking-task-splits](skills/checking-task-splits/) | Task plan generation | Auto-validate `.task.md` quality |
| [create-worktree](skills/create-worktree/) | `git worktree`, isolated workspace, new branch | Create isolated git worktrees with symlinked dependencies |
| [mcp-connection-guard](skills/mcp-connection-guard/) | MCP issues, session start | Detect and fix MCP server disconnects |

## Adding a New Skill

1. Create a directory under `skills/<skill-name>/`
2. Write `SKILL.md` with frontmatter:
   ```yaml
   ---
   name: your-skill-name
   description: One-line trigger description
   ---
   ```
3. Follow the existing pattern: Overview → When to Use → Core Pattern → Quick Reference
4. Keep it actionable — skills are instructions for an agent, not documentation for humans

## Naming Conventions

- **Directory name**: kebab-case, descriptive (`mcp-connection-guard`, not `mcp`)
- **Skill `name` field**: same as directory name, no collisions
- **One skill per directory**: prevents the "flat SKILL.md" ambiguity

## License

MIT

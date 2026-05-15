---
name: checking-task-splits
description: Use when generating implementation plans with .task.md files, after Write tool creates or modifies task split documents, or when task-checker hook reports validation failures
---

# Checking Task Splits

## Overview

Auto-validate task split quality after plan generation. Uses local `task-checker` CLI to enforce size limits, dependency checks, and atomicity rules.

## When to Use

- After `writing-plans` generates `.task.md` files
- After editing existing task files
- When `PostToolUse` hook reports `[task-checker]` output
- Before dispatching subagents to execute tasks
- When task size report shows >100K tokens

**Do NOT use for:** One-off scripts without task splits, or plans without `.task.md` files.

## Core Pattern

```
Generate Plan → Write Tasks → Run task-checker → Fix Issues → Commit
```

1. **Find all task directories**: Search `tasks/` and `docs/**/plans/**/tasks/`
2. **Run check**: `task-checker check <dir>`
3. **Run report**: `task-checker report <dir>` for size analysis
4. **Fix blockers**: Split oversized tasks, eliminate circular deps

## Quick Reference

| Command | Purpose |
|---------|---------|
| `task-checker check .` | Validate all .task.md in directory |
| `task-checker report .` | Generate task-size-report.md |
| `find . -name "*.task.md"` | Discover task files |

## Implementation

**Hook configuration** (in `~/.claude/settings.json`):

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write",
        "hooks": [
          {
            "type": "command",
            "command": "bash -c 'SEARCH_ROOTS=(\"$HOME/Documents/Rush/docs/superpowers/plans\" \"$HOME/Documents/Rush/tasks\"); for root in \"${SEARCH_ROOTS[@]}\"; do [ -d \"$root\" ] || continue; find \"$root\" -name \"*.task.md\" -maxdepth 3 2>/dev/null | grep -q . && { echo \"[task-checker] $root\"; /Users/lyz/.local/bin/task-checker check \"$root\" 2>/dev/null || true; }; done'"
          }
        ]
      }
    ]
  }
}
```

**Manual check** (when hook missed or failed):

```bash
# Find all local task directories
find ~/Documents/Rush -name "*.task.md" -not -path "*/node_modules/*" | xargs -I {} dirname {} | sort -u

# Check each
task-checker check /path/to/task/dir
task-checker report /path/to/task/dir
```

**Remediation checklist**:
- [ ] All tasks ≤ 100K tokens (or flagged with `token_note`)
- [ ] No circular dependencies in `dependencies:` frontmatter
- [ ] Each task has `input`, `output`, `acceptance criteria`
- [ ] `task-size-report.md` generated and reviewed

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Hook only checks `plans/`, misses `tasks/` | Add both paths to `SEARCH_ROOTS` |
| `maxdepth 2` misses nested task dirs | Use `maxdepth 3` or higher |
| task-checker not in PATH | Use absolute path: `/Users/lyz/.local/bin/task-checker` |
| Oversized tasks (>100K) not split | Break into smaller atomic tasks |
| Missing `task-size-report.md` | Run `task-checker report` before review |

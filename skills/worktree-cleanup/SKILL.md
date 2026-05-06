---
name: worktree-cleanup
description: Find and remove stale git worktrees across all projects. Use this skill when the user says "clean up worktrees", "prune stale worktrees", "remove old worktrees", "worktree cleanup", or "I have too many worktrees". Scans all repos under common project directories, identifies worktrees whose branch has been merged or deleted on the remote, and interactively removes them. Always use this skill before the user manually hunts through worktrees — it's faster and safer.
---

# Worktree Cleanup

Find and remove stale git worktrees across all projects. A worktree is stale if its branch has been merged into main, deleted on the remote, or abandoned (no commits in 30+ days and no open PR).

## Step 1: Find all worktrees

Scan common project roots:

```bash
find ~/Projects ~/Work ~/code ~/dev -maxdepth 3 -name ".git" -type d 2>/dev/null | sed 's|/.git||' | while read repo; do
  git -C "$repo" worktree list --porcelain 2>/dev/null | grep "^worktree " | awk '{print $2}' | while read wt; do
    echo "$repo|$wt"
  done
done
```

Skip Cursor IDE worktrees (`~/.cursor/worktrees/`) and any worktree path that is the repo's main working directory — only list additional worktrees, not the primary one.

## Step 2: Classify each worktree

For each worktree found, determine its status:

**Merged** — branch was merged into main/master:
```bash
git -C <repo> branch --merged main | grep -q "<branch-name>"
```

**Remote branch deleted** — branch no longer exists on origin:
```bash
git -C <repo> ls-remote --heads origin <branch-name> | grep -q . || echo "deleted"
```

**Abandoned** — last commit older than 30 days AND no open PR:
```bash
git -C <worktree-path> log -1 --format="%cr" 2>/dev/null
gh pr list --head <branch-name> --state open --json number 2>/dev/null | grep -q "number" || echo "no pr"
```

**Active** — has recent commits or an open PR. Leave these alone.

## Step 3: Present findings

Group by repo and show a summary before touching anything:

```
Worktree audit
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
bluetemberg (23 worktrees, 18 stale)
  🟡 merged    feature/registry-rule-collections   last commit 3w ago
  🟡 merged    fix/path-traversal-check             last commit 2w ago
  🔴 no remote feat/some-abandoned-experiment       last commit 8w ago
  ✅ active    feature/current-work                 PR #42 open

mg-infrastructure (15 worktrees, 11 stale)
  🟡 merged    feat/nest-api-infrastructure         last commit 1w ago
  ...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Total: 29 stale worktrees across 2 repos
```

## Step 4: Confirm before removing

Ask the user which categories to remove:

> "Remove all merged (🟡), all with deleted remote branches (🔴), or let me pick individually?"

Do not remove anything until the user confirms. Never remove worktrees marked as active.

## Step 5: Remove confirmed worktrees

For each confirmed worktree:

```bash
git -C <repo> worktree remove <worktree-path> 2>/dev/null || git -C <repo> worktree remove --force <worktree-path>
```

Use `--force` only if the normal remove fails (untracked files left behind). After removal, run:

```bash
git -C <repo> worktree prune
```

Report how many were removed per repo.

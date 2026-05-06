# claude-settings

Shared Claude Code skills and hook configs for the team.

## Skills

Skills are instruction files that Claude reads when triggered by natural language. Install them once, use them by asking.

| Skill | What it does | Say something like |
|---|---|---|
| `pre-pr` | Runs project-appropriate checks before opening a PR (fmt, lint, build, changelog, commit format) | *"is this ready to PR?"* |
| `self-review` | Reviews your local diff for security issues, incomplete implementations, and design gaps before PR submission | *"review my changes before PR"* |
| `migration-check` | Validates Payload CMS migration sequence and checks if `payload-types.ts` needs regeneration | *"check migrations"* |
| `worktree-cleanup` | Finds and removes stale git worktrees across all projects | *"clean up my worktrees"* |
| `friction-audit` | Analyzes your git history to find recurring friction patterns and suggests skills to build | *"analyze my workflow"* |

### Installing skills

**Option A — Claude Desktop (recommended):**
Go to Settings → Integrations → Add skills repo → paste `https://github.com/jravas/claude-settings`

**Option B — Manual:**
Copy the skill directories into `~/.claude/skills/`:
```bash
git clone https://github.com/jravas/claude-settings /tmp/claude-settings
cp -r /tmp/claude-settings/skills/* ~/.claude/skills/
```

## Hooks

Hook configs are snippets to merge into your project's `.claude/settings.json`.

| Hook | What it does |
|---|---|
| `pre-pr-review.json` | Intercepts `gh pr create` and runs `self-review` before proceeding |

### Applying a hook

Copy the relevant section from the hook file into your project's `.claude/settings.json`. If the file doesn't exist yet, create it at `<your-project>/.claude/settings.json`.

Example for `pre-pr-review`:
```bash
cat hooks/pre-pr-review.json
# Merge the "hooks" block into your .claude/settings.json
```

Commit the resulting `settings.json` (not `settings.local.json`) so the whole team gets the same hooks.

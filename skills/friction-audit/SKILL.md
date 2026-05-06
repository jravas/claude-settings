---
name: friction-audit
description: Analyze a developer's git history and Claude usage to identify workflow friction patterns and suggest skills worth building. Use this skill when someone says "analyze my workflow", "what skills should I create", "find my bad habits", "audit my Claude usage", "what's causing friction in my workflow", or "help me improve my Claude setup". Works by scanning git commit history across projects for recurring patterns — fix-of-fix commits, retriggers, formatting fixes, reverts — and mapping them to concrete skill suggestions. Great for onboarding to Claude Code or for periodic workflow reviews.
---

# Friction Audit

Analyze the developer's git history to find recurring pain points and suggest Claude skills that would eliminate them. Be honest and specific — vague observations aren't useful. Every finding should be backed by commit evidence.

## Step 1: Discover projects

Find git repositories to analyze:

```bash
# Common project root locations
find ~/Projects ~/Work ~/code ~/dev -maxdepth 2 -name ".git" -type d 2>/dev/null | sed 's|/.git||'
```

If that finds nothing, ask the user where their projects live.

List the repos found and ask the user to confirm which ones to include. Skip repos with fewer than 10 commits — not enough signal.

## Step 2: Collect commit data

For each confirmed repo, run:

```bash
cd <repo>
git log --oneline --no-merges -60 --format="%h %s"
```

Also get file churn:
```bash
git log --no-merges -60 --name-only --format="" | sort | uniq -c | sort -rn | head -20
```

Collect this data for all repos before analyzing. Do the analysis after you have everything — patterns that appear across multiple repos are more significant than single-repo patterns.

## Step 3: Identify friction patterns

Scan the commit messages for these signal patterns:

### Formatting / style fix commits
Commits where the subject contains: `fmt`, `format`, `style`, `lint`, `whitespace`, `blank line`, `trailing`, `align`
**AND** the type prefix is `fix` or `chore`

Signal: formatting is not being caught before commit. The hook or pre-commit check exists (or should exist) but isn't being run.

### Fix-of-fix commits
Commits where the subject contains `address`, `review`, `feedback`, `PR comment`, or where a `fix:` commit immediately follows another `fix:` on the same files.

Signal: author review gap. Issues are being caught by reviewers rather than the author before submission.

### Retrigger / re-run commits
Commits where the subject contains `retrigger`, `re-run`, `trigger CI`, `empty commit`, `force run`.

Signal: CI/validation is being used as a development loop instead of running validation locally first.

### Revert commits
Any commit starting with `Revert`.

Signal: feature shipped without clear requirements or sufficient pre-implementation planning.

### Hot files (churn)
Files appearing in 6+ of the last 60 commits.

Signal: either unclear requirements forcing repeated changes, or a component with too many responsibilities.

### Post-merge cleanup
Commits containing `leftover`, `remove unused`, `cleanup after`, `missed in`, `forgot to`.

Signal: commits are being opened before the implementation is complete.

## Step 4: Map patterns to skills

For each pattern found with 3+ supporting commits, map it to a skill category:

| Pattern | Skill to suggest |
|---|---|
| Formatting fix commits | Pre-commit hook or `pre-pr` skill that runs fmt before staging |
| Fix-of-fix / address review | `self-review` skill to catch issues before PR submission |
| Retrigger commits | `pre-pr` skill with local validation step for whatever CI runs |
| Hot files (same component churning) | Depends on the file — ask the user what keeps changing there |
| Post-merge cleanup | `pre-pr` skill with a "did you leave anything behind?" checklist |
| Revert commits | Planning habit (not a skill fix), but worth flagging |

Skills that appear for multiple patterns are the highest priority.

## Step 5: Check for existing skills

Before recommending skills to build, check if similar skills already exist:

```bash
ls ~/.claude/skills/ 2>/dev/null
find ~/Projects -name "SKILL.md" -path "*/.claude/*" 2>/dev/null
```

Don't recommend building something that already exists. If a relevant skill exists but the pattern is still occurring, the skill may need improvement or its trigger description may need to be more aggressive.

## Step 6: Report

Structure the report as:

---

**Friction audit — [N] repos analyzed, [N] commits reviewed**

### Patterns found

For each pattern, list:
- What it is (one sentence)
- Evidence: 3 example commit hashes + messages
- How often it appears (X commits in the last 60)

### Recommended skills

For each suggested skill:
- Name and what it does
- Which pattern(s) it kills
- Rough trigger phrases

### Not worth a skill

Patterns that are one-offs, already covered, or better addressed by a habit change rather than automation. Be honest here — not everything needs a skill.

---

## Step 7: Offer to generate

After the report, ask:

> "Want me to generate any of these skills? I can write the SKILL.md files and either add them to a local `.claude/skills/` directory or to a shared repo if you have one."

If the user has a `claude-settings` repo or similar shared skills repo, suggest adding them there so the whole team benefits.

When generating skills, follow the format from the skill-creator skill: YAML frontmatter with `name` and `description`, markdown body with clear step-by-step instructions, trigger description tuned to activate on natural phrases the developer would actually use.

---

## Notes on attribution

Before flagging a pattern as the current developer's habit, check commit authors:

```bash
git log --no-merges -60 --format="%ae %s" | grep -i "fmt\|retrigger\|address review"
```

Only report patterns where the current user's email appears. Don't attribute a colleague's habits to the wrong person.

---
name: pre-pr
description: Run pre-PR validation checks before opening a pull request. Use this skill whenever the user says "prepare PR", "pre-PR checks", "is this ready to PR?", "ready to open PR", "run checks before PR", or similar. Detects the project type automatically (Terraform, TypeScript/Node, PHP) and runs the appropriate validation suite — formatting, linting, type checking, changelog, and commit message validation. Always use this skill before the user opens a PR to catch the issues that would otherwise require fix commits after review.
---

# Pre-PR Checks

Run the full pre-PR validation ritual for the current project before `gh pr create`. The goal is to catch everything that would otherwise become a fix commit after the PR is open.

## Step 1: Detect project type

Check the current working directory for these markers (check all that apply — a project can be multiple types):

- **Terraform**: any `*.tf` files or a `.terraform/` directory
- **TypeScript/Node**: `package.json` + `tsconfig.json`
- **PHP/Twig**: `composer.json` or `*.twig` files anywhere under `src/` or `resources/`

If the working directory doesn't clearly match any type, ask the user which checks to run.

## Step 2: Run checks by project type

### Terraform

Run all of these:

1. `terraform fmt -recursive --check`
   - If it fails: run `terraform fmt -recursive` to show what needs fixing, then tell the user to run it and re-stage
2. Look for a style check script (common locations: `scripts/style_check.py`, `scripts/validate.py`, `Makefile` with a `lint` or `validate` target) — if found, run it
3. Check if a plan-validator or `terraform plan` has been run recently (look for recent plan output in the terminal or `.terraform/` state). If there's no evidence of a local plan run, warn: "TFC will run a speculative plan — run the plan-validator locally first to avoid a retrigger commit if it fails"
4. Check `moved.tf` / `import.tf` conventions: if any resource was renamed or imported, confirm the correct block exists

### TypeScript / Node

Run all of these:

1. `npm run lint` (or `yarn lint` / `pnpm lint` — detect from `package.json` scripts)
2. `npm run build` — catches type errors that lint misses
3. Check `CHANGELOG.md`: look for an `## [Unreleased]` section with at least one entry. If missing, warn the user to add one.
4. If tests exist (`npm run test` or `vitest` / `jest` config present), ask the user if they want to run them now

### PHP / Twig

Run all of these:

1. `composer run lint` or `./vendor/bin/phpcs` if configured
2. Check for new translation keys: `git diff origin/main..HEAD -- '*.php' '*.twig'` and scan for any new `t(`, `trans(`, or `__()` calls — if found, warn to verify all locale files are updated

## Step 3: Validate commits (all project types)

1. `git log origin/main..HEAD --oneline` — list all commits on this branch
2. Validate each commit message starts with a conventional prefix: `feat`, `fix`, `chore`, `refactor`, `docs`, `test`, `perf`, `ci`, `build`
   - Flag any that don't match
3. `git diff origin/main..HEAD --name-only` — list every changed file
   - Cross-reference against the PR description if the user has one drafted, or ask: "Do you have a PR description drafted? I'll check that all promised files are actually in the diff."

## Step 4: Report

Output a summary table:

```
Pre-PR check results
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ terraform fmt        clean
❌ style check          3 issues (see below)
✅ conventional commits all good
⚠️  plan-validator      no evidence of local plan run
✅ changelog            [Unreleased] section present
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

For each ❌ or ⚠️, give the exact command or change needed to fix it. Don't just say "failed" — tell the user exactly what to run or edit.

Only tell the user they're ready to open the PR once all ❌ items are resolved. ⚠️ warnings are advisory.

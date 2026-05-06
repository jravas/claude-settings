---
name: self-review
description: Review your own code changes before opening a PR — catching security issues, incomplete implementations, and design problems that would otherwise be caught by reviewers. Use this skill when the user says "review my changes before PR", "self review", "check my diff for issues", "what did I miss", "anything I forgot", or "review before I push". This is different from the built-in `review` skill which reviews an existing PR — this skill reviews local uncommitted or unpushed changes before submission. Always suggest this skill when the user is about to open a PR on a feature that touches security-sensitive areas (auth, file paths, user input handling, cache invalidation).
---

# Self-Review

Review your local changes before opening a PR, focusing on what reviewers typically catch that authors miss. The goal is to find issues now rather than after 2-3 post-merge fix commits.

## Step 1: Get the diff

```bash
git diff origin/main..HEAD
```

If the branch hasn't been pushed yet: `git diff HEAD` against the last clean commit.

Read the full diff carefully before categorizing anything.

## Step 2: Security scan

Look for these specific patterns in the diff:

**Path traversal**
- User-controlled input used in file path construction without sanitization
- `path.join()`, `fs.readFile()`, `__dirname +` with variables that came from request params or user data
- PHP: `file_get_contents()`, `include`, `require` with unsanitized variables

**Unvalidated input**
- New function parameters accepted from HTTP requests, CLI args, or config files that are used without validation
- Missing bounds checks on numeric inputs
- String inputs used in commands, queries, or evals without escaping

**Hardcoded secrets**
- API keys, tokens, passwords, connection strings in code (not env vars)
- Private keys or certificates

**Auth gaps**
- New endpoints or routes with no auth check
- Middleware that was applied to existing routes but not the new one

## Step 3: Completeness scan

**Unfinished work**
- `TODO`, `FIXME`, `HACK`, `XXX` in new lines (not pre-existing ones)
- Commented-out code blocks added in this diff
- Functions or methods that are defined but never called anywhere in the diff or existing codebase (dead code added)

**Missing error handling**
- `async` functions with no `try/catch` or `.catch()` on IO operations (file, network, DB)
- Unhandled promise rejections in new code
- PHP: unchecked return values from functions that can fail

**Type safety (TypeScript)**
- Public function parameters typed as `any`
- Type assertions (`as SomeType`) that bypass type checking on untrusted data
- Missing return type on exported functions

## Step 4: Design scan

**Incomplete implementations**
- PR touches a feature in one place but misses the symmetric change elsewhere (e.g., adds a new cache key but doesn't invalidate it anywhere, or adds a field to a form but not the schema)
- New config option added but not documented in the relevant config file or CLAUDE.md

**Consistency**
- New code uses a different pattern than surrounding code for the same operation (different error handling style, different logging approach)
- Naming inconsistencies with the rest of the module

## Step 5: Output

Categorize findings by severity and report them:

```
Self-review findings
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔴 SECURITY  src/registry/index.ts:142
   User-supplied `filePath` passed to fs.readFile() without path.resolve() + prefix check
   Fix: validate that resolved path starts with the allowed base directory

🟡 DESIGN    src/cache/invalidate.ts:89
   New cache version method added but dashboard still calls purgeAll() at line 203
   Fix: update dashboard/CacheController.php to use the new version-bump method

🔵 COMPLETE  src/types.ts:34
   TODO comment in new code: "// TODO: handle edge case where token is expired"
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

If there are zero findings across all categories, say so explicitly — "I found nothing worth flagging" is a useful result.

For each finding, give:
- The file and line number
- What the problem is (one sentence)
- What the fix is (one sentence or command)

Don't pad the list. Only flag things you're actually concerned about, not theoretical issues that the existing codebase already accepts.

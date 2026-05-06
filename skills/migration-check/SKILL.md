---
name: migration-check
description: Validate Payload CMS migrations before committing or opening a PR — checking for sequence conflicts, orphaned files, and stale generated types. Use this skill when the user says "check migrations", "validate payload schema", "migration conflict", "are my migrations ok", "regenerate types", or after any change to Payload collections or config. This skill is specific to projects using Payload CMS (look for `payload.config.ts` or `src/migrations/`). Always suggest it after schema changes to catch the "regenerated conflicted migrations" pattern before it reaches a PR.
---

# Migration Check

Validate the state of Payload CMS migrations and generated types before committing. Catches the issues that typically result in "generated one migration to rule them all" or "regenerated conflicted migrations" commits.

## Step 1: Check migration file consistency

Scan `src/migrations/` (or wherever migrations live — check `payload.config.ts` for a custom path):

**Duplicate timestamps**
List migration filenames and check for duplicate timestamp prefixes:
```bash
ls src/migrations/*.ts | grep -v index
```
Migrations are named like `20240315_120000_add_users_field.ts`. If two files share the same timestamp, one will silently overwrite the other when run. Flag any duplicates.

**Index sync**
Read `src/migrations/index.ts` and extract the list of imported migration files.
Cross-reference against the actual files on disk:
- Files on disk but not in `index.ts` → will never run, silent omission
- Files in `index.ts` that don't exist on disk → will crash at runtime

Report both categories with the exact filenames.

**Migration order**
Check that the export order in `index.ts` matches chronological filename order. Payload runs migrations in the order they're exported — a later migration exported before an earlier one can fail.

## Step 2: Check payload-types.ts freshness

Determine if `payload-types.ts` (usually at `src/payload-types.ts`) is stale:

```bash
# Find schema source files
git diff origin/main..HEAD --name-only | grep -E 'collections|payload\.config\.ts|globals'
```

If any collection files (`*.collection.ts`, `collections/*.ts`) or `payload.config.ts` changed in this branch AND `payload-types.ts` was NOT also modified, it's likely stale.

Stale types mean TypeScript will compile against the old schema — type errors won't surface until runtime.

**Fix command:**
```bash
npx payload generate:types
```

Then re-stage `payload-types.ts` before committing.

## Step 3: Check for merge conflict artifacts

```bash
grep -r "<<<<<<\|>>>>>>\|=======" src/migrations/ 2>/dev/null
```

Merge conflict markers left in migration files will cause a runtime crash when Payload tries to parse them. If found, stop everything — the user needs to resolve the conflict before continuing.

## Step 4: Report

```
Migration check results
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ No duplicate timestamps
❌ index.ts missing entry    20240521_093012_add_promotions_tab.ts
⚠️  payload-types.ts stale   collections/Promotions.ts changed, types not regenerated
✅ No merge conflict markers
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

For each ❌: the exact file and the fix (usually: add the import to `index.ts` in the right chronological position, or delete the orphaned file).

For each ⚠️: run `npx payload generate:types` and stage the result.

Don't mark as ready to commit until all ❌ items are resolved.

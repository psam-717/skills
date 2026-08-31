# Alembic Multi-Head Migration Fix

## Symptom

After deploying a new migration, Render fails with:

```
FAILED: Multiple head revisions are present for given argument 'head';
please specify a specific target revision, '<branchname>@head' to narrow
to a specific head, or 'heads' for all heads
```

## Root Cause

The new migration's `down_revision` pointed to an intermediate revision instead of the actual head. Since another migration already branched from that intermediate revision, Alembic saw two heads.

Example: if the chain was `... → 456c43acf169 → e8e9e970b822 → c1a2b3d4e5f6`, and the new migration `567c43acf170` had `down_revision = "456c43acf169"`, then both `c1a2b3d4e5f6` and `567c43acf170` were heads.

## Diagnosis

List all migrations and their parents:

```bash
for f in migrations/versions/*.py; do
  echo "$(basename $f): rev=$(grep 'revision:' $f | head -1 | cut -d"'" -f2) parent=$(grep 'down_revision:' $f | head -1 | cut -d"'" -f2)"
done
```

The head is the revision that is NOT referenced as `down_revision` by any other file. If two revisions aren't referenced as parents, you have two heads.

## Fix

1. Change the `down_revision` of the new migration to point to the actual head (e.g., `c1a2b3d4e5f6` instead of `456c43acf169`).

In `migrations/versions/567c43acf170_add_note_entries_table.py`:
```python
# Before (wrong — creates fork):
revision = "567c43acf170"
down_revision = "456c43acf169"  # ← intermediate, not head!

# After (correct — linear chain):
revision = "567c43acf170"
down_revision = "c1a2b3d4e5f6"  # ← actual head
```

2. Update the docstring header to match:
```python
"""add_note_entries_table

Revision ID: 567c43acf170
Revises: c1a2b3d4e5f6         # ← update this line too
...
"""
```

3. Amend the commit, force-push the branch, and create/open a PR:

```bash
git add migrations/versions/567c43acf170_*.py
git commit --amend --no-edit
git push --force https://psam-717:$(gh auth token)@github.com/psam-717/psam_vault_backend.git <branch>
gh pr create --base main --head <branch> --title "fix: correct Alembic migration chain for <feature>" --body "..."
```

4. Once merged, deploy to Render. `alembic upgrade head` should now apply exactly one migration cleanly.

## Prevention

Always find the actual head before creating a migration:

```bash
# Check the most recent file's revision ID
tail -1 migrations/versions/*.py | grep -o "revision.*'[^']*'" | head -1
```

The `down_revision` must be the LATEST revision ID, not just any ancestor.

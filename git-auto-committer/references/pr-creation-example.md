# PR Creation Example — Export/Uninstall Backend Endpoints

This reference captures the real PR creation flow from the psamvault backend session.

## Context

After committing 8 files individually with `git commit -m "feat: ..."`, the agent assessed the changes as **major** (new features across schemas, controllers, and routes) and asked the user for approval before proceeding.

## Step-by-step Commands Used

### 1. Create a new branch

```bash
# From the branch that has the commits (e.g. 'browser_login')
git checkout -b feat/export-uninstall
```

Branch naming: use `feat/<description>` for features, `fix/<description>` for fixes.

### 2. Push the branch

```bash
git push -u origin feat/export-uninstall
```

The `-u` sets upstream tracking so future pushes from this branch use `git push` without args.

### 3. Write a PR body file

```bash
cat > .pr_body.md << 'EOF'
## Summary

Explains what was changed and why in concise markdown.

### New Endpoints

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/path` | GET | Description |

### Changes by Layer

- **Schemas**: description
- **Controllers**: description
- **Routes**: description
EOF
```

### 4. Create the PR

```bash
gh pr create \
  --title "type: short description" \
  --body-file .pr_body.md \
  --base main
```

### 5. Clean up

```bash
rm .pr_body.md
```

## Important

- **Always ask the user first** using `clarify` before creating any branch or PR
- **Check `gh auth status`** first to confirm GitHub CLI is authenticated
- If `gh` is not available, use `curl` + `GITHUB_TOKEN` as documented in the `github-pr-workflow` skill
- Use `--base main` when the repo's default branch is `main`

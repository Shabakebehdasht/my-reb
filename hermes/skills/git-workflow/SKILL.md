---
name: git-workflow
description: "Git pitfalls: nested repos, staging, identity."
version: 1.0.0
author: Sydney
license: MIT
metadata:
  hermes:
    tags: [git, workflow, pitfalls, staging, nested-repos, commit]
    category: software-development
---

# Git Workflow

Pitfalls and procedures for everyday git operations that fall outside standard
`gh` CLI workflows (for gh-specific flows see the `github` skill).

## Standing Rules

- Always check `git status` and `git remote -v` before staging or pushing.
- Configure per-repo identity before first commit if global config is absent.
- Never assume a branch name — read it from `git branch --show-current`.
- Inspect `git status --short` and the relevant diff at startup. Treat pre-existing changes as user-owned: do not stage, discard, or claim them as task work.
- Verify the current branch's upstream is the remote branch with the **same name**. A status line such as `branch...origin/beta` proves synchronization with the wrong branch, not delivery of the current branch.

## Dirty Startup and Tracking Checks

```bash
branch=$(git branch --show-current)
git status --short
git branch -vv
git config --get "branch.$branch.remote"
git config --get "branch.$branch.merge"
```

If the checkout was seeded with tracking from another branch, push explicitly first and then repair only the local tracking config:

```bash
remote=$(git config --get "branch.$branch.remote")
git push "$remote" "HEAD:refs/heads/$branch"
git config --local "branch.$branch.remote" "$remote"
git config --local "branch.$branch.merge" "refs/heads/$branch"
```

Do not change remote URLs merely to alter branch synchronization. Verify delivery with both hashes:

```bash
git rev-parse HEAD
git rev-parse "refs/remotes/$remote/$branch"
```

## Pitfalls

### Nested `.git` directories when copying content

When `cp -r` (or similar) a directory that contains its own `.git` into another repo,
`git add` detects the nested `.git` and stages only a submodule reference (mode 160000),
not the actual files. Clones of the outer repo will not contain the copied content.

**Fix — remove nested `.git` before staging:**

```bash
cp -r /source/dir target/inside/repo
rm -rf target/inside/repo/.git
git add target/inside/repo/
```

If already staged as submodule:

```bash
git rm -r --cached target/inside/repo/
rm -rf target/inside/repo/.git
git add target/inside/repo/
git commit -m "Add dir contents (fix nested repo)"
```

### Missing git identity on fresh clones

New clones may lack both global and per-repo `user.name`/`user.email`.
Commits fail with `empty ident name`.

**Fix — set per-repo config before first commit:**

```bash
git config user.email "user@users.noreply.github.com"
git config user.name "username"
```

Detect from `gh auth status` or set manually.

## Verification

- `git status` shows no unexpected submodule entries.
- `git diff --cached --stat` shows actual file additions, not just mode changes.
- Commit and push succeed without warnings about embedded repos.

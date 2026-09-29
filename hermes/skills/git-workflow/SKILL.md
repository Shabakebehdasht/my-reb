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

## Isolating a Change Onto Its Own Branch

When a work branch carries unrelated commits (dependency bumps, doc plans, another issue's work) and the user wants a PR containing only the fix, do not open the PR from that branch — every unrelated commit rides along.

**Rule:** branch from the upstream default, then cherry-pick just the fix, so the PR diff is provably scoped.

```bash
git fetch https://github.com/<upstream>/<repo>.git <default>:refs/remotes/canonical/<default>
git checkout -b <issue-slug> canonical/<default>
git cherry-pick <fix-sha>
git diff canonical/<default>..HEAD --stat   # proves scope
```

`git diff <base>..HEAD --stat` is the scope check — read it before pushing, and confirm no unrelated file appears. A fetch by URL into a private ref keeps remote configuration untouched, which matters when the remotes are managed for you. Re-run the targeted tests on the isolated branch; a cherry-pick can conflict with a different base.

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

### Never guess or mutate a secret you only saw masked

A redacted/masked value in a config file or terminal output means you do **not** have the credential — you have a placeholder. Never run a credential-mutating statement (`ALTER USER ... PASSWORD`, `redis CONFIG SET`, key rotation) with a value you inferred, guessed, or copied from documentation: it silently replaces a working secret with a wrong one and locks out every service using it, and the original is unrecoverable from the masked view.

**Rule:** if you must change a credential, read the real value from a file programmatically without printing it, and verify connectivity immediately afterwards. If you have already broken it, restore from the same file and re-verify with a real connection probe (`db:show`, a ping, a test run) — not by re-reading the masked output.

```bash
PW=$(grep -E '^DB_PASSWORD=' .env | head -1 | cut -d= -f2-)
printf "ALTER USER u WITH PASSWORD '%s';\n" "$PW" | docker exec -i pg psql -U u -d postgres -q
php artisan db:show     # proof it works again
```

A one-off "let me just set it to something I remember" is how a working dev environment becomes a five-minute outage with no undo.

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

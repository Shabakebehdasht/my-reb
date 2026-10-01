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

## Divergent Remote Branch (non-fast-forward push)

`git push` rejected with `! [rejected] ... (non-fast-forward)` means the **remote branch of the same name has commits your local branch does not** — typically a parallel session pushed to the shared branch. Before anything destructive, find out what those commits are:

```bash
git fetch origin <branch>
git log --oneline HEAD..origin/<branch>   # remote-only
git log --oneline origin/<branch>..HEAD   # local-only
git diff --stat HEAD origin/<branch>      # do the trees actually differ?
```

Three responses, in order of preference:

1. **Branch fresh and cherry-pick** (see *Isolating a Change Onto Its Own Branch*). Safest: leaves the other session's remote commits intact and produces a provably scoped PR. Confirm the local original still holds your commit afterwards: `git rev-parse <old-branch>`.
2. **Merge the remote branch in** — only when the divergence is small and the content is genuinely disjoint.
3. **Force-push** — last resort. It deletes the other session's commits from the remote. Get explicit user confirmation naming what will be lost.

**Do not `git rebase` onto the divergent remote to "fix" a non-fast-forward.** If the local branch carries many upstream commits, the rebase replays all of them and conflicts on whichever doc or lockfile those commits touched. `git rebase --abort` restores state exactly; the branch you started from is still there.

## Is A Commit Actually On The Base Branch?

`git log <base>` showing a commit, or a commit sitting in your branch's history, does **not** mean the base branch contains it. A commit can reach your branch through a merge from a fork while the base took a different route.

```bash
git merge-base --is-ancestor <sha> <base> && echo YES || echo NO
git log --oneline <base> -- <file>     # what the base really has for this file
```

This is the check that explains "why is this file in my PR". A dependency bump sitting in your branch's history but absent from the base makes the lockfile show up in the PR diff even though you never touched it. When that happens, it is a **decision for the maintainer** — is the bump supposed to be on the base, or did the base deliberately hold the older version? Ask; do not silently "fix" it by reverting your copy or force-pushing.

## The PR Base May Be Ahead Of Your Local Ref

GitHub computes the PR diff against the base branch's **current** SHA. If you fetched a while ago, your local `git diff <base>..HEAD` can name two files while the PR shows three, or vice versa — and neither is wrong.

```bash
gh api repos/<owner>/<repo>/pulls/<n> --jq '.base.sha, .head.sha'
git rev-parse <base>                     # compare against your local ref
git fetch origin <base>:<remote-tracking-ref>   # explicit, updates the tracking ref
gh pr diff <n> --repo <owner>/<repo> --name-only   # what GitHub actually shows
```

Trust the remote's answer over yours when they disagree, and re-check after the next push. Note that `--list` style flags differ by tool: a PR-diff command naming three files right after a push that cleared two is usually this staleness, not a stray edit — fetch and re-read before changing anything.

## Responding To CHANGES_REQUESTED

Verify each criticism against the code before accepting or defending it. Reviewers check cited claims, and agreeing reflexively bakes a wrong fix into the docs; push back with evidence when they are wrong.

```bash
gh pr view <n> --repo <owner>/<repo> --json reviewDecision,reviews
```

For a docs/claims review, every accepted finding should be re-derived from the source, not patched from the reviewer's wording — and state in the reply which claims you confirmed. If a finding's stated *cause* is wrong but its *symptom* is real, fix the symptom and report the real cause separately rather than adopting the wrong explanation.

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

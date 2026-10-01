# Delivering the docs PR and surviving review

Mechanical sequence for getting a docs-only PR out and keeping it alive
through review. The audit findings are covered in SKILL.md; this file is the
branch-and-host dance that follows.

## 1. Verify base sync before editing, and again before pushing

```bash
git fetch origin --prune
git rev-list --left-right --count origin/<base>...HEAD   # left = behind, right = ahead
```

Do this twice. A base that moves mid-session changes the diff under you, and
a file that was clean at session start can appear in the PR uninvited.

## 2. When the working branch will not push

`git push` rejecting with `non-fast-forward` means another session pushed to
the same branch. Before reaching for `--force`, compare the two:

```bash
git fetch origin <branch>
git rev-list --left-right --count origin/<branch>...HEAD   # both counts nonzero = diverged
git diff --stat HEAD origin/<branch>
git merge-base --is-ancestor <base> origin/<branch>         # is remote on the base?
```

If the remote branch is far behind the base and missing whole features, it is
stale, not authoritative. Force-pushing discards another session's work.
Instead branch fresh from the base and carry only your commit:

```bash
git checkout -b <topic-branch> <base>
git cherry-pick <your-commit>
git diff --stat <base>...HEAD      # must show only your files
git push -u origin <topic-branch>
```

This is the default when the divergence is large. Ask before force-pushing
anything you did not create.

## 3. Uninvited files in the diff

A file in the PR that you never edited is not automatically yours to remove —
first find which commit put it there:

```bash
git log --oneline <base>..HEAD -- <file>
```

If it comes from a pre-existing commit, it is not your change; say so in the
PR rather than silently reverting it. Only revert when the file is genuinely
yours or when you have established the base should win. Reverting a
dependency bump into an unrelated branch is how a docs PR becomes a rollback.

## 4. Host-side vs local-side truth

The host's base SHA is authoritative for what the PR diffs against:

```bash
gh api repos/<owner>/<repo>/branches/<base> --jq .commit.sha
gh pr view <n> --json baseRefOid,mergeable
```

When the local remote-tracking ref disagrees, fetch with an explicit refspec
and `--force`. Do not conclude a file belongs in the PR from a stale local
view.

## 5. PR body that gets approved

- Lead with **why the staleness mattered**, not with a list of edits. Concrete
  harm beats volume: "two docs told an agent to write the change that hangs
  the connection" lands; "updated three sections" does not.
- Separate **verified by running** from **read from source** from **static,
  could not verify**. Reviewers check the first category hardest.
- Disclose the awkward parts: a lockfile you did not touch, a base that moved,
  a claim you are deliberately leaving alone and why.
- Fill the repo's own PR template rather than inventing a structure.

## 6. Responding to change requests

Check for review state before concluding anything:

```bash
gh pr view <n> --json reviewDecision,reviews
gh pr view <n> --json reviews -q '.reviews[] | select(.state=="CHANGES_REQUESTED") | .body'
```

`reviewDecision` can be `APPROVED` while an older `CHANGES_REQUESTED` sits in
the list — a later approval supersedes it. Read every entry, not just the
headline.

Then, per point: verify it yourself, fix it, push, and reply with the
evidence. A reply that says "good catch, the migration runs
`CREATE EXTENSION IF NOT EXISTS` itself, so the plain form is correct" closes
the loop; "fixed" does not.

If the reviewer is right and you were wrong, say which of your assumptions
failed. That is the part the next person reading the thread needs.

## 7. After a merge

The branch you opened from may be behind the base again, and a commit you
pushed while the PR was open may or may not be in the merged result. Check
rather than assume:

```bash
gh pr view <n> --json mergedAt,mergeCommit,commits -q '.commits[].oid[0:7]'
git fetch <upstream-url> <base>:refs/remotes/canonical/<base>
git cherry-pick <the-commit>        # if it landed after the merge point
```

Then open a follow-up PR for anything that missed the merge. Delete the
now-empty branches locally and remotely when they have no further purpose.

## Command notes

- Write commit messages and PR bodies to a file and use
  `git commit -F <file>` / `--body-file <file>`. A heredoc in the terminal
  tool can trip the lifecycle guard, and non-ASCII bodies are easier to
  review in a file you can re-read.
- Verify a protected-file write was actually consumed: re-`grep` for the
  corrected phrase, and confirm the phrase you removed is gone from every
  doc file, not just the one you opened.
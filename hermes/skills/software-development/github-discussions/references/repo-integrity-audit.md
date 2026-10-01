# Auditing what a repository actually enforces

Before telling a reader "the gates are green", "the branch is protected", or "CI covers
this", prove it. Three different things get conflated: what CI *ran*, what GitHub
*requires*, and what is already *in the history*. Each needs its own command.

## What CI ran vs what GitHub requires

```bash
# Did CI run on the integration branch, when, and with what verdict?
gh api "repos/OWNER/REPO/actions/runs?branch=BRANCH&per_page=20" \
  --jq '.workflow_runs[] | "\(.created_at) \(.name) \(.head_sha[0:8]) \(.conclusion) event=\(.event)"'

# Any branch protection rules anywhere in the repo (empty array = none)
gh api graphql -f query='{ repository(owner:"O", name:"R") {
  branchProtectionRules(first:10) { nodes { pattern requiresApprovingReviews
    requiresStatusChecks requiredStatusCheckContexts } } } }'

# Per-branch: a 404 here IS the answer "unprotected", not a failed command
gh api repos/O/R/branches/BRANCH/protection
```

A green check run proves a *head commit on a PR branch* passed. Without branch protection and
without a `push` trigger, nothing binds the merge commit that actually landed, so "checks were
green" describes a different object than the code in production. Say which of the two you are
citing, and check both before asserting either.

## Did a workflow stop running on a branch?

A trigger removed months ago reads like live config until you read the file's own history:

```bash
git log --oneline -p --follow -- .github/workflows/NAME.yml   # find the removing commit
git log -1 --format='%h %ad :: %s' --date=format:'%Y-%m-%d' <sha>
git show <sha> -- .github/workflows/NAME.yml
```

Then confirm against the API above, because "the YAML says so" and "runs exist" are
different claims. A workflow triggered only by `pull_request` never validates the integration
branch — not a direct commit to it, and not a merge whose conflict was resolved by hand.

## Duplicate commits on an integration branch

Two branches implementing the same change, each merged into the integration branch without
rebasing, leave two commits with the same subject and the same content. No behaviour is wrong;
`git log`, `git log -S`, and `blame` stay permanently misleading, and a shared doc both
branches edited is why the later merge needed manual resolution.

```bash
# Same subject, different hashes, in one branch's history
git log --format='%h %T %s' origin/BRANCH | sort -k3 | uniq -f2 -d
```

Confirm it is content and not coincident subjects, file by file, by blob identity per path:

```bash
for f in $(git show A --name-only --format=); do
  [ "$(git rev-parse A:$f)" = "$(git rev-parse B:$f)" ] || echo "DIFFERS: $f"
done
```

Then find the mechanism:

```bash
git log -1 --format='%h parent=%p' A
git log -1 --format='%h parent=%p' B
git merge-base --is-ancestor A origin/BRANCH && echo "A is in the history"
git rev-list --ancestry-path --merges A..origin/BRANCH | tail -1
```

Two commits at the same position in two different branches' histories, both present in the
integration branch, with merge subjects naming a remote-tracking merge, is the signature of
one branch merging the integration branch into itself instead of rebasing onto it. The fix
lives upstream of the duplicate commits: look at what landed on the base before opening the PR.

```bash
git fetch origin
git log --oneline origin/BRANCH..HEAD   # my work
git log --oneline HEAD..origin/BRANCH   # what landed while I was working
```

A non-empty second list means rebase or merge the base *first*. A PR opened against a stale
base is what produced the duplicate in the first place, and no amount of rebase-after-the-fact
cleans up the two commits already on the integration branch.

# Status digest from repo history

A "what happened recently" discussion post. The point is a decision thread, not a changelog dump.

## Gather (run all; each number in the post comes from one of these)

```bash
# What landed, per PR, with the curated title
gh pr list -R OWNER/REPO --state merged --limit 20 \
  --json number,title,mergedAt,author --jq '.[] | "\(.number) | \(.mergedAt[0:10]) | \(.author.login) | \(.title)"'

# In-flight work
gh pr list -R OWNER/REPO --state open --json number,title

# Backlog
gh issue list -R OWNER/REPO --state open --json number,title

# Work not yet in a PR, per author (local clone only)
git log --all --no-merges --since=<date> --pretty='%cs %an %s'

# Divergence — only for a ref backed by a configured remote
git remote -v
git rev-list --left-right --count origin/BRANCH...HEAD
```

Prefer `gh pr list --state merged` over parsing `git log --merges` for the narrative: merge subjects are boilerplate ("Merge pull request #N from …") and carry no intent, while a PR title is already the curated summary of its commits.

## Structure

1. **Title carries the window** ("… — <date range>"), so the thread is self-describing in a list view.
2. **Group by theme, not by author and not by date.** A reader asks "what changed in the product". Theme headings are things like quality/stability, domain features, integrations, API contracts, docs. Conventional-commit scopes in the PR titles already tell you the themes — read them off rather than inventing categories.
3. **Inline the PR number as an anchor** after each item. It is a citation, not the content.
4. **Current state section:** open issues (say "none open" explicitly — an empty list IS the status), test counts, branch sync.
5. **One proposed next step**, with a decision or an owner attached. End with a question; a discussion exists to be answered.

## Pitfalls

- Do not restate a PR title you did not read. One-line it, and keep the title's own vocabulary.
- A branch that is in sync is a claim about the push target, which is a specific configured remote — say which one or drop the claim.
- Do not report test-suite counts from memory or from a docs file; run the suite or the count command, otherwise omit the figure.
- Do not list raw commit subjects from `git log --all` for work that already shipped in a PR — that produces duplicate entries under two different headings.

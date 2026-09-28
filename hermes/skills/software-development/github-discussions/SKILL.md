---
name: github-discussions
description: "Use when posting or reading GitHub Discussions."
version: 1.0.0
metadata:
  hermes:
    tags: [github, gh, discussions, digest, status]
    category: software-development
    related_skills: [github]
---

# GitHub Discussions

Discussion threads only. Issues, PRs, reviews, releases, and repo admin belong to the `github` skill — do not duplicate that workflow here.

## When to Use

- Posting a new discussion, replying in a thread, or editing one.
- A user hands you a `github.com/OWNER/REPO/discussions/new` URL and asks "can you write something here" — the capability exists, but the content still has to be asked for (step 3).
- Summarizing recent repo work as a discussion post rather than as a chat reply.
- Reading or triaging existing discussion threads.

## Procedure

1. **Preflight auth and scope.**
   ```bash
   gh auth status
   ```
   The Discussions API needs the `write:discussion` scope; the `repo` scope does not imply it. If the scope is absent, stop and tell the user — do not attempt the post and do not predict that it will work.
2. **List categories before choosing one.** Never guess a slug: slugs are not display names (`Q&A` → `q-a`).
   ```bash
   gh api graphql -f query='{ repository(owner:"OWNER", name:"REPO") { discussionCategories(first:25) { nodes { name slug emoji } } } }'
   ```
   Pass the display **name** to `--category`; both name and slug are accepted.
3. **Clarify the payload when the user supplied only a URL.** "Can you post at this discussions/new link" is a capability question, not content. Offer concrete candidate bodies in ONE multi-choice prompt. Do not draft a body first and do not publish a guess.
4. **Write the body to a scratch file and post non-interactively.** All three of `--title`, `--body-file`, `--category` are required in a non-TTY agent session — omit any one and `gh` falls back to interactive prompts that hang with no terminal.
   ```bash
   gh discussion create -R OWNER/REPO --category "ideas" --title "..." --body-file /path/to/body.md
   ```
5. **Verify with a read-back before reporting success.** The URL `gh` prints is not proof of the category, author, or body.
   ```bash
   gh api graphql -f query='{ repository(owner:"OWNER", name:"REPO") { discussion(number:N) { title url category { name } author { login } body } } }'
   ```
   Check category, author, and that the body is the full text you wrote (compare length, tail).
6. **Reply or edit:** `gh discussion comment -R OWNER/REPO <number> --body-file ...` / `gh discussion edit -R OWNER/REPO <number> --body-file ...`. Same read-back rule.

## Participating in existing threads

"Add your take to each active discussion" is two phases, and the second is where the value is.

1. **Read the whole thread before writing.** Fetch the discussion list *including* `body` and every comment, then read them. Commenting from titles alone produces an answer to a question nobody asked.
2. **Re-verify every load-bearing claim against the code before you assert or refute it.** A thread's own analysis is a claim, not a finding. Grep the cited line, check the flag, confirm the route exists. Other participants are frequently right about the symptom and wrong about the mechanism — that gap is where your contribution lives.
3. **Contribute what the thread lacks, not agreement.** Ranked by value: a fact nobody checked; a mechanism correction ("the failure happens *before* the cache, so a guard inside the cached callback never runs"); a missing constraint ("a config file for flaky tests should be a trait, because a blocklist grows and one day it needlessly halves the suite"); a disagreement with a reason. Pure restatement is noise.
4. **Disagree with evidence, and say which check produced it.** Naming the command that confirmed it is what makes a dissent actionable rather than a matter of taste.
5. **State the residual uncertainty.** If something was not checked this session, say so instead of implying coverage.

One comment per thread, each tailored. If the user said you need no approval to post, that removes the confirmation step — not the verification step.

### Follow-up comments in a thread you already commented in

A second comment in the same thread is judged against your first one. Before writing, list the points your earlier comment already made and rule them out of the new one — restating your own argument back at the thread is the fastest way to make both comments worthless.

The follow-up should be either a *new independent verification* (you re-measured something, found a different fact, or the code moved) or a *narrow correction* to your earlier claim. Address the commenter you are replying to by name, say which of your earlier points you am confirming or retracting, and keep it short — a follow-up that re-litigates the whole thread is noise. If the thread has not moved since your last comment, say that you have nothing new rather than manufacturing a fifth point.

## Writing the content

- **Match the repository's language.** An RTL Persian project gets a Persian post with Persian headings. Keep identifiers, paths, flags, and error strings verbatim in their original script.
- **Always `--body-file`, never `-b '...'`** for anything multi-line. RTL text plus newlines plus shell quoting is a broken combination.
- A digest longer than a screen belongs in a file; do not compress it into a summary of a summary.
- **Write each long body in its own call.** Several multi-line string literals in a single code cell is a syntax-error single point of failure: one bad literal aborts the whole cell, so *none* of the files get written and the batch looks like it ran. One call per file, then a separate call to publish.

## Digest sourced from repo history

Gathering commands, grouping strategy, and the structure that works for a status post: `references/history-digest.md`.

## Pitfalls

- **`gh discussion` is preview.** Its human-readable table and flags shift between releases — never parse its table output; assert through `gh api graphql`.
- **The poster is the token owner, not the repo owner.** `.permissions` from `gh api repos/OWNER/REPO` tells you push rights and predicts nothing about discussion rights; only the token scope does.
- **Stale remote-tracking refs are not remotes.** `git branch -a` and `git log --all` list refs whose remote was deleted, so `git fetch <name>` fails on them. Run `git remote -v` first and compute branch divergence only against refs backed by a configured remote.
- **Never carry a number you did not measure this session.** Test counts, open-issue counts, and "branches are in sync" claims each need their own command in the current session. A figure copied from a docs file is a stale figure; omit it instead of laundering it.
- **Omitting `body` from a GraphQL selection returns empty bodies with no error.** Asking for `number title category` and *not* `body` yields a successful response whose every body is null — which reads exactly like "these posts are empty." If content comes back blank, suspect the selection set before the posts.
- **GraphQL rejects unknown fields loudly; use that.** An invalid field name fails the whole query with a `does not exist on type` error naming the field. Keep selections minimal rather than guessing optional fields — a speculative field costs a round trip and tells you nothing about the data you wanted.
- **Repo settings are checkable, so check them.** "CI gates merges" and "there is a PR template" are both single API calls (`/branches/BRANCH/protection`, a file listing). Asserting either from assumption is the exact failure this skill exists to prevent.
- **`gh discussion comment` is preview like the rest of the porcelain.** For a reply whose URL you must verify, post via the `addDiscussionComment` GraphQL mutation and read the returned `comment { url author }` back — the mutation is stable and gives you the handle in the response.
- **Invisible characters in a prompt payload are a hard failure, not a warning.** ZWNJ (U+200C) is required to render Persian correctly, and any RTL prose pasted into a `cronjob_manage` or subagent `prompt` field is rejected as possible prompt injection. Do not debug the schedule when this is the cause — check the payload first. Write scheduled/standalone job prompts in English, and put the language requirement in as an instruction ("write the body in Persian, and do not use U+200C") rather than embedding sample Persian text. The same applies to any field that carries a prompt across a serialization boundary.
- **A discussion number is not a GraphQL node ID.** `addDiscussionComment` takes the opaque `D_kwDOOL…` global ID; passing the number fails with `Could not resolve to a node with the global id of '123'`. This is independent of the `-f`/`-F` flag choice — both fail on a number. Resolve `{ repository(...) { discussion(number: N) { id } } }` first.

Query and post recipes for multi-thread work: `references/thread-participation.md`.

---
name: agent-docs-audit
description: "Audit AGENTS.md/CLAUDE.md against real code."
version: 1.0.0
author: Rebecca
license: MIT
metadata:
  hermes:
    tags: [agent-docs, documentation, audit, correctness, staleness]
    category: software-development
---

# Agent Instruction Docs Audit

`AGENTS.md` / `CLAUDE.md` are **executable contracts**, not prose. Every sentence is
either a rule a future agent obeys or a trap it falls into. A stale doc is worse than
no doc: it is confidently wrong, so the agent trusts it and produces exactly the bug
the doc once warned about. An audit that only *adds* new features to the file misses
the larger half of the damage.

Trigger: asked whether instruction docs need updating, or after a burst of commits may
have made them stale.

## When To Use

- Asked whether `AGENTS.md` / `CLAUDE.md` / instruction docs need updating.
- A burst of commits may have made the docs stale (new features, renamed classes,
  changed counts).
- A reviewer flagged a claim in an instruction doc as unverifiable or wrong.

Not for auditing end-user documentation or README prose — this is specifically for
docs that an automated agent will follow as a contract.

## Rule

**Every claim is verified against the code, and every claim about existence is
verified by looking.** A doc that names a class, method, flag, or file is asserting it
exists. That is the claim most likely to be false and most expensive to leave in.

## Procedure

1. **Establish the window.** Find when the doc was last touched and diff the commit
   range since. The doc-review note usually carries a date; confirm with
   `git log -1 -- <doc>` rather than trusting it.
2. **Enumerate the window's commits** and group them by theme. Each group is a
   candidate doc section. Use `--stat` on each commit — the changed file list tells you
   whether a commit added a feature, a fix, or only a dependency bump.
3. **Verify the existence claims first** — they are the highest-value findings.
   For each class, method, route, config key, and table named in the doc, grep for it
   and confirm the shape:
   ```bash
   grep -rn 'class Foo\|function bar' app/ | head
   ```
   A method the doc tells an agent to call but that does not exist is the single most
   damaging defect a doc can carry.
4. **Verify the numeric claims** — test counts, file counts, model/factory counts, list
   lengths. Derive each from the source, never from memory or from a previous doc.
5. **Cross-check doc against code for contradictions.** Two doc files disagreeing with
   each other is a common and severe form: an agent that reads only one of them gets
   the wrong contract.
6. **Run the project's own gate** if it is cheap, and record the real result as the new
   verified figure.
7. **Label what you could not verify.** If a browser, binary, or service is absent, say
   so in the doc and mark the number static. Never invent a number to fill the slot.
8. **Write a doc-review note** at the top recording the window, what was added, and what
   was corrected — including the contradiction fixes, which are the point of the review.

## Pitfalls

### Counting tests with grep undercounts

Grepping `test(` and `it(` misses every PHPUnit-style class, which declares tests as
`public function test…` and uses neither wrapper. The result looks plausible and is
silently low. Use the runner's own enumeration and cross-check the sum against a real
run:

```bash
php artisan test <file> --list-tests | grep -c '^\s*-\s'   # per file
php artisan test <a> <b> <c>                                 # sum must match "N passed"
```

A method named `test` inside the file body also inflates a naive grep. If the doc lists
per-file counts, **name the counting method in the doc** so the number is reproducible
rather than a figure someone must trust.

### An empty rule or a name change is a finding

`grep -rn 'EmailNotification' app/` returning nothing means the doc lists a feature
that does not exist. So does a feature list whose items are all transient UI state that
is never persisted. Check what a component actually saves, not what its markup shows.

### Divergence from upstream is not correctness

The most expensive error available in this workflow: your branch and the
upstream base both edit the same file, and you assume **your** copy is the
fixed one because it is the one you have open. Divergence means one side is
stale, nothing more — often the side you are on.

Before writing a fix for a line that differs from upstream, find out which
side changed it and when:

```bash
git log --oneline -S '<the exact string>' -- <file>   # who added/removed it
git log -1 --format=%H <commit>                       # read the message
```

A line you believe is a bug may be the remnant of the original commit, and
upstream may have deliberately removed it. Commit messages and the commit
that introduced the line settle this in one command; guessing does not.
Never resolve it by asserting which side "looks newer".

Note that a stale local fetch makes this worse: the PR's base SHA on the
host can be ahead of your local remote-tracking ref. Compare against the
host's own base, and fetch explicitly:

```bash
gh api repos/<owner>/<repo>/branches/<base> --jq .commit.sha
git fetch <url> <base>:refs/remotes/canonical/<base> --force
```

### Two doc files drift apart silently

A reference file restating a claim the main doc also states will survive a fix to one
of them. When correcting a claim, grep **every** doc file for the same phrase and fix
all copies in the same commit — otherwise the contradiction is still live, now harder to
find.

### Do not rewrite history headers

Doc-review notes from previous passes are an audit trail. Leave the old note's date and
figures alone even after you correct the file's body — a future reader needs to see when
each claim was true. Add a new note instead.

### Writing to the doc may need explicit consent

`AGENTS.md` and similar instruction files can be write-protected by the harness. If a
write is blocked or a consent prompt times out, do not route around it through another
tool. Report what needs changing and ask; a timeout is not consent.

### Scope the PR to your own change

A docs PR should show only docs. If the branch carries unrelated commits, branch fresh
from the base and cherry-pick just the doc commit (see `git-workflow` →
*Isolating a Change Onto Its Own Branch*), and confirm with a diff stat before pushing.
An unrelated lockfile or dependency bump in a docs PR invites a review that ignores the
documentation.

## Writing The Corrections

- State the *mechanism* of each stale claim, so the note teaches rather than just
  reports: "uses `UNION` deliberately because the set operator dedupes and a cycle must
  terminate" — not "changed UNION ALL to UNION".
- Preserve load-bearing explanations the original author wrote. They are the reason the
  rule exists; your job is to add what is missing, not to compress it away.
- Keep traps prominent. A warning that an option key must be passed explicitly, with the
  empty-render symptom it causes, saves the next agent hours.
- Reference the file that owns the contract (`PruneStaleCache::NAMESPACES` is the source
  of truth for namespaces) so the doc points at code that can be re-read.

Branch mechanics, the host-side divergence checks, and how to work through
review comments: `references/delivering-and-reviewing-the-pr.md`.

## Handling review on the doc PR

A reviewer who read your diff against the code is a second pair of eyes, not
an obstacle to route around. Read every comment, including the ones that
praise part of the work — they tell you which claims were checked.

**Verify each requested change before applying it.** A reviewer's claim can
be as wrong as yours. Re-open the file, re-run the grep, and read the call
site. Then:

- If the reviewer is right, say so plainly in your reply, name the evidence
  that settles it, and revert your change.
- If a point is a non-blocking suggestion, accept it and do it anyway, in
  the same PR — a reviewer flagging real staleness in a related line is a
  gift.
- State the direction of the fix. When a code path and a doc disagree, say
  explicitly whether the code should change or the doc should, and why. This
  is the part reviewers most often want and most often have to guess at.

When you revert a claim you were confident about, the reply is the durable
artifact: what the code actually does, and why your reading was wrong.
Naming your own erroring assumption is what stops the next agent repeating
it.

## Settling a disputed claim by experiment

When two sources disagree and reading code is ambiguous, stop arguing and run
the smallest thing that decides it. Create the resource in the plain form,
run the real code path, and observe the result — for a database claim, that
means creating it without the special option and checking what the
migrations actually produce. Then quote the output in the doc or the reply.

This settles disputes that argument cannot, and it is the only acceptable
basis for a "the docs are wrong, the code is right" verdict.

## Verification

- Every existence claim in the doc resolves to code you opened yourself.
- Every number traces to a command you ran, and per-file sums match a real test run.
- `grep` for each corrected phrase across all doc files returns only the corrected form.
- The project's own quality gate passes, and the doc cites that result.
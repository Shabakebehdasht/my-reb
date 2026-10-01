---
name: docs-claims-audit
description: "Audit project docs against recent commits."
---

# Auditing project docs against code

Agent-instruction files (AGENTS.md, CLAUDE.md, references/*.md) are executable
contracts: each line is either a rule or a trap someone already fell into. A
stale line becomes a source of bugs — an agent that trusts it writes code that
hangs a connection or calls a method that does not exist.

## Workflow

1. **Establish the window.** `git log -1 --format=%H -- AGENTS.md`, then
   `git log --oneline $LAST..HEAD`. Review only what landed since.
2. **Check base sync first.** `git fetch origin --prune` and
   `git rev-list --left-right --count origin/<base>...HEAD` (left = behind).
   Reviewing against a stale base invents conflicts that are not real.
3. **Read the doc top to bottom, then verify each factual claim:**
   - counts → count from the files or a real command, never from memory
   - class / method / table / migration names → `grep`, `find`, `ls`
   - behaviour claims → open the source and read it
4. **Run the suite before editing**, so you have a real baseline to cite.
5. **Fix contradictions *between* doc files first.** They are the highest-cost
   errors, because two files tell an agent to make opposite changes.
6. **Label what you could not verify.** If a check cannot run (no browser, no
   service, no credentials), say so in the doc and mark the number static.
   A fabricated count is worse than an acknowledged gap.

## Counting tests — the trap that keeps biting

`grep -c 'test('` and `grep -c 'it('` **undercount**. PHPUnit-style test
classes declare tests as `public function test…`, using neither wrapper, so a
grep reports a fraction of the real number.

```bash
XDEBUG_MODE=off php artisan test tests/Feature/FooTest.php --list-tests | grep -c '^\s*-\s'
```

Cross-check against a real run — the per-file counts must sum to the total:

```bash
XDEBUG_MODE=off php artisan test tests/Feature/A.php tests/Feature/B.php | grep 'Tests:'
```

Naming the counting method in the doc makes the number reproducible instead of
a figure someone has to trust.

## Pitfalls found the hard way

- **A method you name may not exist.** Do not infer a method's *kind*
  ("static convenience dispatcher") from context — grep for it. Both a phantom
  method and a phantom dispatcher were caught in review, in the same PR.
- **"Dispatched from X" requires reading X.** What looks like a job dispatch is
  often a direct synchronous service call. Verify the call site, not the intent.
- **Stale vs. fixed claims.** A doc describing a failure mode ("the example file
  omits APP_LOCALE") may already be fixed upstream. Check the committed file
  before repeating the warning — and keep historical review headers intact, since
  those are a record, not a live claim.
- **Test DB often absent.** `php artisan verify:preflight` prints the resolved
  target plus the exact CREATE DATABASE command. Run it before blaming code.
- **Protected instruction files** need explicit user consent to write, and a
  consent prompt can time out. Ask up front instead of losing the edit.
- **Reviewers check counts against code.** Every number in a docs PR gets
  verified; a wrong count costs a round trip even when the prose is right.

## Delivering the change

- Open the PR from a **fresh branch off the base** when the working branch has
  diverged. Never force-push over another session's commits.
- Put the *why* in the PR body, and state which claims were verified by running
  something versus read from the source.
- Separate unrelated pre-existing changes (lockfile bumps, merges) out of a
  docs PR so the diff stays reviewable.
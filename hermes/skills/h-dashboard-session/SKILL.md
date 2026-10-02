---
name: h-dashboard-session
description: "Use when working in h-dashboard repo."
version: 1.0.0
author: Hermes Agent
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [h-dashboard, laravel, bootstrap, codegraph, git, session-start]
---

# h-dashboard Session Bootstrap

## When to Use
Any session touching `/home/runner/h-dashboard`: new feature, bug fix, refactor, test run,
or a plain structural question about the code. Run sections 1-5 before real work.

Repo: `/home/runner/h-dashboard` (also the configured Hermes `terminal.cwd`).
Branch: `rebecca`. Canonical upstream: `asgarimehdi/h-dashboard`, branch `beta`.
Only remote is `origin` (this server's fork); **never add, rename, or delete remotes.**

## 1. Never
- Do not clone a second copy of the repo.
- Do not switch branches unless asked.
- Do not push dev work straight to `beta`.
- Do not hardcode a fork URL or branch name — read `git remote -v` / `git branch --show-current`.

## 2. Session start sequence
```bash
cd /home/runner/h-dashboard
git remote -v
git branch --show-current
git status
git fetch origin
git merge origin/beta     # resolve conflicts; never rebase published work
```
Push only after step 4 verification passes.

## 3. Read project rules first
Root `AGENTS.md` is authoritative — read before touching code and keep it in context.
It already documents CodeGraph, Boost/Context7/GitHub MCP, Pest, Playwright, conventions,
and the gotchas table. Do not re-derive them.

## 4. Verify tooling, never assume
```bash
codegraph status .        # must be up to date; else `codegraph sync`
hermes mcp list           # codegraph, laravel_boost, context7, github enabled
```
Fix anything broken before continuing. Never claim a tool was used when it wasn't.

## 5. CodeGraph first for structural questions
Before grep/glob/Read crawling to understand structure, call CodeGraph MCP
(`mcp__codegraph__codegraph_explore`) or the CLI:
```bash
codegraph explore "how does AccessService accessibleUnitIds resolve unit hierarchy"
codegraph query "HardwareAuditObserver" --limit 5
```

## 6. Before committing
```bash
git status
git diff
vendor/bin/pint --test
composer test        # pest
```
Clear commit message, then `git push origin <current-branch>`.

## 7. Pull requests
When the user says `pr`, open a PR from the current branch to `beta` of
`https://github.com/asgarimehdi/h-dashboard` via GitHub MCP. **Do not merge** unless asked.

## 8. Superpowers skills are mandatory
`superpowers` plugin is enabled (15 skills). Load the process skill before acting:
brainstorming before plan mode, systematic-debugging for bugs, test-driven-development
before writing behavior, verification-before-completion before claiming done.
Invoke as `skill_view("superpowers:brainstorming")` etc.

## 9. Docs and audit skills
- `read-the-damn-docs` (software-development/read-the-damn-docs) — read official/current
  docs before implementing against any third-party API or library. Context7 MCP for
  version-specific Laravel/PHP docs.
- `improve` — shadcn audit skill, audit-only, do NOT execute its findings unless asked.
---
author: Hermes Agent (session-derived)
license: MIT
---

# Reading and replying across many threads

## Fetch threads with bodies and comments

One query, minimal selection. Both `body` fields are required — omit either and you get nulls with a success status.

```bash
gh api graphql -f query='{
  repository(owner:"OWNER", name:"REPO") {
    discussions(first: 50, orderBy: {field: UPDATED_AT, direction: DESC}) {
      nodes {
        number title body
        category { name }
        author { login }
        createdAt
        comments(first: 30) { totalCount nodes { author { login } body createdAt } }
      }
    }
  }
}'
```

Notes:
- Do not add speculative fields (`isLocked`, `state`, …) to be safe. An unknown field fails the whole query by name, which costs a round trip and teaches you nothing about the data.
- `UPDATED_AT` ordering surfaces the threads that actually have traffic; `CREATED_AT` ordering buries them.
- Pipe through a JSON parser, not a substring grep — the bodies contain the characters you would grep for.
- Dump the parsed result to a scratch file when the set is large, then read it in slices, so a failure partway through does not lose the whole fetch.

## Post a reply

`discussionId` must be the **opaque global node ID** (`D_kwDOOL…`), never the discussion number. Fetch it first, then pass it as a string variable:

```bash
gh api graphql -f query='{ repository(owner:"OWNER", name:"REPO") { discussion(number: 123) { id } } }'
# → { "data": { "repository": { "discussion": { "id": "D_kwDOOLHV784Apm-N" } } } }

gh api graphql \
  -f query='mutation($discussionId: ID!, $body: String!) {
    addDiscussionComment(input: {discussionId: $discussionId, body: $body}) {
      comment { id url author { login } createdAt }
    }
  }' \
  -f discussionId=D_kwDOOLHV784Apm-N \
  -F body=@/path/to/comment.md
```

- `-f discussionId=D_kwDOOL…` — `-f` sends a plain string, which is what an `ID!` variable needs.
- `-F body=@file` — `-F` with `@` is what reads the file into the `String!` variable.
- **Passing the number fails, and the flag choice is not what saves you.** `-F discussionId=123` and `-f discussionId=123` both return `NOT_FOUND: Could not resolve to a node with the global id of '123'`, because a discussion's global ID is not its number. Resolve the node ID first; the number is only an input to the lookup.

Use the `url` in the mutation's return value as the delivery handle, and read it back before reporting the post as done.

Per-thread batch:

1. Write body 1 to its own scratch file (own call).
2. Post it, assert the response contains a `url` for that discussion number.
3. Continue. Do not batch all writes into one script — a syntax error or a consent timeout in a single cell then loses the entire batch with nothing published.

## What earns a comment

Ranked, strongest first:

| Contribution | Example shape |
|---|---|
| Unverified fact | "Checked this directly: `<endpoint>` returns 404, so the gate is inert today." |
| Mechanism correction | "The failure happens *before* the cache — the cache key is built outside the cached callback — so a guard inside the callback never runs." |
| Missing constraint | "Hooks are per-machine and skippable; CI has to stay the authoritative gate." |
| Disagreement + reason | "I'd order these the other way: this work writes to new tables only, so its regression risk against running jobs is zero." |
| Restatement / "I agree" | Cut it. The thread already says it. |

Before asserting a mechanism, find *where* the code runs relative to the thing you think guards it. Ordering is invisible in a bug report and decides the fix.

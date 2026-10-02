---
name: "feature-clarify"
description: "Clarify a vague feature idea before building it."
version: "1.0.0"
author: "Mehdi (adapted from github/spec-kit speckit-clarify, MIT)"
license: "MIT"
metadata:
  source: "Adapted from github/spec-kit speckit-clarify (MIT); standalone version for Muse and portable to Hermes (~/.hermes/skills/)"
  hermes:
    tags: ["spec-driven-development", "requirements", "planning", "questions"]
    related_skills: ["superpowers:brainstorming", "improve"]
user-invocable: true
triggers: "clarify this feature, ask me questions before starting, شفافش کن, قبل از شروع بپرس, vague or one-line feature idea whose unknowns must be resolved first. Standalone: does NOT require Spec Kit, a .specify/ directory, or any CLI."
---

# Feature Clarify

## When to Use
- The user hands over a vague, one-line, or half-specified feature idea and
  wants unknowns resolved before planning/coding.
- Trigger phrases: "clarify this feature", "ask me questions before starting",
  "شفافش کن", "قبل از شروع بپرس".
- Input may be an idea, a draft GitHub issue, an existing spec/Plan doc, or a
  verbal description — clarify whichever it is.
- Do NOT use for: pure bug reports (use systematic-debugging), pure style
  preferences, or when the user explicitly wants to skip straight to code.
- Standalone: no Spec Kit, no `.specify/` directory, no CLI needed.

Interrogate a feature idea before any implementation starts: scan it for
ambiguity across a fixed taxonomy, ask targeted clarification questions one at
a time (max 5 per round, each with a recommended answer), integrate every
answer immediately, and finish with a clarified feature brief ready to become a
spec, a Plan-format GitHub issue, or an implementation plan.

## Purpose
Reduce downstream rework by resolving the ambiguities that actually change
implementation, tests, or acceptance — *before* planning or coding starts.
This is the clarification stage of Spec-Driven Development, extracted so it
can be used on its own, on any input: a one-line idea, a draft GitHub issue,
an existing spec/Plan document, or a verbal description.

## Workflow

### 1. Load the input
- Take the feature description from the user's message, the file they point
  to, or the issue/spec already under discussion.
- If the input is an existing document, clarify *that* document and update it
  in place (step 4). If it is just an idea, the output is a new brief (step 5).
- If the user names focus areas, weight the scan toward them.

### 2. Ambiguity scan (internal — do not dump the raw map on the user)
Mark each category Clear / Partial / Missing:

1. **Functional scope** — core goal, success criteria, explicit out-of-scope
2. **Domain & data** — entities, attributes, identity/uniqueness, lifecycle
3. **Interaction flow** — main journeys, error/empty/loading states
4. **Non-functional** — performance, scale, reliability, observability,
   security/privacy, compliance (only those relevant to this feature)
5. **Integrations** — external services/APIs, failure modes, import/export formats
6. **Edge cases & failure handling** — negative scenarios, conflicts, limits
7. **Constraints & tradeoffs** — stack/hosting limits, rejected alternatives
8. **Terminology** — ambiguous words, vague adjectives ("fast", "simple",
   "robust") lacking a measurable meaning
9. **Completion signals** — testable acceptance criteria, definition of done

Skip categories that do not apply to the feature's kind (e.g. UX states for a
backend job). Note implementation-method questions internally but do not ask
them unless the answer blocks functional clarity.

### 3. Question loop — one at a time, max 5 per round
Build a prioritized queue (highest Impact × Uncertainty first), but:
- Present **exactly one question at a time**. Never reveal the queue.
- Each question is a full interrogative sentence in plain language, followed
  by one short "why it matters" sentence.
- Answerable in one tap whenever possible:
  - Multiple choice (2–5 mutually exclusive options) with a clearly marked
    **recommended** option and 1–2 sentences of reasoning, or
  - A short answer (≤5 words) with a **suggested** answer.
  - Use the options widget (clarify tool) for multiple-choice questions.
  - The user can accept the recommendation, pick an option, or answer freely;
    if the reply is ambiguous, disambiguate within the same question.
- Stop when: all critical ambiguities are resolved, the user says
  "done / بسّه / proceed", or 5 questions were asked.
- If nothing is worth asking, say so plainly ("No critical ambiguities
  detected") and skip to the brief.
- If high-impact categories remain after 5, list them as **Deferred** in the
  summary; the user can say "continue" to open another round of up to 5.

### 4. Integrate each answer immediately
- Record one line per answer: `Q: <question> → A: <answer>`.
- If clarifying an existing document: append the line under a
  `## Clarifications` section (create it if missing, dated subheading per
  session), apply the answer to the relevant section of the document,
  replace — never duplicate — any statement it invalidates, and save after
  each answer.
- If clarifying a bare idea: keep the running list in the conversation;
  it becomes part of the brief in step 5.

### 5. Completion report + clarified brief
Report: questions asked/answered, categories Resolved / Clear / Deferred /
Outstanding (compact list), and produce the brief:

```markdown
# <Feature name> — Clarified Brief
## Goal (what & why)
## Scope — in / explicitly out
## Behaviour & acceptance scenarios (Given/When/Then for the main flows)
## Data & entities
## Edge cases & failure behaviour
## Constraints & non-functional targets (measurable, no vague adjectives)
## Clarifications (Q → A list)
## Deferred / still open (with why each was deferred)
```

The brief must be directly usable as the body of a Plan-format GitHub issue
or as a spec.md — no further rewriting needed.

## Operating Rules
- Language: ask in the user's language (for Mehdi: Persian; keep technical
  terms in English).
- Max 5 questions per round; quality over quantity — every question must be
  one whose answer changes implementation, tests, or acceptance.
- Always offer a recommendation/suggestion; make accepting it a one-word or
  one-tap action.
- Never ask about pure style preferences or plan-level execution details
  unless they block correctness.
- Respect early stop signals immediately; still deliver the brief with the
  answers gathered so far and mark the rest Deferred.
- Do not start planning or implementing inside this skill — hand off the
  brief and let the user decide the next step.

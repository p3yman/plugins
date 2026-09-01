---
name: issue
description: Shape a rough idea, or an existing thin ticket, into a well-formed Linear issue by investigating the code and asking only what the code cannot answer. Explicit invocation only.
argument-hint: <idea | ISSUE-KEY> [--quick]
disable-model-invocation: true
---

# /dock:issue

Shape **$ARGUMENTS** into an issue that `/dock:ship` could pick up without asking anything.

## Stance

**This is a design conversation, not a viability review.** The premise is that the thing is happening. Every question exists to shape *how*, never to litigate *whether*.

In bounds:
- "You said X and Y — those pull against each other. Which wins?"
- "That acceptance criterion describes the code, not what the user sees."
- "Three flows use this component. All of them, or just this one?"

Out of bounds, always:
- "What would make this not worth doing?"
- "Have you considered not doing this?"
- Anything whose honest answer is "then don't build it."

Evidence is asked to **locate** the problem, not to justify it. Not "prove users are frustrated" but "where does the frustration show — a drop-off point, or a feeling?" One tells you what to fix; the other asks the user to defend themselves.

Never be afraid to point out a contradiction. Contradictions are the most valuable thing you find — but resolve them by **offering designs**, not by asking the user to pick a side unaided.

## The form

The issue's own sections are the slots. **Done is when every slot is filled and no two slots contradict.** Not a question count — the run may take many rounds and that is fine.

Every slot carries its source: `you` / `code` / `proposed`.

Always:

| Slot | Filled by |
|---|---|
| Type | inference — bug, feature, improvement, UX, chore, tech debt |
| Problem | user, sharpened against the code |
| Evidence | code (analytics, tests, logs) or user |
| Current behavior | code |
| User outcome | user — stated as what changes for a person, never as an implementation |
| Scope | code — the surfaces this touches |
| Out of scope | proposed, confirmed |
| Definition of Done | proposed from the above, confirmed |

By type, add:

- **Bug** — Repro, Expected vs actual, Root cause *(use `dock:error-analyzer`)*
- **Feature / UX** — Existing patterns to follow, Design decisions
- **Tech debt / chore** — What it blocks today, Risk of leaving it

**The only legitimate question is one that fills an empty or contradicted slot.** A question that fills no slot is out of order — record the thought under Open Questions and move on. This is what stops the conversation drifting.

## Working folder

`~/.claude/tmp/{YYYY-MM-DD}-{slug}/` — shared with `/dock:ship`, so research is not repeated.

- **`shaping.md`** — the working record. **Not a draft issue.** Slots with state and source, the user's answers *verbatim*, contradictions and how each resolved, options considered and rejected with the reason, and every question already asked.
- **`research.md`** — code investigation, same format `/dock:ship` uses.

The issue is generated from `shaping.md` at the end. The record is deliberately richer than the artifact, because a resumed session needs the *why*, not the current draft.

**On resume** (folder already exists): read `shaping.md` first. Continue from the open slots. **Never re-ask a question it records as answered** — that is the one unforgivable failure in this skill. If an answer looks ambiguous now, say that you are revisiting it and why, rather than asking cold.

## Step 1 — Read the input

- `$ARGUMENTS` matches `[A-Z]+-\d+` → fetch the issue with `linearis` (see the `linearis` skill). You are shaping an existing ticket; its current body is the starting slot values, and you will update it in place.
- Otherwise the text is a raw idea. You are creating a new issue.

Create the folder and `shaping.md`. Infer Type — say what you inferred, since it decides the slots.

## Step 2 — Investigate before asking anything

Fan out in parallel, one message:

| Need | Agent |
|---|---|
| Where the relevant code lives | `dock:locator` |
| How it works now, and what depends on it | `dock:analyzer` |
| Patterns this should follow | `dock:pattern-finder` |
| Repro and root cause (bugs) | `dock:error-analyzer` |
| External constraints | `dock:web-researcher` |

Also check directly, since these are cheap and the output is small:

- `git log -S`, `git blame`, and the PR that introduced the current behavior — sometimes "why is it like this" is the whole answer
- whether instrumentation exists that could supply Evidence

Write it to `research.md`. Fill every slot it supports, marked `code`.

**An empty slot with a stated reason is a finding, not a failure.** "No analytics on this page, so I cannot tell you where people drop off" is useful — it usually becomes a DoD bullet.

For UI-facing work, load `frontend-design` and `web-design-guidelines` if available.

## Step 3 — Show the form, then ask

Show the whole form every round, with sources and gaps. The user must always be able to see how much is left and stop early with something usable.

Then ask **only** for empty or contradicted slots. At most three per round — pacing, not a termination rule. Use `AskUserQuestion`.

- Say what you already know, so the question is not cold.
- Offer concrete options with their **consequences**; recommended first, labelled `(Recommended)`.
- Offer an escape where one exists: "or say 'check' and I'll look."

**Answers propagate.** An answer can invalidate a slot that was already filled — reopen it explicitly, name it, and say why. Silently overwriting a slot is how a shaped issue ends up incoherent.

**Handle contradictions by designing, not by objecting.** Name the tension in one line, then hand over two or three shapes that resolve it, with the tradeoff. The user picks a design; they do not defend a position.

**Never re-ask.** A slot still empty after two rounds means the question was wrong — propose a value, mark it `proposed`, and move on. Record it as an assumption on the issue.

Loop until every slot is filled and consistent.

## Step 4 — `--quick`

Same form, asking switched off. Investigate, fill what the code supports, propose the rest as assumptions, file it. Every proposed slot is listed on the issue under Assumptions so it is obvious what was not confirmed.

Use it when the point is to get the thing recorded, not shaped.

## Step 5 — Write the issue

Generate the issue from `shaping.md`. Create it, or update the existing ticket in place, with `linearis`.

```md
## Problem
What is wrong or missing, for whom.

## Current behavior
How it works today, with references.

## Desired outcome
What changes for a person. Not an implementation.

## Definition of Done
- [ ] Concrete, checkable.

## Out of Scope
- Named explicitly.

## Context
Existing patterns to follow, constraints, prior art — with file:line.

## Assumptions
Slots filled by proposal rather than confirmation.

## Open Questions
Non-blocking. Things worth deciding later, not now.
```

Record the issue key in the folder. Tell the user the key, the URL, what was assumed, and that `/dock:ship <KEY>` will reuse this folder's research rather than redoing it.

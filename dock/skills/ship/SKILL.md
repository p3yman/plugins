---
name: ship
description: Drive one Linear issue or one task prompt from research through plan, validation, implementation and review to a single PR. Explicit invocation only.
argument-hint: <ISSUE-KEY | task description> [--ask] [--no-worktree]
disable-model-invocation: true
---

# /dock:ship

Deliver **$ARGUMENTS** as one PR.

You own the pipeline **and you write the code**. Subagents exist for work that reads far more than it writes, and for the two independent-angle passes. The pipeline runs straight through to the PR; only three things stop it — the `--ask` checkpoint, a genuine blocker, or a budget ceiling.

## Task folder

`~/.claude/tmp/{YYYY-MM-DD}-{slug}/` — the ledger. Three files, from this skill's `templates/`:

- **`task.md`** — goal, DoD, out of scope, and the run manifest (base branch, worktree, verify commands, PR URL, issue key)
- **`research.md`** — written by subagents, read by everything downstream
- **`plan.md`** — phases with checkboxes. **This is the state machine.**

Linear, when there is an issue, is a mirror: status moves plus one comment with the PR link. It is never the source of truth.

## Delegation rule

**Delegate work that reads much more than it writes.**

| Work | Who |
|---|---|
| Locating, analyzing, pattern-finding, web research, repo recon | subagent — reads 40 files, returns a page |
| Plan validation, final review | subagent — the independent angle *is* the value |
| Writing the PR | subagent — mechanical |
| **Implementing a planned phase** | **you, in this thread** |

You built the plan and hold the context. Handing a phase to a subagent is a lossy re-serialization of what you already know, and it takes the user out of the loop for the twenty minutes they most want to be in it.

Delegate a phase only when: the plan marks phases independent with disjoint files and they can run in parallel; it is mechanical grind across many files; or context is genuinely running hot. If a phase needs a large exploratory read first, **delegate the read** (`dock:analyzer`) and implement in-thread.

While implementing, reach for `dock:analyzer` on an unfamiliar path, `dock:pattern-finder` before writing code that should match a local convention, `dock:error-analyzer` when a check fails cryptically.

## Step 0 — Resume or clean up

If a folder for this task already exists under `~/.claude/tmp/`, this is a resume. Read `task.md` for the manifest, then `plan.md`.

- A PR is recorded → `gh pr view <n> --json state,mergedAt`.
  - **Merged** → move Linear to done, post the closing comment, `ExitWorktree({action:"keep"})` if inside, remove the worktree and the branch if fully merged, archive the folder to `~/.claude/tmp/_done/`, tell the user, stop.
  - **Open** → continue from the first unchecked phase.
- **Reconcile before trusting a checkbox.** An unchecked phase whose work is already committed means a prior run died before ticking it. Check `git log --oneline {base}..HEAD` and the diff against that phase's Change Surface. If the work is there, tick the box, note in `Notes` that it was reconstructed after the fact, and move on. Never re-implement committed work — duplicating or reverting delivered work is far worse than a missing tick.

Otherwise this is a fresh run. Continue.

## Step 1 — Frame the task

Slug from the issue title or the prompt. Create the folder; write `task.md` from `templates/task.md`.

- `$ARGUMENTS` matches `[A-Z]+-\d+` → fetch the issue with `linearis` (see the `linearis` skill for commands). Body verbatim into `task.md`, its acceptance criteria as the DoD.
- Otherwise the prompt *is* the task; write goal and DoD yourself.

**The issue is assumed ready. Do not interview the user about it.** But `task.md` must end up with a concrete, checkable DoD. If the input is too thin to write one, that is a real blocker — ask, then continue.

In parallel, spawn **`dock:scout`** to fill in the manifest: base branch, the fast diff-scoped check, the full gate, lint/format commands, and whether `AGENTS.md`/`CLAUDE.md` carry rules worth following. Write its answer into `task.md` so nothing downstream rediscovers it.

## Step 2 — Worktree

Default on. Skip on `--no-worktree`, if the user says so, or if this is not a git repo.

- Already inside the right one → continue.
- Listed in `git worktree list` → `EnterWorktree({path})`.
- Remote branch or PR exists → `git fetch origin <branch>`, `git worktree add`, enter.
- Neither → `EnterWorktree({name: "<slug>"})` off the base branch. This is when Linear moves to "in progress".
- Inside a *different* worktree → `ExitWorktree({action:"keep"})` first, then retry.

**Dependencies.** If the lockfile matches an already-installed checkout, `node_modules` may be symlinked — Node resolution doesn't care where the symlink lives. **Never symlink a Composer `vendor/`:** the generated `vendor/composer/autoload_*.php` bakes in the absolute path of the checkout it was installed from, so a symlinked vendor silently autoloads the *other* checkout's classes and PHP tests go green against code that isn't on this branch. A lockfile mismatch always means a real install.

Before running the project's gate for the first time in a new worktree, read the gate script for worktree-specific footguns — a fixed Docker `container_name` colliding with another checkout is the common one.

**Stop and tell the user if:** there are uncommitted changes that aren't leftovers from a prior run on this task, or the base branch has diverged in a way that doesn't rebase cleanly.

## Step 3 — Research

**Inspect the repo before asking the user anything.**

Skip research only when the task is a single obvious phase in code `task.md` already names. It earns its cost at three or more phases, or whenever the same context would otherwise be read twice.

Fan out in parallel — one message, one scoped brief each:

| Need | Agent |
|---|---|
| Where things live | `dock:locator` |
| How the current code works | `dock:analyzer` |
| Local conventions to follow | `dock:pattern-finder` |
| External API / library behavior | `dock:web-researcher` |
| Repro and root cause (bug tasks) | `dock:error-analyzer` |

For a bug, `dock:error-analyzer` replaces the locate/analyze pair — the research *is* the root cause.

Collect the results into `research.md` from `templates/research.md`. Keep the file:line references; they are what make the plan implementable without re-exploration.

## Step 4 — Plan

Write `plan.md` from `templates/plan.md`, referencing `research.md`.

**Decision-complete where repo evidence supports a choice.** Do not leave implementation approach, file ownership, verification, or commit boundaries to be invented later. Record only true unknowns as questions.

- Every task must be concretely checkable. "Improve the UX" is a decision or a blocking question — never a task.
- Do not invent exact line numbers research didn't find.
- Size every phase (XS–XL). Size feeds the delegation rule in Step 7.
- **Out of Scope is not optional.** It is the scope-creep guard.

For UI-facing work, load the `frontend-design` and `web-design-guidelines` skills before writing the plan, if they are available.

## Step 5 — Validate the plan

Spawn **`dock:plan-validator`** with the paths to `task.md`, `research.md`, `plan.md`, and the base branch. It challenges and reports; it does not edit.

**You apply the fixes yourself.** Editing the plan is cheap text work and you own the document.

Skip validation only when the plan has two or fewer phases and no open questions.

## Step 6 — Checkpoint (`--ask` only)

Only questions that survived **both** the plan pass and validation reach the user. Two models already failed to settle them from repo evidence — that is the bar.

**If nothing is unresolved, show nothing and continue.** This is not an approval gate.

Use `AskUserQuestion`, at most three at a time. Each question names the phase it affects and the evidence that failed to settle it. Each option describes the **consequence**, not the label. Recommended option first, labelled `(Recommended)`.

Never ask: "should I proceed", anything the codebase answers, anything validation already resolved, or naming and style the repo already settles.

Answers go into that phase's `Implementation Decisions`. Record `Checkpoint: passed` in `plan.md` so a resume doesn't re-ask.

**Without `--ask`:** decide from best evidence, write the decision *and its reasoning* into `Implementation Decisions`, and surface it as an explicit assumption in the final report and the PR body. Nothing is silently guessed in either mode.

The exception holds in both modes: if proceeding either way would produce work that is wrong if the guess is wrong, stop and ask.

## Step 7 — Implement

Per phase, in plan order — parallel only where the plan says the phases are independent with disjoint files:

1. Implement it. You, in this thread, unless the delegation rule says otherwise.
2. Run that phase's own verification from the plan.
3. Commit. Prefix `<ISSUE-KEY>: ` when there is an issue.
4. Tick the phase checkbox and append a `Notes` block under the phase — what changed, decisions made, gotchas hit.
5. Next phase.

The `Notes` trail is how a resumed run, or a delegated agent on a later phase, learns what earlier phases did. Write it even when it feels obvious.

Run the fast diff-scoped check from the manifest continuously. The full gate waits for Step 9.

### Delegated phases

Same rules, plus: **the prompt is that agent's entire world** — it cannot delegate further. Give it the phase section verbatim, paths to `research.md` and `task.md`, the verify command, the commit convention, and the resume contract:

> Before reading the codebase, run `git status --short`, `git log --oneline {base}..HEAD`, and `git diff`. If work is already in progress, continue it — read only the files the diff touches, verify them against the phase criteria, and finish. Do not re-derive the plan and do not restart from a cleaner base.

It updates only its own checkbox, never reverts another phase's work, and reports files changed, verification run, and any blocker.

If a delegated agent had to go exploring, the plan was underspecified. Say so in the report — it is a signal, not normal operation.

## Step 8 — Review

Spawn **`dock:reviewer`** on the accumulated diff against the DoD, **before the PR exists**. A review after the PR opens produces fixup commits on something a human may already be reading.

Give it: `{base}..HEAD` (or the diff), the DoD from `task.md`, and `plan.md`. It reports; it does not edit.

Fix what it finds. A finding in code a delegated agent wrote goes back **to that agent** via `SendMessage` — it still holds the context. Spawn fresh only when the finding names a file this run never touched; an agent with no working memory will spend its whole budget re-reading the diff and produce nothing.

**An unmet DoD bullet is never a follow-up.** Either fix it, or state it as unmet with the reason in the report and the PR body.

## Step 9 — Gate and push

Run the full gate from the manifest **once**, then push. Record the SHA it passed against, so a resumed run doesn't repeat it.

**Stop at green.** Gate passes, branch pushed, PR reflects it → the run is over. No further subagents, no second look, no re-verification of a diff that already passed, no re-running the gate against unmoved HEAD. A doubt that surfaces after green goes into the PR body as an open question for the reviewer. Everything after green belongs to the reviewer, not to this run.

## Step 10 — PR

Spawn **`dock:pr`** with base branch, head branch, the DoD, a summary of what shipped, the assumptions recorded in Step 6, any deviations, and verification results. It writes and opens the PR and returns the URL.

**Never merge it.** Record the URL in `task.md`. Move Linear to its review/preview state.

## Step 11 — Report and stop

`ExitWorktree({action:"keep"})`. **Keep the worktree and the task folder** — the PR isn't merged, and review comments are coming.

Tell the user: PR link, phases completed, assumptions made, anything left unmet, how many subagents ran and roughly how long it took, and which ceiling stopped the run if one did. Note that merging the PR and re-running `/dock:ship` moves the issue to done and cleans up.

Spawn count and elapsed time are the only signal the user gets that a plan was mis-sized. Report them plainly.

## Budget guard

Hitting **any** of these ends the run with a report. None is satisfied by "but I'm nearly done":

- **Phases** — running past roughly double the plan's phase count. Re-plan; don't push through.
- **Spawns** — more than one per delegated phase (a second is a resume, a third is thrash), plus research, validation, review, and PR.
- **Wall clock** — ~90 minutes from Step 1. Check it at three points: before the review pass, before the full gate, and after every round-trip chasing a gate failure. That last one is the one that actually gets skipped, because a regression hunt always feels close to done.
- **Green** — Step 9. No exceptions at all.

A ceiling ends the run cleanly with work pushed. It is not a blocker and it does not become a question.

Break early **for a blocker** only when a human decision is genuinely required: requirement ambiguity that would change the approach, a non-trivial merge conflict, or a gate failure you could not fix after a real attempt. Unfamiliar is not a blocker — that is what the research agents are for.

## Linear (only when there is an issue)

Workflow-state names vary by team — never hardcode one. Once per run, list the team's states with `linearis` and match case-insensitively by substring:

- **work started** → "in progress", else any started-type state that isn't review or preview
- **PR opened** → "preview", else "review", else leave it alone
- **PR merged** → "done", else the first completed-type state

Only ever move an issue forward. Post **one** comment when the PR opens — link plus summary. Not a running log; `plan.md` is the log.

Work discovered mid-run that is separately decidable becomes its own Linear issue with a parent, linked from that comment. Not a sentence saying it "wants its own issue."

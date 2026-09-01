# dock

A Claude Code plugin. Shapes an idea into an issue, takes an issue through research,
planning, implementation and review to one PR, explains the code you have to work in, and
sweeps for the problems scoped work never surfaces, and keeps dependencies moving — all off
one shared pool of agents.

## Install

```
claude plugin marketplace add p3yman/plugins
claude plugin install dock@p3yman
```

## Skills

### `/dock:issue <idea | ISSUE-KEY> [--quick]`

Shapes a rough idea, or an existing one-line ticket, into an issue `/dock:ship` can pick up
without asking anything.

It is a **design conversation, not a viability review** — the premise is that the thing is
happening, so every question exists to shape *how*, never to litigate *whether*. The issue's
own sections are the slots, and it is done when every slot is filled and none contradicts
another. Not a question count.

It investigates the code before asking anything and only asks what the code cannot answer,
shows the whole form each round so you can stop early with something usable, resolves
contradictions by offering designs rather than asking you to defend a position, and never
re-asks something you already answered — a slot still empty after two rounds gets a proposed
default and is recorded as an assumption.

`--quick` is the same form with asking switched off: investigate, fill what the code
supports, list the rest as assumptions, file it.

Work lands in the same `~/.claude/tmp/{date}-{slug}/` folder `/dock:ship` uses, so shipping
the issue afterwards reuses the research instead of redoing it.

### `/dock:ship <ISSUE-KEY | task description> [--ask] [--quick] [--no-worktree]`

Runs the whole pipeline in one invocation:

```
resume check → task.md → worktree → research → plan
             → validate plan → [--ask] → implement → review → gate → PR
```

Work lives in `~/.claude/tmp/{date}-{slug}/` as three files: `task.md` (goal, DoD, and the
repo's verify commands), `research.md`, and `plan.md`. The checkboxes in `plan.md` are the
state machine, so re-running `/dock:ship` picks up wherever the last run stopped. Once the
PR is merged, the next run cleans up the worktree and archives the folder.

A Linear issue is optional. Given one, dock mirrors status and posts the PR link; given a
prompt, `task.md` is the only record.

`--ask` surfaces questions that neither the planner nor the validator could settle from the
code, at the one point where answering them is still cheap — after the plan exists, before
any code does. If nothing is unresolved it asks nothing.

`--quick` is for a task you have already judged small and clear: one obvious change in code the
task names. It skips research, `plan.md` and plan validation, and goes straight from `task.md`
to implementation, keeping the independent review, the gate and the PR. If the change turns out
to be bigger than one or two commits, the run stops with the verified work committed and tells
you to re-run without the flag, which resumes as a full run on the same folder.

### `/dock:explain <path | symbol | question>`

Fans agents out in parallel over an unfamiliar subsystem and returns one answer: how it
works, what depends on it, why it is like that, and where to start reading.

It picks a depth first and tells you which — a specific function gets read directly, because
spawning three agents to explain a helper is worse than just opening it. Only "why is this
like this" questions pay for the history pass.

### `/dock:audit [what to look for] [where]`

Finds the problems that scoped work never flags *because they are correctly out of scope* —
a PR review ignores everything outside its diff, and `/dock:issue` parks tangents under Open
Questions. This is the pass that looks at the whole.

Five dimensions, fanned out in parallel: security, data access, architecture, test value,
dead code. Ask it whenever it feels like time — there is no schedule and no due date, and it
does not track when it last ran. Dependencies are not a dimension; `/dock:upgrade` covers
that ground properly.

**It writes nothing to the repo.** A stale audit record is worse than none, because a later
agent reads it as current truth. The report is ephemeral and the only durable output is a
Linear issue. Memory lives in Linear instead: findings already covered by an open issue are
listed as tracked rather than re-reported, and findings you didn't file are supposed to come
back next time.

Every finding has to name a concrete failure or a concrete cost. Test value is judged, not
counted — coverage is an input that says where to look, never the finding itself.

### `/dock:upgrade [composer | npm | frontend | backend | <package>]`

Raises dependencies in a named scope and keeps the suite green. Not a `/dock:ship` plan,
because an upgrade cannot be planned in advance — the breakage is only discoverable by doing
it.

It runs the tests *before* touching anything, and stops if they are already red, since no
regression is detectable against a broken baseline. It writes no tests during the run: a test
written now covers code you just changed, so it proves nothing. Minors and patches go in one
batch. Every major goes on its own, with its own test run and its own commit.

Risk analysis runs on majors only. `web-researcher` reads the release notes in parallel, then
the bare names they mention — `Class::method()` in the notes is `$object->method()` in your
code — are searched across your source, excluding vendored directories. You approve the set
of majors in one round, not one question per package. A major that breaks reverts its own
manifest and lock and the run continues to the next.

## Agents

Every agent is a function of its prompt. None of them knows about a task folder, a plan, or
which skill called it — so they work inside a skill, inside another skill, or by hand.

| Agent | Model | Purpose |
|---|---|---|
| `dock:locator` | Sonnet | Where code, tests, config and docs live |
| `dock:analyzer` | Sonnet | How existing code works, and what depends on it, with file:line |
| `dock:pattern-finder` | Sonnet | Local precedent that new code should imitate |
| `dock:web-researcher` | Sonnet | External docs, APIs and versions, with citations |
| `dock:error-analyzer` | Opus | Runs a failing check, or takes an error → root cause → scoped fix |
| `dock:scout` | Sonnet | How a given repo builds, verifies and tests |
| `dock:plan-validator` | Fable | Pressure-tests a plan before any code exists |
| `dock:implementer` | Opus | One specified unit of work, verified and committed |
| `dock:reviewer` | Fable | Reviews a finished change, including whether it's the *right* fix |
| `dock:pr` | Sonnet | Writes and opens the PR |

## How it decides what to delegate

**Delegate work that reads much more than it writes.** Research reads forty files and
returns a page — always delegated. Implementing a planned phase reads little and writes a
lot, so the orchestrator does it in the main thread, where it still holds the plan and you
can still steer it.

The two review passes are delegated for a different reason: an independent angle is the
whole point. `plan-validator` runs before code exists, `reviewer` after — on a different
model family, because most of a review's value is in not sharing the author's blind spot.

## Development

Symlink the plugin into a skills directory and it auto-loads as `dock@skills-dir`:

```
ln -s ~/Code/Projects/dock/dock ~/.claude/skills/dock
```

`SKILL.md` edits apply immediately. Agent edits need `/reload-plugins`.

## Auto-delegation

Subagents have no opt-out from automatic delegation — the `description` field is the signal.
These are written as capability statements rather than triggers, and avoid "use proactively",
which keeps them from being picked up at random. To be certain, add to your global
`CLAUDE.md`:

> Never auto-delegate to `dock:*` agents — they are invoked explicitly by dock skills.

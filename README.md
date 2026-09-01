# dock

A Claude Code plugin. Takes a task from an issue or a prompt through research, planning,
implementation and review, and lands it as one PR.

## Install

```
claude plugin marketplace add p3yman/dock
claude plugin install dock@dock
```

## Skills

### `/dock:ship <ISSUE-KEY | task description> [--ask] [--no-worktree]`

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

## Agents

Every agent is a function of its prompt. None of them knows about a task folder, a plan, or
which skill called it — so they work inside a skill, inside another skill, or by hand.

| Agent | Model | Purpose |
|---|---|---|
| `dock:locator` | Sonnet | Where code, tests, config and docs live |
| `dock:analyzer` | Sonnet | How existing code works, with file:line |
| `dock:pattern-finder` | Sonnet | Local precedent that new code should imitate |
| `dock:web-researcher` | Sonnet | External docs, APIs and versions, with citations |
| `dock:error-analyzer` | Opus | Failure → root cause → scoped fix |
| `dock:scout` | Sonnet | How a given repo builds, verifies and tests |
| `dock:plan-validator` | Opus | Pressure-tests a plan before any code exists |
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

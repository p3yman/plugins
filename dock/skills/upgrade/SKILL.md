---
name: upgrade
description: Raise dependencies within a named scope to newer versions, one risky step at a time, keeping the tests green, and open a PR. Explicit invocation only.
argument-hint: [composer | npm | frontend | backend | <package>]
disable-model-invocation: true
---

# /dock:upgrade

Raise the dependencies in **$ARGUMENTS** and keep the suite green.

Minor and patch bumps go in one batch. **Every major goes on its own**, with its own test run and its own commit, because breakage is only discoverable by doing it — which is why this is not a `/dock:ship` plan.

## Step 0 — Scope

Take it from `$ARGUMENTS` — a manifest (`composer`, `npm`), a loose area (`frontend`, `backend`), or a single package. If nothing is given, ask once with `AskUserQuestion`; do not audit every manifest by default.

Read the lock file to pick the actual package manager: `pnpm-lock.yaml` means pnpm, `yarn.lock` means yarn, `poetry.lock` means poetry.

## Step 1 — Preflight

**Clean tree.** `git status --short`. Anything uncommitted, stop and ask the user to commit it first — you will be reverting manifests and lock files, and you must not take their work with you.

**Commands.** Spawn `dock:scout` for the install command, the test command, and the base branch.

**Baseline.** Run the tests *before touching anything*, and decide this step from that output alone.

- **Red already** → stop. Name the failures, say plainly that they predate this work so no regression can be detected against them, and raise nothing.
- **No tests, or zero collected** → say so and ask whether to continue blind. Wait for the answer.

Do not read test files to judge this, and **do not judge a test by its quality here**. For a baseline, a weak test is still a signal that something changed. (That is the opposite of `/dock:audit`'s test dimension, deliberately — different job, different rule.)

**Write no tests during this whole run.** A test written now covers code you just changed, so it can prove nothing about regression.

## Step 2 — Survey

| Manifest | Report | Raise one | Align |
|---|---|---|---|
| `composer.json` | `composer outdated --direct` | `composer require <pkg>:^<ver> --with-all-dependencies` | `composer bump` |
| `package.json` | `npm outdated` | `npm install <pkg>@^<ver>` | `npm update --save` |
| `Cargo.toml` | `cargo upgrade --dry-run --incompatible` | `cargo add <pkg>@<ver>` | `cargo upgrade` |
| `pyproject.toml` | `uv tree --outdated` | `uv add <pkg>@<ver>` | `uv-bump` |
| `go.mod` | `go list -m -u all` | `go get <pkg>@<ver>` | `go mod tidy` |

Substitute the project's actual package manager. Split what you find into **minors and patches** and **majors**.

## Step 3 — Risk, majors only

Reading release notes for a patch bump is waste. For each major:

1. `dock:web-researcher` — release notes, changelog and upgrade guide between the two versions. Fan these out in parallel, one per package; the notes are long and belong in an agent's context, not yours.
2. **Search your code for every bare name the notes mention.** The notes write `Class::method()`; your code calls `$object->method()`. Grep for the bare class, method, function and option names — never the qualified string. Exclude `vendor/`, `node_modules/` and `packages/`; only your own code counts.

That search is what separates a breaking-change check that works from one that returns nothing and feels thorough.

Per package, record: version now, version available, breaking changes, which of them touch your files, and an honest risk call.

## Step 4 — Report, then approve

Show the whole picture in one round — every minor and patch as a batch, every major with its risk and the files it touches — then ask once with `AskUserQuestion` which majors are in. Recommended first.

One approval round, not one question per package.

## Step 5 — Execute

`EnterWorktree({name: "upgrade-<scope>"})` off the base branch.

**Never symlink `vendor/`** — you are changing the very thing whose autoloader bakes in an absolute path, so a symlinked vendor would test the other checkout's packages. Real install, always.

Then:

1. **Minors and patches, all at once** → install → test. No question first; a patch carries no breaking change by contract.
2. **Each approved major, alone** → raise → apply the changes the notes named → test → commit.

**Commit per package**, message `upgrade: <pkg> <from> → <to>`. That makes the PR readable, resume free from `git log`, and rollback one revert.

**On a failing test after a major:**

- If the search missed a breaking change: `git checkout -- <manifest> <lockfile>`, change no source, report that package as skipped with what broke, and continue to the next.
- If the user explicitly approved it and you cannot fix it after a real attempt: same revert, same continue.

A failure ends *that package*, never the run.

## Step 6 — Align

Run the align command for each touched manifest, then test again. It writes installed versions back into the manifest, so `^1.0` against an installed `1.9.3` becomes `^1.9.3` — otherwise the manifest keeps lying about what the project needs.

Revert manifest and lock if this breaks the tests.

## Step 7 — PR

Spawn `dock:pr` with base, head, and the three lists from Step 8. Never merge it.

## Step 8 — Report

Give this in the **last message of every answer**, including one that asks a question or stops early — put the report first and the question last, since the run halts until an answer arrives and the user should not have to ask what happened so far.

- **Raised** — package, from → to
- **Skipped** — package, from → available, and why
- **Source files changed** — path, and which upgrade forced it

Then one sentence of its own naming the last change made and stating that the tests passed after it. One sentence, not two — two test-run sentences hide which run was last.

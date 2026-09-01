---
name: explain
description: Explain how a part of this codebase works, what depends on it, and why it is the way it is. Explicit invocation only.
argument-hint: <path | symbol | question>
disable-model-invocation: true
---

# /dock:explain

Answer **$ARGUMENTS** about this codebase.

The whole value here is **fan-out**: several agents reading in parallel, so you get one coherent answer and this thread's context stays clean. That only pays on something genuinely unfamiliar.

## Pick a depth, then commit to it

**Say which depth you chose and why**, in one line, before you start. Guessing wrong costs the user either accuracy or a minute of waiting, and they should know which trade you made.

**Direct** — a specific function, a single file, a narrow question you can answer by reading two files.
Read them and answer. **No agents.** Spawning three subagents to explain a helper is the failure mode this skill has to avoid.

**Standard** — "how does X work", where X is a feature, flow or subsystem.
`dock:locator` for the map, then `dock:analyzer` in parallel on the two or three paths that matter — one of them briefed for the **reverse** direction, so you can say what depends on it.

**Deep** — "why is it like this", an architecture-level question, or code that clearly fought someone.
Standard, plus `dock:pattern-finder` for whether it is idiomatic here or an outlier, plus history — which you run yourself, since it is a handful of commands returning little:

```
git log --oneline -- <path>
git log -S '<symbol>' --oneline
git blame -L <range> -- <path>
gh pr list --search '<path>' --state merged
```

The commit or PR that introduced something often *is* the explanation, and it is the one thing no amount of reading the current code will recover.

Add `dock:web-researcher` only when the answer depends on an external library's behavior rather than this repo's.

## Answer

Lead with the answer. Omit any section you have nothing real for — do not pad.

```
## <thing>

**In one line.**

### How it works
The path, in order, with file:line. Enough that they could change it without reading it first.

### What depends on it
Who breaks if this changes, and who merely touches it.

### Why it is like this
Only when history or a pattern actually explains it. Cite the commit or PR.

### Watch out for
Gotchas, sharp edges, things that look wrong but are load-bearing.

### Start here
The two or three files to open, in the order that makes sense.
```

## Rules

- **Every claim that matters gets a `path:line`.** An explanation you cannot check is a guess with formatting.
- **Say what you could not establish.** "I could not find where this is invoked in production" is a real finding, and far more useful than a plausible sentence covering the gap.
- Separate what the code *does* from what you infer it is *for*. Name which is which.
- **"It looks wrong" is worth saying only with evidence.** Load-bearing weirdness is common; call it out as suspicious, not as broken, unless you can show the failure.
- This skill answers questions. It does not change code, and it does not write files. If the answer turns into work, that is `/dock:issue` or `/dock:ship` — both do their own research.

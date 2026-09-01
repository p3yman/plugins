---
name: audit
description: Sweep a project for problems that scoped work never surfaces — security, data access, architecture, test value, dead code, dependencies. Reports only; writes nothing to the repo. Explicit invocation only.
argument-hint: [what to look for] [where]
disable-model-invocation: true
---

# /dock:audit

Find the problems that scoped work never flags **because they are correctly out of scope**. A PR review ignores everything outside its diff; `/dock:issue` parks tangents under Open Questions. Both are right to. This is the pass that looks at the whole.

Dependencies are not one of the dimensions — `/dock:upgrade` covers that ground properly.

## Two hard rules

**Write nothing to the repo.** No audit file, no findings doc, no `.dock/` directory. A stale audit record is worse than none — a later agent reads it as current truth and is misled by a problem that was fixed months ago. The report lives in this conversation; the only durable output is a Linear issue.

**Every finding needs a concrete failure or a concrete cost.** The input and state that break it, or the hours it will cost and when. "This could be cleaner", "consider extracting", "might be a problem at scale" are not findings. A long list of maybes is how an audit gets skimmed once and never run again.

## Step 1 — Scope it

If `$ARGUMENTS` names what to look for, use it and skip the questions.

Otherwise ask both at once with `AskUserQuestion`:

**What to look for** (multi-select):
- Security
- Data access
- Architecture & dead code
- Test value

Dependencies are deliberately not here — `/dock:upgrade` surveys them properly, checking
breaking changes against your actual code rather than just reporting version distance.

**Where**: whole project *(recommended for a first run)* / a named module or directory / only what changed recently.

"Where" matters — six agents reading a large repo is expensive, and a module-scoped audit is usually sharper anyway.

## Step 2 — Check what's already tracked

Search Linear for open issues covering the area (`linearis`, see its skill).

**Do not re-report a finding that already has an open issue.** List it separately as already tracked, with the key. This is the whole memory mechanism: findings you cared about became issues, findings you didn't are supposed to resurface. Re-reporting tracked work is what makes an audit feel like noise.

## Step 3 — Fan out

One agent per selected dimension, all in one message, each with a brief naming the paths from Step 1 and this dimension's specific quarry:

| Dimension | Agent | Looks for |
|---|---|---|
| Security | `dock:analyzer` | missing or bypassable authz, injection, secrets in code, unsafe deserialization, mass assignment, unvalidated redirects |
| Data access | `dock:analyzer` | N+1 queries, missing eager-loading, queries without limits, filters with no index, work inside loops |
| Architecture | `dock:analyzer` + `dock:pattern-finder` | layering violations, circular deps, classes doing several jobs, two implementations of one rule |
| Test value | `dock:analyzer` | tests that cannot fail, tests asserting on mocks, critical paths with no test, skips older than a few months, suites that have never gone red |
| Dead code | `dock:analyzer` *(reverse direction)* | unreferenced code, flags never flipped, commented-out blocks, endpoints nothing calls |

Two notes:

**Test value is not coverage.** Run the coverage report if one is configured, but as *input* — it says where to look, it is not the finding. Ninety percent coverage with assertions that cannot fail is worse than sixty with real ones, because it lies. Uncovered trivial code is not a finding; uncovered payment logic is. This is the opposite of the rule in `/dock:upgrade`, which treats any test as a valid baseline signal — different job, different rule.

Dead code is the natural use of `dock:analyzer`'s reverse direction: nothing referencing it is exactly the question that agent answers, and it is also why "delete this" needs care — check git history before calling something dead, since code can look unused and still be load-bearing.

## Step 4 — Report

Terminal only. Most severe first, each anchored to `path:line`.

```
## Audit: <dimensions> · <scope>

### <severity>: <finding>
path:line — how it fails: the input or state, and the wrong result.
Cost of leaving it: <what it breaks, or what it will cost and when>

### Already tracked
- ENG-412 — <finding> (open)

### Checked and clean
<dimension> — what you verified and found sound. Be specific; it tells the
reader what not to worry about.

### Could not check
<dimension> — why, and what would make it checkable.
```

"Checked and clean" is not filler. An audit that only ever returns problems gives no signal about the parts that are fine, and those are most of the codebase.

If a dimension produced nothing worth reporting, say so in one line. Padding a thin result is worse than a thin result.

## Step 5 — Offer to file

Propose which findings are worth a Linear issue — the ones you would act on, not everything found. For each accepted one, hand it to `/dock:issue --quick` so it lands properly shaped rather than as a pasted paragraph.

Findings not filed are not tracked anywhere, deliberately. They come back next audit if they still matter.

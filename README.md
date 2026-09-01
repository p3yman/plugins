# plugins

A Claude Code plugin marketplace. Add it once per machine, then install whatever you want
from it.

```
claude plugin marketplace add p3yman/plugins
```

## Plugins

| Plugin | Install | What it does |
|---|---|---|
| [dock](dock/) | `claude plugin install dock@p3yman` | The full lifecycle of a change — shape an idea into an issue, ship it as a reviewed PR, explain unfamiliar code, audit what's rotting, upgrade what's stale. Five skills over one shared pool of ten agents. |

Installed plugins update with `/plugin update`.

## Adding a plugin to this marketplace

Drop the plugin directory in at the repo root — it needs its own
`.claude-plugin/plugin.json` — and add an entry to `.claude-plugin/marketplace.json`
pointing at it. Machines that already added this marketplace can install it immediately
with `claude plugin install <name>@p3yman`; no second `marketplace add`.

Validate before pushing:

```
claude plugin validate .          # the marketplace manifest
claude plugin validate ./<name>   # the plugin manifest
```

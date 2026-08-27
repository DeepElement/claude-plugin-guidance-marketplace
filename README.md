# claude-plugin-guidance-marketplace

A Claude Code plugin marketplace (name: `claude-plugin-guidance`). All plugins
in this marketplace use the `guidance-` name prefix.

This marketplace provides guidance *for people building their own Claude Code
plugin marketplaces* — opinionated conventions (naming, required docs,
attribution) layered on top of the official plugin/marketplace standard.
It does not redefine that standard. The plugin/marketplace directory
structure, manifest schema, and component types (skills, commands, agents,
hooks, etc.) are a living convention owned by Anthropic and documented at:

- [Plugins reference](https://code.claude.com/docs/en/plugins-reference)
- [Create plugins](https://code.claude.com/docs/en/plugins)
- [Create and distribute a plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)
- [Discover and install plugins](https://code.claude.com/docs/en/discover-plugins)

Always defer to those pages for what's structurally required — this repo
only adds opinions on top.

## Scope

This marketplace is an **authoring layer for people building their own
Claude Code plugin marketplaces** — it is not a general-purpose plugin
collection, and its plugins are not meant to be installed into ordinary
application workspaces. The intended consumer is someone who owns or is
creating a plugin marketplace repo (their own `.claude-plugin/marketplace.json`
plus a `plugins/` directory) and wants that repo to conform to the
conventions and architecture documented here.

Given that scope, every plugin in this repo is designed to be installed
directly into a marketplace-builder's own project, where it can see and
act on that project's plugin files as they're authored. Installing these
plugins should make it easy to *fall into* the right structure — sane
defaults, checks that run proactively as files are edited — and, when a
marketplace repo doesn't conform, produce a clear, specific notice about
what's wrong and how to fix it, rather than a silent gap or a vague
warning.

## Add to Claude Code

```
/plugin marketplace add <this-repo-url-or-path>
```

## Architecture

Plugins in this marketplace that represent a swappable capability (a
"concept" with multiple possible provider implementations) follow a
concept/realization pattern: an abstract concept, a realization contract,
one or more concrete provider realizations, and a consuming-workspace
settings file that selects and configures a provider per concept.

- [How this marketplace works](./docs/architecture.md) — the strategy,
  in plain terms: what problem it solves and what adopting it produces.

## Plugins

Install these into a marketplace-builder's own repo (not into a downstream
application workspace). Each is designed to surface conformance issues as
clear, specific notices — proactively, while you're authoring plugin
files, or on demand — rather than requiring you to remember the rules.

- **guidance-conventions** (`plugins/guidance-conventions`) — provides:
  - `validate-marketplace` — checks your marketplace repo against these
    conventions (naming prefix, required docs, no restated standard,
    attribution notice), separately flagging schema issues against the
    official docs above.
  - `validate-concept-plugin` — checks any plugin in your repo that has
    opted into the [concept/realization architecture](./docs/architecture.md)
    (any plugin with `skills/concept/`) for a correct tier 1/2/3
    structure, a single default realization, published realization
    config schemas, and activation-check invocation.
- **guidance-activation-check** (`plugins/guidance-activation-check`) —
  provides `check-realization-config`, the shared pre-flight check a
  concept realization invokes before doing real work: validates a
  consuming workspace's `marketplace-plugin-settings.yml` against the
  active realization's published config schema and halts with a
  specific, actionable message on anything missing or stale. Also
  usable directly to audit a settings file.

## Structure

- `.claude-plugin/marketplace.json` — marketplace manifest (name:
  `claude-plugin-guidance`)
- `plugins/` — individual plugins, each named `guidance-<name>` with its own
  `.claude-plugin/plugin.json`, following the official plugin layout linked
  above

## Adding a plugin

1. Create `plugins/guidance-<name>/.claude-plugin/plugin.json` following the
   [official plugin reference](https://code.claude.com/docs/en/plugins-reference).
2. Add an entry to `.claude-plugin/marketplace.json`'s `plugins` array
   pointing at `./plugins/guidance-<name>`.
3. Run the `validate-marketplace` skill from `guidance-conventions` to check
   it against these conventions.

## License

Licensed under the Apache License, Version 2.0, with an additional public
attribution requirement (see [LICENSE](./LICENSE), Section 10). Any public
use of this project or a derivative work must include a visible statement
of attribution to DeepElement / claude-plugin-guidance with a link back to
this repository. This isn't just legal boilerplate — it's how people who
could use these tools find them.

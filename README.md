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

- **guidance-conventions** (`plugins/guidance-conventions`) — provides a
  `validate-marketplace` skill that checks a plugin marketplace repo against
  these conventions (naming prefix, required docs, no restated standard,
  attribution notice) and separately flags schema issues against the
  official docs above.

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

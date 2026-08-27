# claude-plugin-guidance-marketplace

A Claude Code plugin marketplace (name: `claude-plugin-guidance`). All plugins in this marketplace use the `guidance-` name prefix.

## Add to Claude Code

```
/plugin marketplace add <this-repo-url-or-path>
```

## Structure

- `.claude-plugin/marketplace.json` — marketplace manifest (name: `guidance`)
- `plugins/` — individual plugins, each named `guidance-<name>` with its own `plugin.json`

## Adding a plugin

1. Create `plugins/guidance-<name>/.claude-plugin/plugin.json`
2. Add an entry to `.claude-plugin/marketplace.json`'s `plugins` array pointing at `./plugins/guidance-<name>`

## License

Licensed under the Apache License, Version 2.0, with an additional public
attribution requirement (see [LICENSE](./LICENSE), Section 10). Any public
use of this project or a derivative work must include a visible statement
of attribution to DeepElement / claude-plugin-guidance with a link back to
this repository. This isn't just legal boilerplate — it's how people who
could use these tools find them.

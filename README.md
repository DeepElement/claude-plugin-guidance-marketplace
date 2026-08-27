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

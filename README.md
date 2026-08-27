# claude-plugin-guidance

An authoring layer for people building their own Claude Code plugin
marketplaces. Install these plugins into your marketplace repo and they'll
guide you toward good structure as you write it, and give you a clear,
specific notice when something's off — instead of leaving you to remember
the rules yourself.

This is **not** a general-purpose plugin collection — it's not meant to be
installed into an ordinary application workspace. It's for the person
building the marketplace repo itself (a `.claude-plugin/marketplace.json`
plus a `plugins/` directory).

## Install

Run these from inside your marketplace project:

```
/plugin marketplace add DeepElement/claude-plugin-guidance-marketplace
/plugin install guidance-conventions@claude-plugin-guidance
/plugin install guidance-activation-check@claude-plugin-guidance
```

## What you get

- **guidance-conventions** — checks your marketplace repo's naming,
  required docs, and structure as you author it, including a deeper check
  for any plugin that adopts the [concept/realization pattern](./docs/architecture.md)
  described below.
- **guidance-activation-check** — a shared pre-flight check your plugins
  can call before doing real work, so a missing or outdated configuration
  produces one clear, actionable message instead of a confusing failure.

## The concept/realization pattern

If a plugin you're building represents a capability with more than one
possible implementation (e.g. secrets, notifications — something with
swappable providers), this marketplace recommends a specific pattern for
it: an abstract concept, a contract concrete providers must satisfy, one
or more provider realizations, and a single settings file where a
consumer picks and configures a provider.

Read **[How this marketplace works](./docs/architecture.md)** for the
full explanation and the detailed spec — this README stays intentionally
short.

## The base standard

This repo doesn't redefine the Claude Code plugin/marketplace standard
itself (directory layout, manifest schema, component types) — that's
owned by Anthropic and documented at:

- [Plugins reference](https://code.claude.com/docs/en/plugins-reference)
- [Create plugins](https://code.claude.com/docs/en/plugins)
- [Create and distribute a plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces)

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the PR workflow and the
conventions your plugin needs to follow.

## License

Apache 2.0, with an added requirement: any public use of this project
("claude-plugin-guidance") or a derivative must visibly credit
"DeepElement" and link back to its source repository,
[DeepElement/claude-plugin-guidance-marketplace](https://github.com/DeepElement/claude-plugin-guidance-marketplace).
See [LICENSE](./LICENSE) for the exact terms.

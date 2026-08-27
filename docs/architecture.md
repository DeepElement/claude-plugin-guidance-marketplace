# How this marketplace works

This document explains the strategy behind `claude-plugin-guidance` in
plain terms: what problem it solves, how the pieces fit together, and
what a workspace actually gets by adopting it. For the base Claude Code
plugin/marketplace standard this builds on, see the official docs linked
from the [README](../README.md) — this document does not restate that
standard.

## The problem

Plugin marketplaces tend to accumulate one plugin per provider: a
Slack-notifications plugin, a Teams-notifications plugin, an
AWS-secrets plugin, a Vault-secrets plugin. Two things go wrong as that
grows:

1. **Duplication.** Every provider-specific plugin re-explains the same
   underlying concept ("send a notification," "fetch a secret") in its
   own words, with its own conventions, its own quality bar.
2. **Lock-in by convenience.** A workspace picks whichever provider
   plugin they installed first. Nothing in the plugin's own instructions
   distinguishes "what this concept does" from "how this one provider
   happens to do it" — so other skills that need the concept end up
   coupled to that one provider's plugin by name, and switching
   providers means rewriting every caller.

This marketplace is organized to avoid both, by treating **the concept**
— not the provider — as the unit of distribution.

## The strategy: concepts over providers

Every plugin here represents one capability, named for what it does
(`guidance-secrets`, not `guidance-aws-secrets-manager`). Inside that one
plugin, three layers separate "what" from "how":

```
guidance-<concept>/
├── concept              (Tier 1 — abstract: what this capability does)
├── realization-contract (Tier 2 — the schema a provider must satisfy)
└── realize-<provider>   (Tier 3 — one or more concrete providers)
```

- **Anything that needs this capability references the concept by
  name** — never a specific provider. A deployment skill that needs
  secrets says "use the `secrets` concept," full stop. It has no idea,
  and doesn't need one, whether that resolves to AWS Secrets Manager,
  1Password, or a workspace's own internal vault.
- **The realization contract is the seam.** It's a published schema —
  what operations a provider must expose, what configuration it needs
  from the workspace — that any provider implementation must satisfy to
  plug in. This is what makes providers swappable: they're not
  compatible by convention, they're compatible by contract.
- **Providers are interchangeable and extensible.** This marketplace
  ships at least one working provider for every concept, so a workspace
  is never left with an abstraction and nothing to run. But a workspace
  isn't limited to what we ship — they can write their own provider
  skill, satisfy the same contract, and it works identically to a
  marketplace-shipped one.

## Where the workspace fits in

A workspace using this marketplace makes exactly one kind of decision,
in exactly one file — `marketplace-plugin-settings.yml` at its root: for
each concept, *which provider* to use, and *what configuration that
provider needs* (credentials, endpoints, account IDs, whatever the
provider's contract calls for).

That's the entire integration surface. Nothing about the workspace's own
skills needs to know which provider is selected — they keep referencing
the concept by name, and the settings file is what resolves that
reference to a concrete implementation at the point of use.

## Why this is safe to adopt: the activation check

The one place this pattern could go wrong quietly is configuration drift
— a marketplace update renames a provider, or a workspace never filled
in a required credential, and a skill fails deep inside a provider
implementation with a confusing error, or worse, silently does the wrong
thing.

This marketplace closes that gap with a dedicated check that runs before
any provider does real work: it reads the workspace's settings, confirms
the selected provider still exists and matches what's configured, and
validates every required configuration value is present. If anything is
wrong, the workspace gets a specific, actionable message — "concept X's
selected provider Y needs config field Z" — instead of a failure buried
in provider-specific logic. This is what makes the pattern trustworthy
to adopt broadly: a missing or stale configuration is always a clear,
early, blocking message, never a silent failure.

## What adopting this produces

For a workspace:

- One configuration file to manage every marketplace capability it uses,
  regardless of how many concepts or providers are involved.
- The ability to switch providers for any concept — or replace a
  marketplace-shipped provider with an internal one — without touching
  any skill that consumes the concept.
- A guaranteed default: every concept is usable immediately after
  install, with at most some configuration values left to fill in, and
  a clear signal when they're missing.

For this marketplace's maintainers:

- New provider support is additive — one new Tier 3 skill against an
  existing contract, not a new plugin with its own conventions to learn.
- The abstract/contract layers (Tier 1/2) rarely change once a concept
  is established, so the surface that could break workspace integrations
  is small and stable.
- Quality and documentation standards apply once, at the concept level,
  rather than being re-litigated per provider.

## What this is not

This pattern applies to *concept plugins* distributed through this
marketplace — plugins meant to represent a capability with swappable
providers. It is deliberately more structure than a small, single-purpose
plugin needs, and shouldn't be forced onto one. Whether a given plugin
in this marketplace should follow this pattern, versus being a simple,
single-tier plugin, is a judgment call made when the plugin is proposed,
not a rule applied universally.

## Detailed design

This section is the precise technical specification for the pattern
described above: how a concept plugin is structured internally, how
realizations are supplied and configured, and how the activation check
works. It governs the internal design of concept plugins built for this
marketplace — it does not redefine the Claude Code plugin/marketplace
standard itself (directory layout, `plugin.json` schema, skill file
format); see the official docs linked from the README for that.

### 1. A root plugin is a "Concept"

Every top-level plugin in `plugins/` that follows this pattern
represents one abstract capability ("Concept"), named for what it does,
not for how it's implemented — e.g. `guidance-secrets`, not
`guidance-aws-secrets-manager`. Concepts are siblings: none depends on
another's internal structure. A concept may *reference* another concept
by name alone (see §5) without knowing or caring which realization backs
it in a given workspace.

Each concept plugin contains, as skills within that single plugin:

- **Tier 1 — Concept skill** (`skills/concept/SKILL.md`)
  Describes the capability in provider-agnostic terms: what problem it
  solves, what operations it exposes, when to use it. This is the file
  other concepts' skills reference by name. It contains no knowledge of
  any specific realization.

- **Tier 2 — Realization contract** (`skills/realization-contract/SKILL.md`
  plus a sibling `skills/realization-contract/schema.json`)
  Defines what a realization *must* provide to satisfy this concept:
  - the operations/interface a realization implements (described in the
    SKILL.md prose)
  - the **base input configuration schema** a realization requires from
    the consuming workspace (see §3), as a JSON Schema document in
    `schema.json` — versioned, so both marketplace-shipped and
    workspace-authored realizations can declare conformance and be
    validated against it mechanically (see §7 for the schema.json
    convention)
  - the identifying name a realization registers under (used in
    `marketplace-plugin-settings.yml`, see §3)

  This tier is a contract, not an implementation. It never runs on its
  own.

- **Tier 3 — Realizations** (`skills/realize-<provider>/SKILL.md` plus a
  sibling `skills/realize-<provider>/schema.json`, one per concrete
  provider, N+1 where N ≥ 0 — see §4)
  Each is a concrete implementation satisfying the Tier 2 contract for
  one specific provider/technology. Each realization:
  - declares which concept + contract version it satisfies (in SKILL.md)
  - publishes its own `schema.json`, a superset compatible with Tier 2's
    base `schema.json` (see §3, §7)
  - begins its instructions by invoking the activation-check convention
    (§6) before doing any real work

### 2. Realizations may come from two sources

A concept's usable realizations are the union of:

- **Marketplace-shipped realizations** — Tier 3 skills bundled directly
  in the concept plugin in this repo, pre-built bridges for known
  providers.
- **Workspace-authored realizations** — skills living in the consuming
  workspace's own skill collection (e.g. `.claude/skills/`), which
  declare conformance to a specific concept + Tier 2 contract version
  by name, without needing to live in or be known to this marketplace.

Both are referenced identically from `marketplace-plugin-settings.yml`
by a realization name — the settings file doesn't care where a
realization physically lives.

### 3. Workspace configuration: `marketplace-plugin-settings.yml`

A consuming workspace declares its bindings in a single root-level file:

```yaml
# marketplace-plugin-settings.yml
secrets:
  realization: aws-secrets-manager
  config:
    region: us-east-1
    profile: default

notifications:
  realization: my-custom-slack-bridge   # workspace-authored realization
  config:
    webhook_env_var: SLACK_WEBHOOK_URL
```

- Top-level keys are **concept names**.
- `realization` selects which Tier 3 realization is active for that
  concept in this workspace, by the realization's registered name.
- `config` is validated against **that realization's `schema.json`**
  (which must be compatible with — a superset of — the concept's Tier 2
  base `schema.json`). Realizations are responsible for publishing this
  schema (§1, Tier 3; conventions in §7) so tooling and the activation
  check (§6) can validate a workspace's `config` block without
  inspecting the realization's implementation.

### 4. Every concept ships at least one default realization

Each concept plugin in this marketplace must include at least one
marketplace-shipped Tier 3 realization, pinned as the default (e.g. via
a `default: true` marker in the contract or an explicit note in the
concept skill). This guarantees a workspace can use any marketplace
concept immediately after installing it, without first having to author
or select a realization — they may still need to fill in required
`config` values, which is the activation check's job to surface (§6),
not a reason the concept fails to have a usable default.

### 5. Cross-concept references stay abstract

When one concept's instructions need another concept's capability
(e.g. a deployment concept needing secrets), they reference the other
concept **by name** (its Tier 1 skill) only. They never reference a
specific realization or assume one is active. Resolution of "which
realization backs `secrets` in this workspace" happens the same way for
every caller: consult `marketplace-plugin-settings.yml`, then run the
activation check (§6) before use.

### 6. Activation check: a shared convention, not a hook

A dedicated plugin, `guidance-activation-check`, provides a shared skill
that any Tier 3 realization's instructions invoke first, before doing
real work. It:

1. Reads `marketplace-plugin-settings.yml` for the concept in question.
2. Confirms a `realization` is selected and that it matches the
   realization currently running (catches stale selection after a
   marketplace update renames/removes a realization).
3. Validates the `config` block against the active realization's
   `schema.json` (§3, §7), reporting specific missing/invalid fields.
4. If anything fails, halts with a clear, actionable message (what's
   missing, which file to edit, which schema to satisfy) **instead of**
   letting the realization proceed and fail deeper or silently
   misbehave.

This is implemented as a shared skill convention (every realization's
SKILL.md is expected to invoke it as its first step) rather than a
Claude Code hook, so it works the same way regardless of how a given
realization is triggered, and doesn't depend on hook mechanics that may
change independently of this pattern.

### 7. Best practices per tier

These are recommendations, checked (where mechanical) by
`validate-concept-plugin` in `guidance-conventions`. They describe what a
*good* Tier 1/2/3 looks like, beyond the structural minimum in §1–§6.

#### Tier 1 — Concept skill

- State the capability as an action a caller wants, not as a technology
  category — "store and retrieve a secret by name," not "a secrets
  manager." This keeps it easy for another concept to reference without
  absorbing implementation vocabulary.
- Enumerate the operations a realization must support as a short list
  (verbs, inputs, outputs), not prose — this list is what Tier 2 turns
  into a contract.
- Name failure modes the concept can have in the abstract (e.g. "secret
  not found," "not authorized") without naming how any one provider
  reports them. Tier 3 realizations map their provider's actual errors
  onto this vocabulary.
- Never mention a specific provider, SDK, or vendor API in the body
  text. A reader should not be able to guess which realization is the
  default from reading Tier 1 alone.

#### Tier 2 — Realization contract

- Keep the SKILL.md prose focused on the *interface*: which operations
  from Tier 1 map to which required behavior, and what a realization is
  allowed to vary (e.g. performance characteristics, error detail) versus
  required to guarantee (e.g. idempotency, the shape of a returned
  value).
- The `schema.json` sibling file is the base configuration contract —
  keep it minimal. Only include a field here if *every* realization,
  regardless of provider, would need something in that shape (for
  example, most secrets providers need some notion of a "scope" or
  "namespace," even if the field name a given provider uses differs
  after mapping). Provider-specific fields belong in Tier 3's schema,
  not here.
- Version the contract explicitly (a `contractVersion` field in
  `schema.json`, e.g. `"1.0.0"`). Bump it on any breaking change to the
  base schema or required operations, so realizations and the activation
  check can detect incompatibility instead of failing confusingly.
- Write at least one realistic example `config` block satisfying the
  base schema, even though Tier 2 has no concrete provider — this is
  what a workspace-authored realization's author copies as a starting
  point.

#### Tier 3 — Realizations

- Name the realization after the provider/technology, not after the
  concept (`realize-aws-secrets-manager`, not `realize-provider-a`) —
  the identifying name in `marketplace-plugin-settings.yml` should be
  recognizable to someone who already knows the provider.
- `schema.json` here must validate as a strict superset of Tier 2's
  base schema: every field Tier 2 requires stays required (or is
  narrowed, never removed or loosened), and provider-specific fields are
  added as needed (e.g. `region`, `vault_address`, `profile`). Never
  reuse a field name from Tier 2's schema with a different meaning.
- Every field in `schema.json` should have a `description` explaining
  what it's for and, where relevant, an example value — this is the text
  a workspace author reads when filling in `marketplace-plugin-settings.yml`
  for the first time, often with no other documentation in hand.
- Mark optional-but-recommended fields with sensible defaults in the
  schema (`default:`) rather than making everything required — a
  realization should need the smallest config that actually works,
  deferring advanced options to optional fields.
- Invoke the activation check (§6) as literally the first instruction in
  the realization's SKILL.md, before any operational logic, so it's
  unambiguous that nothing else runs first.
- If a realization is the concept's pinned default (§4), say so
  explicitly in its SKILL.md (not only in the concept skill), so anyone
  reading the realization file directly still knows its status.

#### The `schema.json` convention (Tier 2 and Tier 3)

A plain [JSON Schema](https://json-schema.org/) document, one per
`skills/realization-contract/` and per `skills/realize-<provider>/`
directory:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "contractVersion": "1.0.0",
  "type": "object",
  "properties": {
    "region": {
      "type": "string",
      "description": "AWS region the secret lives in.",
      "examples": ["us-east-1"]
    },
    "profile": {
      "type": "string",
      "description": "Named AWS CLI profile to use for credentials.",
      "default": "default"
    }
  },
  "required": ["region"]
}
```

Tooling (the activation check, `validate-concept-plugin`) reads this file
directly rather than parsing prose, which is why it exists as a sibling
file instead of embedded narrative in SKILL.md.

### Consequences

- Adding a new provider for an existing concept means adding one Tier 3
  skill to the concept plugin (or to a workspace's own skills) — Tier 1
  and Tier 2 don't change.
- Consuming workspaces get one predictable file
  (`marketplace-plugin-settings.yml`) and one predictable failure mode
  (a clear activation-check block) regardless of which concept or
  realization is involved.
- This pattern adds structure/ceremony that a single-purpose plugin
  doesn't need — it applies to concept plugins in this marketplace, not
  to every plugin anyone ever writes.
- Naming discipline matters: concept names, realization names, and
  contract versions are the join keys across settings.yml, Tier 2, and
  Tier 3. Renaming any of these is a breaking change for consuming
  workspaces (consistent with the plugin-name stability rule in the
  official plugin standard).

### Open questions (not yet decided)

- Whether contract versioning uses semver strings compared by the
  activation check, or a simpler compatible/incompatible flag.

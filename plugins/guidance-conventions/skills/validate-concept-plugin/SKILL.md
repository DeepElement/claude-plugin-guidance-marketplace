---
name: validate-concept-plugin
description: Validate that a plugin claiming to be a "Concept" plugin correctly follows the concept/realization architecture (tier 1 concept, tier 2 realization contract, tier 3 realizations, default realization, activation-check invocation). Use proactively whenever the user is authoring or editing a plugin under plugins/*/skills/concept/, plugins/*/skills/realization-contract/, or plugins/*/skills/realize-*/, or explicitly asks to check/validate/audit a concept plugin's structure.
---

# Validate Concept Plugin

Checks a single plugin in this marketplace against the concept/realization
architecture described in
[docs/architecture.md](../../../../docs/architecture.md). Read that document
first if you haven't already — this skill enforces it, it doesn't restate
it.

This skill only applies to plugins that opt into the pattern. It does not
apply to plain, single-tier plugins, and should never demand that a plugin
adopt this structure — only that a plugin which *has* adopted it (see
"How to detect a Concept plugin" below) does so correctly.

## How to detect a Concept plugin

A plugin is a Concept plugin if it has a
`skills/concept/SKILL.md` file. Presence of that directory is the sole
signal — no `plugin.json` field is required to opt in. If a plugin has
`skills/realization-contract/` or `skills/realize-*/` but no
`skills/concept/`, flag that on its own (see check 1) rather than assuming
intent.

If a plugin has none of `skills/concept/`, `skills/realization-contract/`,
or `skills/realize-*/`, it's a plain plugin — skip all checks below and
say so.

## Checks

For each Concept plugin found (or the one specified by the user):

1. **Tier 1 present and abstract.** `skills/concept/SKILL.md` exists and
   its content describes the capability without naming a specific
   provider/vendor/technology in the body (a provider name appearing only
   as a passing example is fine; the operations/description should not
   assume one specific provider's API or behavior). Flag any tier 1 file
   that reads like documentation for one particular provider.

2. **Tier 2 present and is a contract, not an implementation.**
   `skills/realization-contract/SKILL.md` exists and defines the
   operations/interface a realization implements and the identifying
   name convention a realization registers under. Flag if this file
   contains actual working implementation logic rather than a
   schema/contract definition — that content belongs in a tier 3
   realization.

3. **Tier 2 `schema.json` present and valid.**
   `skills/realization-contract/schema.json` exists, parses as valid
   JSON Schema, and includes a `contractVersion` field (blocking if
   missing entirely; non-blocking if present but missing
   `contractVersion`). Per the
   [`schema.json` convention](../../../../docs/architecture.md#the-schemajson-convention-tier-2-and-tier-3),
   this should stay minimal — flag (non-blocking) any field that looks
   provider-specific rather than something every realization would need.

4. **At least one Tier 3 realization exists.** At least one
   `skills/realize-<provider>/SKILL.md` exists. If zero exist, this is a
   hard failure — a Concept plugin with tier 1 and tier 2 but no
   realization is not usable by a consuming workspace.

5. **Exactly one default realization.** Among all `skills/realize-*/`
   directories, exactly one is marked as the default (per whatever marker
   the contract specifies, e.g. `default: true` in frontmatter or an
   explicit statement in the realization's SKILL.md). Flag if zero are
   marked default (workspace has no out-of-the-box option) or if more than
   one claims to be default (ambiguous).

6. **Every realization publishes a valid, superset `schema.json`.** Each
   `skills/realize-<provider>/schema.json` exists, parses as valid JSON
   Schema, and is a superset of tier 2's `schema.json` — every field
   `required` in tier 2 is still `required` here (never dropped or
   loosened), and field types/meanings aren't changed. Missing entirely
   is blocking; present but not a valid superset (e.g. narrows or drops
   a required field) is also blocking, since it silently breaks the
   contract tier 2 promises. Also check, non-blocking:
   - every property has a `description`
   - optional fields have a `default` where a sensible one exists
   - the realization's own default status (§4 of the architecture doc)
     is stated in its SKILL.md, not only in the concept skill

7. **Realization naming matches provider, not concept.** Flag
   (non-blocking) any `skills/realize-<provider>/` whose name is generic
   or repeats the concept name (e.g. `realize-default`) rather than
   naming the actual provider/technology it implements.

8. **Every realization invokes the activation check first.** Each
   realization's SKILL.md instructs Claude to run the activation check
   (from `guidance-activation-check`, if installed) before doing any real
   work, so missing/stale workspace configuration is caught with a clear
   message rather than failing inside the realization. Flag any
   realization missing this step. If `guidance-activation-check` isn't
   installed in this marketplace yet, note that as a separate,
   marketplace-level gap rather than failing every realization
   individually.

9. **Naming consistency.** The realization identifying names used in each
   `skills/realize-*/SKILL.md` are unique within the plugin and match
   what tier 2 says the naming convention should be (e.g. matching the
   directory name minus the `realize-` prefix).

## Reporting

Group findings as:
- **Blocking** — missing tier 1, missing tier 2, missing or invalid tier
  2 `schema.json`, zero realizations, a realization missing `schema.json`
  or whose `schema.json` isn't a valid superset of tier 2's, zero or
  multiple default realizations. These mean the plugin cannot be used as
  documented, or silently breaks the contract, and should be fixed before
  merging/publishing.
- **Non-blocking** — missing `contractVersion`, missing field
  descriptions/defaults, undocumented default status, generic realization
  naming, missing activation check invocation, naming drift. These
  degrade the guarantees the architecture doc promises (clear config
  errors, swappability, self-explanatory config) but don't make the
  plugin unusable.

For each finding, give the file and a one-line fix suggestion. Do not
silently modify files — report and let the user decide, unless they've
explicitly asked you to fix issues found.

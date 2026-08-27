---
name: validate-concept-plugin
description: Validate that a plugin claiming to be a "Concept" plugin correctly follows the concept/realization architecture (tier 1 concept, tier 2 realization contract, tier 3 realizations, default realization, activation-check invocation), validate a workspace-authored realization skill against a concept's published contract, and check that concepts reference each other only by name. Use proactively whenever the user is authoring or editing a plugin under plugins/*/skills/concept/, plugins/*/skills/realization-contract/, or plugins/*/skills/realize-*/, whenever a consuming workspace is authoring its own realization skill for a marketplace concept, or whenever the user explicitly asks to check/validate/audit a concept plugin, a workspace realization, or cross-concept references.
---

# Validate Concept Plugin

Checks marketplace concept plugins, and workspace-authored realizations of
them, against the concept/realization architecture described in
[docs/architecture.md](../../../../docs/architecture.md). Read that document
first if you haven't already — this skill enforces it, it doesn't restate
it.

This skill only applies to plugins/skills that opt into the pattern. It
does not apply to plain, single-tier plugins, and should never demand that
a plugin adopt this structure — only that a plugin or realization which
*has* adopted it (see "How to detect a Concept plugin" below) does so
correctly.

This skill covers three distinct checks, run independently depending on
what's being authored or what the user asks for:

- **A. Marketplace concept plugin structure** — checks 1–9 below, for a
  concept plugin living in this marketplace's own `plugins/` tree.
- **B. A workspace-authored realization** — see "Validating a
  workspace-authored realization" below, for a realization skill living
  outside this marketplace (e.g. in a consuming workspace's own
  `.claude/skills/`) that claims to satisfy a marketplace concept's
  contract.
- **C. Cross-concept reference discipline** — see "Cross-concept
  reference check" below, scanning any concept's Tier 1 skill for leaked
  coupling to a specific realization.

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

## Checks (A: marketplace concept plugin structure)

For each Concept plugin found (or the one specified by the user):

1. **Tier 1 present and abstract.** `skills/concept/SKILL.md` exists and
   its content describes the capability without naming a specific
   provider/vendor/technology in the body (a provider name appearing only
   as a passing example is fine; the operations/description should not
   assume one specific provider's API or behavior). Flag any tier 1 file
   that reads like documentation for one particular provider. Also check,
   non-blocking:
   - the operations a realization must support are enumerated as a
     short list (verb, inputs, outputs) rather than buried in prose
   - failure modes are named in the abstract (e.g. "not found," "not
     authorized") rather than as one provider's actual error names/codes

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
   Also flag (non-blocking) if `skills/realization-contract/SKILL.md`
   contains no example `config` block satisfying the base schema — this
   is what a workspace author writing their own realization copies as a
   starting point.

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
   message rather than failing inside the realization. Flag (blocking)
   any realization missing this step entirely; flag (non-blocking) one
   where the invocation is present but not the literal first instruction
   (e.g. operational steps precede it). If `guidance-activation-check`
   isn't installed in this marketplace yet, note that as a separate,
   marketplace-level gap rather than failing every realization
   individually.

9. **Naming consistency.** The realization identifying names used in each
   `skills/realize-*/SKILL.md` are unique within the plugin and match
   what tier 2 says the naming convention should be (e.g. matching the
   directory name minus the `realize-` prefix).

## Validating a workspace-authored realization (B)

Per §2 of the architecture doc, a consuming workspace can author its own
realization for a marketplace concept (e.g. in `.claude/skills/` outside
this marketplace) instead of using a marketplace-shipped one. Use this
check when the user is authoring such a skill, or asks to validate one.

1. **Identify the target concept.** The workspace realization's SKILL.md
   must declare which concept and contract version it satisfies (per §1
   Tier 3 requirements). If it doesn't say, ask the user which marketplace
   concept plugin it's meant to realize — do not guess.

2. **Locate that concept's contract.** Find the concept plugin's
   `skills/realization-contract/schema.json` and `SKILL.md` (in this
   marketplace, or wherever the concept plugin is installed). If the
   concept plugin isn't available to inspect, report that this check
   can't complete and say what's needed (the concept plugin installed or
   its contract files provided).

3. **Run the same checks as Tier 3 realizations in set A**, applied to
   this workspace skill instead of a `skills/realize-<provider>/`
   directory:
   - it publishes its own `schema.json` (blocking if missing)
   - that `schema.json` is a valid superset of the concept's tier 2
     `schema.json` (blocking if it drops/narrows a required field)
   - every property has a `description` (non-blocking)
   - it invokes the activation check as its first instruction
     (non-blocking if present-but-not-first, blocking if entirely absent)
   - its declared identifying name doesn't collide with an existing
     marketplace-shipped realization's name for the same concept unless
     it's intentionally meant to override/replace it (ask the user if
     unclear)

4. **It does not need**: a matching directory name convention, a
   `default: true` marker (workspace realizations are opted into via
   `marketplace-plugin-settings.yml`'s `realization:` field, never as an
   implicit default), or to live under any particular path — those A-set
   rules are about this marketplace's own shipped plugins specifically.

Report the same way as set A: blocking vs. non-blocking, file and fix
suggestion per finding.

## Cross-concept reference check (C)

Per §5 of the architecture doc, one concept's instructions may reference
another concept by name only — never a specific realization, and never
assuming one is active. Use this check when reviewing any concept
plugin's Tier 1 (`skills/concept/SKILL.md`) or Tier 3
(`skills/realize-*/SKILL.md`) content that mentions another concept, or
when the user asks to audit cross-concept coupling across the
marketplace (or a subset of it).

1. **Scan for other concepts' realization names.** For each concept
   plugin found, collect the full set of realization identifying names
   registered by every *other* installed concept plugin (from their
   `skills/realize-*/SKILL.md` files).

2. **Check for leaked references.** Search this concept's `skills/concept/`
   and `skills/realize-*/` files for any of those other concepts'
   realization names appearing in the instruction text. A reference to
   another concept by its **concept name** (e.g. "use the `secrets`
   concept") is correct and expected — flag only references to a
   specific *realization* of another concept (e.g. "use
   `aws-secrets-manager`" instead of "use the `secrets` concept").

3. **Check for hardcoded assumptions.** Flag (non-blocking) any
   instruction phrased as though a particular realization is always
   active for another concept (e.g. "since this uses AWS, assume the
   region config is set") rather than resolving it generically through
   that concept's own activation check.

This check is inherently best-effort text matching — report findings as
suggestions to review, not certainties, since natural-language text can
mention a provider name for reasons unrelated to a hard dependency (e.g.
a comparison in documentation). Always show the matched text so the user
can judge intent.

## Reporting

State which check set(s) ran (A, B, C, or a combination) before listing
findings, so the user knows what was and wasn't covered. Group findings
within each set as:

- **Blocking** — missing tier 1, missing tier 2, missing or invalid tier
  2 `schema.json`, zero realizations, a realization or workspace-authored
  skill missing `schema.json` or whose `schema.json` isn't a valid
  superset of tier 2's, zero or multiple default realizations (set A
  only — not applicable to a workspace-authored realization), a
  realization/workspace skill missing the activation-check invocation
  entirely. These mean the plugin/skill cannot be used as documented, or
  silently breaks the contract, and should be fixed before
  merging/publishing/using it.
- **Non-blocking** — missing `contractVersion`, missing example config
  block, missing field descriptions/defaults, undocumented default
  status, generic realization naming, activation check present but not
  first, naming drift, un-enumerated operations or non-abstract failure
  modes in Tier 1, and any set-C cross-concept reference finding (always
  non-blocking — text-matched, needs human judgment). These degrade the
  guarantees the architecture doc promises (clear config errors,
  swappability, self-explanatory config, loose coupling) but don't make
  the plugin unusable outright.

For each finding, give the file and a one-line fix suggestion. Do not
silently modify files — report and let the user decide, unless they've
explicitly asked you to fix issues found.

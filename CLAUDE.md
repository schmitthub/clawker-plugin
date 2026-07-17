# Clawker Plugin — Development Guide

This repository is a Claude Code plugin and its marketplace in one: the
`clawker-support` plugin lives at the repo root, and
`.claude-plugin/marketplace.json` serves it with a relative source (`"./"`).
It ships two skills: `skills/clawker-support` — a clawker internals expert for
end users (setup, config, troubleshooting) — and `skills/bundle-creator` — a
guided authoring workflow for bundles and their components (harnesses, stacks,
monitoring extensions). The shared documentation philosophy is
**minimal concrete details; point at live docs**.

## Skill Plugin Conventions

These are Claude Code agent skills — not libraries, apps, or scripts. They
follow the Claude Code plugin and skill authoring conventions:

- **Use the Claude Code skill creator** (`/skill-creator` or the skill creator
  agent) for auditing skill definitions, validating SKILL.md frontmatter, and
  checking that skills follow Claude Code's plugin standards.
- **SKILL.md is the skill definition.** Its frontmatter (`name`, `description`,
  `allowed-tools`, `license`, `compatibility`) must conform to the Claude Code
  skill spec. The body is the prompt that runs when the skill is invoked.
- **Reference files are context, not code.** Files in `reference/` directories
  are loaded by skill agents during execution. They teach methodology — they
  are not executed, compiled, or parsed programmatically.
- **plugin.json is the package manifest.** It follows the Claude Code plugin
  spec (`name`, `description`, `version`, `author`, `license`, `repository`,
  `homepage`). Its `version` is the plugin's single version of record; bump it
  once per unmerged release.
- Consult `https://code.claude.com/docs/en/plugins` for the current plugin and
  skill authoring standards when making structural changes.

## Relationship to the Clawker Codebase

This repo is mounted as a git submodule inside the main
[schmitthub/clawker](https://github.com/schmitthub/clawker) repo (at
`clawker-plugin/`). That mount exists solely to prevent drift: changes to
clawker's config schema, CLI commands, or architecture land in the same
working tree as the skill docs they invalidate, and clawker's CI and
pre-commit hooks drift-check the plugin against the templates it mirrors.
The submodule pointer in the clawker repo records which plugin commit a given
clawker commit was verified against.

Distribution is direct: users add this repo as a marketplace (or install via
the clawker CLI), and Claude Code fetches the plugin from this repo at its
default branch.

## Repository Structure

```
clawker-plugin/
├── .claude-plugin/
│   ├── marketplace.json          # Marketplace catalog (serves this repo via "./")
│   └── plugin.json               # Plugin metadata and version of record
├── README.md                     # User-facing install and usage docs
├── CLAUDE.md                     # This file — development guide
├── skills/bundle-creator/
│   ├── SKILL.md                  # Guided authoring workflow: interview → scaffold → walkthrough → validate → build
│   └── reference/
│       ├── bundle-envelope.md    # Bundle identity, layout, addressing, publishing, path:/url: dev loop
│       ├── stacks.md             # Stack authoring: interview questions, self-guard pattern, gotchas
│       ├── harnesses.md          # Harness authoring: manifest walkthrough order, security gotchas
│       └── monitoring.md         # Monitoring-extension authoring: lanes, routing, retention, gotchas
└── skills/clawker-support/
    ├── SKILL.md                  # Main skill definition and workflow
    └── reference/
        ├── Dockerfile.base.tmpl          # Copy of the actual base-image template (drift-checked)
        ├── Dockerfile.harness-image.tmpl # Copy of the actual harness-image template (drift-checked)
        ├── project-config.md     # Project config discovery, layering, troubleshooting
        ├── sample-go.yaml        # Reference config: Go project (clawker's own)
        ├── sample-node.yaml      # Reference config: Node.js project
        ├── settings.md           # User settings schema, troubleshooting
        ├── mcp-recipes.md        # MCP setup methodology, troubleshooting
        ├── troubleshooting.md    # Entry point routing to domain-specific sections
        ├── docker-hygiene.md     # Docker disk space diagnosis and cleanup
        ├── monitoring.md         # OTel + OpenSearch + Prometheus stack, Clawker workspace, telemetry env, troubleshooting
        ├── firewall-security.md  # Proactive VCS egress lockdown — git credential-exfil defense
        ├── claude-code.md        # Claude Code integration — in-container auth model, managed-config staging, /login troubleshooting
        └── known-issues.md       # Active bugs and workarounds
```

## Core Principle: Minimal Concrete Details

This skill avoids concrete configuration details where possible. Configs,
packages, APIs, CLI flags, and tooling evolve constantly. Baking field names,
domain lists, or flag syntax into the skill produces stale guidance that agents
treat as authoritative.

**When concrete details DO appear, they are deliberate and load-bearing.**
They represent either stable architectural concepts (e.g., `agent.post_init`
as the build-time vs runtime boundary, and `agent.pre_run` as its once-vs-every-
start counterpart) or curated reference samples that are manually kept current.

### Reference config samples

`reference/sample-*.yaml` files contain working `.clawker.yaml` configs for
different stacks (Go, Node.js). These are standalone YAML files — not inlined
in markdown — so they only consume context when the agent reads them for a
relevant task. `project-config.md` has a table pointing to each sample.

Samples are manually maintained and NOT drift-checked. When updating, copy
from a known-working source. The docs site
(`https://docs.clawker.dev/configuration`) remains the authoritative schema
reference and should still be fetched for field-level details.

### What belongs in the skill files

- Decision frameworks and methodology (how to think about a problem)
- Workflow steps and interview questions
- Pointers to live documentation URLs
- Architectural concepts that are stable (discovery rules, layering model)
- Gotchas about common mistakes
- Curated reference config samples (manually maintained, not drift-checked)

### What does NOT belong in the skill files

- Exhaustive field name lists (point to docs instead)
- CLI flag syntax (point to docs instead) — **exception:** the firewall
  path-scoping flags (`--path`/`--action`, `path_default`/`path_rules`) are
  deliberate and load-bearing security methodology, kept concrete in
  `firewall-security.md`, SKILL.md, and `troubleshooting.md`. The narrowest-
  scope advice is meaningless without them. Treat as a curated reference, not
  drift-prone syntax.
- Domain lists (hardcoded firewall domains, registry URLs)
- Version numbers or base image digests

## Reference File Conventions

Each domain reference file (`project-config.md`, `settings.md`, `mcp-recipes.md`)
follows the same structure:

1. What it is and how it differs from other domains
2. How to get the current schema (always a docs URL)
3. Domain-specific methodology
4. **Troubleshooting** section (consistent heading name across all refs)

`troubleshooting.md` is the entry point — it has a routing table that points
to domain-specific troubleshooting sections and keeps only global/cross-cutting
diagnostics (clawker not found, firewall, credentials, container won't start).

## Dockerfile Template Sync

`Dockerfile.base.tmpl` and `Dockerfile.harness-image.tmpl` in `reference/` are
copies of the actual templates from the clawker repo's
`internal/bundler/assets/`. If clawker's templates change, these copies must be
updated to match. The clawker repo's pre-commit hook and Plugin CI workflow
(`Dockerfile template drift check`) catch drift via the submodule checkout.

When updating the templates, never hardcode new field names into the skill —
update methodology and docs URLs instead.

## Versioning

Plugin version lives in `.claude-plugin/plugin.json`, mirrored in the
marketplace entry in `.claude-plugin/marketplace.json` — keep both in sync.
**Every change bumps the patch number (`1.0.Z` → `1.0.Z+1`). No exceptions.**

The version IS the delivery mechanism: the marketplace caches by version, so a
change with no bump never reaches installed users. "Is this worth a bump?" is
the wrong question: if you touched the plugin, bump it. A typo fix, a new
reference file, and a workflow rewrite are each just the next patch.

Keep it patch-only. Minor/major are reserved for a deliberate, announced change
in how the plugin works, which effectively never happens here.

## Completion Gate

After making changes to the plugin:

1. Check that `known-issues.md` is still accurate — remove entries for fixed
   bugs; verify reference cross-references are consistent (troubleshooting
   routing table, SKILL.md research step references)
2. Bump the patch version in `plugin.json` AND the marketplace entry (see
   Versioning — the release pipeline requires an increment per change)
3. In the clawker repo, bump the `clawker-plugin/` submodule pointer to the
   new commit so drift checks run against it

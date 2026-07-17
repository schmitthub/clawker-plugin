# The Bundle Envelope

What makes a directory of components distributable. Current schema:
fetch https://docs.clawker.dev/authoring-bundles before walking these fields.

## Interview questions

1. **Namespace** — the maintainer brand (org, handle, umbrella). Self-declared,
   no central registry, so it must be distinctive: a generic namespace invites
   identity collisions (two declared bundles with the same `(namespace, name)`
   pair are a hard error for consumers). Reserved and rejected: `clawker`,
   impersonation forms (`clawker-*`, `*-clawker`), and `official`.
2. **Bundle name** — combines with namespace to form the identity. Never
   derived from the repo URL, so renaming the repo never breaks consumers.
3. **Where will it live?** A dedicated git repo, or a subdirectory of an
   existing one — both are installable sources.
4. **Version now or later?** `version` is optional and carries **no
   compatibility semantics** — it exists only so update checks can detect
   change. When absent, the resolved source commit is the version. Fine to
   omit while authoring.

Charset for namespace, name, and every component name: lowercase letters,
digits, internal hyphens. **No dots** — dots are the address separator.

## Layout

```
my-bundle/
├── .clawker-bundle/
│   └── bundle.yaml          # pure metadata — nothing about components
├── harnesses/<name>/        # any subset of the three convention dirs
├── stacks/<name>/
└── monitoring/<name>/
```

Components are discovered purely from the convention directories; the
manifest never lists them. A single-component bundle is the same shape.
Unknown top-level dirs are advisory warnings (`--strict` fails them).

## Addressing

A bundled component is addressed everywhere as
`namespace.bundle.component` — in `build.stacks`, `monitor.extensions`, the
`-t`/`@:` harness selector, and image tags. Three segments, always splits
cleanly because names contain no dots.

**Promotion gotcha:** when a loose component moves into a bundle, any
self-references change spelling. The big one: a bundled harness that depends
on a stack shipped in the same bundle must reference that stack by the
bundle's own qualified address, not the bare name (a bare name resolves loose
tiers and floor — not the bundle it sits in).

## Publishing and the dev loop

There is no registry. Publishing = commit, tag, share the URL. Consumers:

```yaml
# clawker.yaml
bundles:
  - url: https://github.com/acme/tools.git
    ref: v1.2.0        # optional — unpinned tracks the default branch
```

then `clawker bundle install` (no-arg installs the declared set). Exact
reproduction = pin `sha` instead of `ref`.

**Author dev loop:** declare the working tree as a `path:` source — it loads
in place, every edit is live, no reinstall:

```yaml
bundles:
  - path: ./my-bundle    # relative to the declaring file's directory
```

Swapping `url:` ↔ `path:` toggles released vs working copy: an identity is
claimed by exactly one live declaration, and the cached release goes inert
(not purged) while the `path:` entry holds the identity. Declaring **both**
at once is an identity collision — clawker refuses to resolve until one is
dropped. A local directory never silently stands in for an installed bundle.

## Validation ritual

`clawker bundle validate <dir>` locally while iterating;
`--strict` in anything resembling CI or a pre-publish check. Hard failures
are genuine load errors caught early; warnings are hygiene.

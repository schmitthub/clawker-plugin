# Authoring Stacks

A stack provisions a language toolchain into the image. Small manifest, and
Dockerfile fragments that do the real work. Current schema: fetch
https://docs.clawker.dev/authoring-stacks before walking the fields.

## Interview questions

1. **What toolchain, and how is it canonically installed?** Prefer the tool's
   official install path (upstream apt repo, official tarball, vendor
   installer script) over language-manager indirection. Check how the
   example repo's stacks do it before proposing anything.
2. **System-wide, per-user, or both?** Decides which fragments exist:
   - system-level install (e.g. `/usr/local`) → root fragment
   - per-user tooling (version switchers, `~/.local` installs) → user fragment
   - many toolchains want both (runtime system-wide + switcher per-user)
3. **Is the version overridable?** Almost always yes — expose it as a
   build `ARG` so consumers can `clawker build --build-arg X_VERSION=...`.
4. **Name** — the directory name is the stack name. Bare-name loose stacks
   with a built-in's name (`node`, `go`, `python`, `rust`, ...) deliberately
   shadow the built-in — fine if intended, surprising if not.

## Layout

```
<name>/
├── stack.yaml                     # manifest — essentially just `description`
├── Dockerfile.stack-root.tmpl     # runs as root, before user creation
└── Dockerfile.stack-user.tmpl     # runs as the container user, after switch
```

Ship whichever fragments the toolchain needs; root-only is common.

## The load-bearing pattern: self-guard

A stack must be safe to declare unconditionally — the same name can render in
more than one build stratum (project-declared stacks render in the base
image; harness-declared stacks render in the harness image), and the image
may already carry the runtime. Every fragment guards itself:

```dockerfile
ARG MYTOOL_VERSION=1.2
RUN if command -v mytool >/dev/null 2>&1; then \
      echo "clawker stack mytool: already present — skipping"; \
    else \
      # ... install ... \
    ; fi
```

Skip the guard and a doubly-declared stack breaks the build or double-installs.
This is the number-one thing to keep when adapting an example.

Fragments are Go text templates with a handful of build-context variables
available. Don't enumerate the variables from memory — copy the shape from
the closest stack in the example repo
(https://github.com/schmitthub/clawker-bundle-example, `stacks/`), which also
demonstrates GPG/checksum verification for downloads. Keep verification when
the upstream offers it.

## Walkthrough order

1. `stack.yaml` — one `description` field; write a sentence that says what
   lands where (e.g. "Node.js LTS on /usr/local plus nvm as the per-user
   switcher").
2. Root fragment — install steps, guarded, version ARG.
3. User fragment — only if per-user tooling exists.
4. Validate, then prove with a real `clawker build` if available (declare in
   `build.stacks`).

## Gotchas

- Project-declared stacks render **before** the project's
  `root_run`/`user_run` instructions — user build steps may rely on the
  toolchain, never the reverse.
- Pin versions concretely in the ARG default; "latest" makes builds
  unreproducible.
- Downloads happen on the host daemon at build time (outside the agent
  firewall), but keep sources official and verified anyway — the stack runs
  in every consumer's image.

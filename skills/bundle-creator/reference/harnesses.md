# Authoring Harnesses

A harness tells clawker how to build and run a coding-agent CLI: a manifest
declaring runtime needs, plus a Dockerfile fragment that installs the agent.
The most involved component type — walk it slowly. Current schema: fetch
https://docs.clawker.dev/authoring-harnesses before walking the fields.

## Interview questions

1. **What agent, and what is its canonical install?** Read the agent's actual
   install script/docs — do not trust secondhand descriptions of installer
   flags. Respect the tool's native layout; if its installer picks a
   directory, extend `PATH` via `ENV` rather than relocating or symlinking.
2. **What does it run on?** Node? Python? Self-contained binary? → the
   `stacks` dependency list. A bundled harness depending on a stack in the
   same bundle references it by the bundle's own qualified address.
3. **Where does it keep state?** Config dir, data/session dir, auth — these
   become `volumes` so logins and sessions survive container recreation.
4. **What host state should follow the user in?** Global instructions file?
   Settings? → `staging`. **Credentials and host-path-keyed config stay
   out** (see gotchas).
5. **What network does the agent need to function?** → the `egress` floor.
   Minimal by design: the API endpoints the agent can't start without.
   Widen only from observed blocked traffic, never preemptively.
6. **Does it load a managed instructions file?** (like `/etc/claude-code/CLAUDE.md`)
   → `managed_prompt` destination. Omit the block if there's no such location.

## Layout

```
<name>/
├── harness.yaml               # manifest
├── Dockerfile.harness.tmpl    # install fragment for this harness's image
└── assets/                    # files referenced by seeds (optional)
```

## Walkthrough order (each concept builds on the last)

1. **`version`** — how the agent's version resolves at build time: an npm
   package, a GitHub release (with optional tag prefix strip), or none. The
   resolved version is available to the fragment — install exactly that
   version, don't re-resolve "latest" inside the fragment.
2. **`stacks`** — runtime dependencies. Self-contained binaries need none.
3. **`volumes`** — each entry becomes a named volume mounted under the
   container home (`path` is home-relative). Declare one per state dir that
   must persist. Every seed/staging destination must land inside one.
4. **`seeds`** — first-boot defaults applied by clawker's init step, sourced
   from `assets/`. Apply modes cover copy-if-missing and JSON-merge shapes —
   fetch the docs for the current list.
5. **`staging`** — create-time host→container copies of agent state living
   outside the workspace. Explicit `src`→`dest` entries only; nothing is
   copied by convention. `src` expands `~` and env-var fallbacks. Entries can
   narrow to an allowlist of JSON keys so host secrets never travel.
6. **`managed_prompt`** — where clawker bakes its managed container-context
   briefing into the image (content is clawker's; the harness only names the
   destination). Build-time copy — the destination must NOT sit under a
   declared volume, or the mount shadows it.
7. **`egress`** — the firewall floor, same rule vocabulary as project
   firewall rules (`dst`, optional `proto`/`port`/`path_rules`). This is a
   security boundary: declare exactly what the agent needs, no more.
8. **The fragment** — `Dockerfile.harness.tmpl` is a set of
   `{{define "<block>"}}` bodies filling named slots in the master template.
   Slot names encode permission scope + position in the build timeline
   (root vs user, before/after stack rendering and shell switch), and one
   slot sets the container `CMD`. Fetch the authoring-harnesses docs for the
   current block table — defining any other template name is a validation
   error, and project inject-point names are reserved. Install the agent CLI
   in the user-scope block the shipped harnesses use, set `CMD` to launch it.

Start the fragment from the closest example in
https://github.com/schmitthub/clawker-bundle-example (`harnesses/` — claude,
codex, and opencode, the last being the standalone-installer case).

## Gotchas

- **Never stage credentials.** Auth files (`auth.json`, keychains, tokens)
  stay on the host; the user logs in inside the container, and the auth
  lands in a declared volume so it persists. Also skip config files that
  embed host paths (instruction-file references, local MCP command paths) —
  they dangle in the container; let the agent regenerate defaults.
- **Egress floor is load-bearing security.** A container refuses to start if
  its harness label names a harness that's no longer available, precisely so
  it never runs with a weaker floor than it was built for.
- **Installer contracts:** verify by reading the live install script. Vendor
  installers change flags and layouts; docs summaries (including AI ones)
  are frequently stale.
- Version pinning: the resolver hands the fragment a concrete version — wire
  it through to the installer (env var or flag) so builds reproduce.

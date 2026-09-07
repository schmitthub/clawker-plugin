# Known Issues

Active bugs and workarounds. Check this before deep-diving into
troubleshooting — the user's problem may already be documented here.

## go build VCS error in older worktree containers

In worktree containers created before the `GOFLAGS=-buildvcs=false` default,
any `go build` of a main package fails:

```
error obtaining VCS status: exit status 128
	Use -buildvcs=false to disable VCS stamping.
```

Go skips the worktree's `.git` *file*, walks up to the mounted main `.git`
directory, and trips git's `safe.directory` check on the root-owned mount
scaffold dir. Plain git inside the worktree works, which makes this confusing.

Fix: recreate the container (new worktree containers set
`GOFLAGS=-buildvcs=false` automatically). In-place workaround for an existing
container: `export GOFLAGS=-buildvcs=false` (or add it to `agent.env` and
recreate). Do **not** suggest `git config --global --add safe.directory` for
the main repo path — git at that path sees the whole tree as deleted, and a
`git add -A` there would stage mass deletions into the host's real index.

## Empty refreshToken / forced /login in older containers

Containers created with an older clawker release injected a host keychain
credential blob that was round-tripped through a typed struct, fabricating a
zero-value `organizationUuid` (`00000000-...`) the user is not a member of.
Claude Code's token refresh endpoint rejects that claim, and Claude Code is
known to **wipe its stored credential with empty data when a refresh fails**
(upstream anthropics/claude-code behavior) — the user sees an empty
`refreshToken` in `~/.claude/.credentials.json` and a forced `/login` at boot.

Current releases never copy host credentials into containers at all —
authentication happens inside the container and the credential persists in the
harness config volume — so new containers are unaffected. The poisoned blob
persists in config volumes created by affected releases, though.

Fix: run `/login` once inside the affected container — Claude Code writes a
fresh credential into the config volume, which self-heals from there.
Host-side re-authentication has no effect on containers (credentials are
independent families; nothing is copied or synced).

**Also affected-releases-only:** containers whose volumes hold a host-copied
credential can hit an *expected* one-time `/login` from refresh-token
rotation — refresh tokens are single-use, so if the host "turned in" the
shared token first, the container's copy is invalidated (`invalid_grant`).
Same fix: `/login` once; the container then owns its own credential family.
The struct-roundtrip bug above is distinguished by a zero-value
`organizationUuid` in the poisoned blob. Current auth model and `/login`
troubleshooting in `claude-code.md`.

## --worktree on a branch already checked out

`clawker run --worktree <branch>` for a branch that is already checked out in
the main repo (or another worktree) is refused:

```
branch is already checked out in another worktree: "<branch>" is checked out at <path>
```

This mirrors native `git worktree add` — one branch cannot be checked out in two
places, or a commit in one moves the other's HEAD onto content it never checked
out. Common trigger: `git switch -c <branch>` in the repo root, then
`clawker run --worktree <branch>` on the same name.

Fix: either switch the root checkout off the branch (`git switch <other>` there),
or pass `--worktree <new-branch>` and let clawker create and own a fresh branch
(don't pre-create it in root). If a worktree commit already slid root's HEAD,
`git -C <root> switch <branch>` (or `git switch -f <branch>`) re-separates them;
no commits are lost — they live on the branch ref.

## Claude Code ignores `nvm use` / `nvm alias default` / `.nvmrc`

Images built with the node stack (declared by the claude harness, or via
project `build.stacks`) carry Node + npm on PATH and the agent's Bash tool
uses them out of the box — so this only bites once a user installs
**additional** Node versions with `nvm` and expects the agent to switch onto
one.

Claude Code selects the Node for its Bash tool when it builds its shell
snapshot, and it does **not** honor nvm's normal version-selection mechanisms:
`nvm use <ver>`, `nvm alias default`, and a project `.nvmrc` are all ignored.
It picks a version by scanning `~/.nvm/versions/node/*` directly, so an
installed-but-not-selected version can win and the agent runs a different Node
than `nvm` reports. This is an upstream Claude Code bug
([anthropics/claude-code#54135](https://github.com/anthropics/claude-code/issues/54135)),
not a clawker defect — until it's fixed, the nvm directives won't steer the
agent.

Until upstream fixes it, nvm's version directives won't steer the agent's Bash
tool. The reliable fix is to bake the Node version you need as the single
default instead of juggling versions with nvm: override the `NODE_VERSION` build
arg to the major line you require, e.g. `clawker build --build-arg NODE_VERSION=22`.
That major is then the only Node on `/usr/local`, so the agent always selects it
and the selection bug never applies. `NODE_VERSION` takes a **major** line only
(`22`, `24`, …) — the latest LTS patch of that line is resolved per build; minor
and patch pins are not supported. Only reach for `nvm` (and hit this bug) when
you genuinely need multiple Node versions in one image.

## Docker daemon transport: Unix socket required, TCP not supported

This is a limitation, not a bug. Clawker connects its containers to the
Docker daemon only through a Unix socket bind mount.

Clawker resolves the daemon address in this order (same as the `docker`
CLI):

1. `DOCKER_HOST`, when it is set
2. The `docker.host` key in `settings.yaml` — a full scheme-qualified
   address, for example `unix:///run/user/1000/docker.sock`. Releases
   before 2026.8.3 named this key `docker.socket`; a stored value
   migrates automatically and gets a `unix://` prefix.
3. The active docker CLI context (`docker context show`)
4. The default path `/var/run/docker.sock`

A rootless install records its socket in the docker CLI context, so
clawker finds it with no configuration. Releases before this resolution
always used the default path; on those releases, container create fails on
a rootless host with: `bind source path does not exist:
/var/run/docker.sock`. The fix is to upgrade clawker. Rootless Docker is
fully supported from 2026.8.3 — see
https://docs.clawker.dev/container-internals#rootless-docker for the two
once-per-boot sudo steps.

A daemon address that is not `unix://` (for example `tcp://`) has no socket
file. Clawker cannot mount it. Two features do not operate in that
configuration:

- The control plane container. Its Docker client serves the firewall stack
  and the agent watcher through the mounted socket. The `agent.from_env`
  option does not apply to the control plane container.
- Docker-outside-of-Docker for agents (`security.docker_socket: true`).

Support for a local daemon over TCP is a planned follow-up
([schmitthub/clawker#450](https://github.com/schmitthub/clawker/issues/450)).
A daemon on a different machine is out of scope by design: the eBPF firewall
attaches to cgroups on the local kernel.

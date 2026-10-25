# Clawker Monitoring Stack

The clawker monitoring stack is an opt-in OpenTelemetry pipeline that
captures Claude Code logs/metrics, clawker CLI logs, control-plane logs,
and firewall (Envoy + CoreDNS) logs. It runs on its own Docker Compose
network, never on the `clawker-net` agent network, and is managed
entirely through the `clawker monitor` subcommand. Every published port
binds `127.0.0.1`; agent containers cannot resolve or reach any
monitoring service.

## What it is

Four containers brought up by `clawker monitor up`:

- **OTel Collector** — receives agent telemetry from the control plane
  (agents export to a loopback relay in their container; the control
  plane stamps the verified `project`/`agent`/`container_id` and forwards
  on its own mTLS receiver), fans out to OpenSearch (logs) and Prometheus
  exporter (metrics).
- **OpenSearch** — log storage. Core infra indices (`clawker-cli`,
  `clawkercp`, `clawker-envoy`, `clawker-coredns`, `clawker-ebpf-egress`)
  are always present; the `claude-code` index and its dashboards are
  contributed by the built-in `claude-code` monitoring extension, which a
  project opts into via `monitor.extensions` (no default selection). Other
  extensions add their own indices.
- **OpenSearch Dashboards (OSD)** — UI at `http://localhost:5601`.
- **Prometheus** — metrics scrape + UI at `http://localhost:9090`.

A one-shot `clawker-opensearch-bootstrap` container runs after
OpenSearch is healthy and applies index templates, ingest pipelines, an
ISM retention policy, the `clawker_prometheus` direct-query datasource,
and imports the **Clawker analytics workspace** with index patterns +
dashboards. The collector gates on bootstrap completing successfully —
it never starts until the cluster is preconfigured. Prometheus starts in
parallel with bootstrap (bootstrap depends on Prometheus being up so
the datasource registration can validate the configured URI). If
bootstrap fails, the collector does not start.

## Monitoring extensions (what the stack observes)

Beyond the core firewall/control-plane telemetry, what the stack observes is
contributed by **monitoring extensions** a project selects under
`monitor.extensions` in `clawker.yaml` (a whole-value selection — the highest
config layer that sets it wins; the default selection is the built-in
`claude-code` extension, and an explicit empty list opts out of all
monitoring). `clawker monitor up` is bring-up only: it seeds the current
project's selected extensions (indices, ingest pipelines, dashboards) when it
starts the stack, and exits untouched — with an already-up notice — when the
stack is running; there is no host-side enable registry, and the collector
routes from the union of every extension ever seeded. Apply a selection edit
to a running stack with `clawker monitor reload` (seeds the selection,
re-renders the config, and recreates the collector). An extension is a `harnesses`/`stacks`-peer component
resolved across the same three tiers (built-in floor, loose
`.clawker/monitoring/<name>/`, installed bundle `namespace.bundle.component`).
Fetch `https://docs.clawker.dev/monitoring-extensions` (consume/select) and
`https://docs.clawker.dev/authoring-monitoring` (author) for details.

## How to get into the workspace

From the OpenSearch Dashboards splash / welcome screen, click
**Clawker** under the **Analytics** panel on the far right of the page.

Inside the workspace, the left navbar's **Explore** section has
**Logs** (across all six indices) and **Metrics** (Prometheus-backed).
Three pre-built dashboards live under the workspace's **Dashboards**
view — **Claude Code Cost & Usage** (session / cost / token / tool
KPIs), **Claude Code Activity** (prompts, tooling, code editing,
permissions, hooks, MCP, plugins audit), and **Clawker Networking**
(Envoy / CoreDNS / eBPF egress action pies). Users are expected to
build additional dashboards on top.

The authoritative reference for the stack and its semantics is
`https://docs.clawker.dev/monitoring` — fetch it for index field
schemas, OTel resource attribute conventions, or detailed
configuration. Do not invent field names from training data.

## What agent containers send

Every clawker-built image has OTel env vars baked at build time
(`OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`, the three
`OTEL_*_EXPORTER=otlp` settings, `CLAUDE_CODE_ENABLE_TELEMETRY=1`,
etc.). `OTEL_EXPORTER_OTLP_ENDPOINT` is always
`http://127.0.0.1:4318` — the loopback OTLP/HTTP relay that the
container's clawkerd daemon serves. Agent telemetry never leaves the
container directly: clawkerd carries each batch to the control plane
over its mTLS identity, the control plane stamps the verified
`project`/`agent`/`container_id` and forwards to the collector. Do
not diagnose by probing the collector from inside an agent; check
clawkerd's log (`/var/log/clawker/clawkerd.log`) for
`otlp_relay_ready`, `otlp_relay_unavailable`, or
`otlp_relay_cp_unavailable` instead.

At container creation, Clawker checks whether the `otel-collector`
container is running. If it is not, it overrides
`OTEL_METRICS_EXPORTER`, `OTEL_LOGS_EXPORTER`, and
`OTEL_TRACES_EXPORTER` to `none` for that container's lifetime, so
there is no error or retry storm. When the stack comes up, telemetry
flows on the next container start. Already-running agents do NOT
retroactively enable — the SDK reads the exporter settings at process
start, not per-export.

This means: **start the monitoring stack before starting agent
containers** if you want to capture their telemetry. Containers that
predate `monitor up` will produce no data until they restart.

## Troubleshooting

### No data in any index

1. **Is the stack actually up?**
   ```
   clawker monitor status
   ```
   Reports per-service container state and bootstrap exit status.

2. **Did bootstrap fail?** If it did, the collector never started
   (it gates on bootstrap). Prometheus starts in parallel — it may
   be running even if bootstrap failed. Inspect the bootstrap
   container's logs:
   ```
   docker logs clawker-opensearch-bootstrap
   ```
   Common cause: stale OpenSearch volume from a prior incompatible
   version. The monitoring stack is preconfigured and ephemeral by
   design — tear it down with volumes and re-init rather than trying
   to migrate state:
   ```
   clawker monitor down --volumes
   clawker monitor init --force
   clawker monitor up
   ```

3. **Agents started before the stack came up?** Their exporters were
   set to `none` at creation. Restart the agents so they are recreated
   with exporters enabled:
   ```
   clawker container restart --agent <agent-name>
   ```

### `claude-code` index has no documents

`CLAUDE_CODE_ENABLE_TELEMETRY=1` is baked into the image and is never
toggled. The switch is the exporter set: if the agent container was
created before the monitoring stack came up, it has
`OTEL_METRICS_EXPORTER`, `OTEL_LOGS_EXPORTER`, and
`OTEL_TRACES_EXPORTER` set to `none` for its lifetime. Restart the
agent after `clawker monitor up` so it is recreated with the exporters
set to `otlp`. Confirm with `docker inspect <agent> --format
'{{.Config.Env}}'` (Claude Code strips OTEL vars from its shell env,
so `env` inside the agent is misleading). If the exporters are `otlp`
and the index is still empty, check clawkerd's log for `otlp_relay_*`
events — the relay degrades silently when it cannot reach the control
plane. Fetch `https://docs.clawker.dev/monitoring` for the current
variable list and recommended values.

### `clawker-envoy` / `clawker-coredns` indices empty

These indices fill only when the firewall is enabled AND processing
traffic. If the firewall is disabled or no agent has made an outbound
request, the indices will exist but be empty. Confirm via:
```
clawker firewall status
```

### Workspace `Clawker` not visible in OSD

Bootstrap likely failed or was skipped. Re-init:
```
clawker monitor down --volumes
clawker monitor init --force
clawker monitor up
```
Do not edit OSD saved-objects by hand expecting them to persist — the
stack is throwaway and `monitor down --volumes` wipes them.

### Prometheus has no metrics

1. Check that the collector is scraping itself and exporting on the
   expected port. The collector's metrics-exporter expiration is
   tuned for long-lived series (currently `8760h` — never expire);
   a setting of `0` would expire every metric on every Collect cycle.
2. Confirm Prometheus is up at `http://localhost:9090/targets` — the
   `otel-collector` job should be `UP`.

### Cannot run `clawker monitor` commands from inside an agent container

The monitoring stack is host-side infrastructure. From inside an
agent, use `docker exec` against the stack containers (or check
their IPs via `docker inspect`; they are not on `clawker-net`) — do NOT try to run
host-side `clawker monitor up/down/status` from within an agent
container.

## Anti-patterns

- **Treating the stack as durable state.** It is preconfigured +
  ephemeral. Core index templates, ISM policies, and the Clawker
  workspace ride the bootstrap; extension assets (indices, pipelines,
  dashboards) are seeded from each project's `monitor.extensions`
  selection at bring-up (or `monitor reload`). The answer to almost every
  state-management question is "regenerate with `clawker monitor down
  --volumes && clawker monitor init --force && clawker monitor up`" —
  `--volumes` clears both telemetry data and every seeded extension.
- **Adding fields to OSD index patterns by hand.** They will be
  overwritten on the next `monitor init`. If a field is missing,
  the fix is in the bootstrap image, not in OSD.
- **Running `docker compose` against the stack directly.** Use the
  `clawker monitor` subcommands — they are the only supported entry
  point and handle the bootstrap ordering correctly.

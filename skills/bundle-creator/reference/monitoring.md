# Authoring Monitoring Extensions

A monitoring extension contributes observability assets to clawker's
monitoring stack: the OpenSearch indices a telemetry stream lands in, the
collector routing that sends records there, plus optional pipelines,
dashboards, and retention policies. Current schema: fetch
https://docs.clawker.dev/authoring-monitoring before walking the fields.

## Interview questions

1. **What emits the telemetry?** The `service.name` values on the records
   are the routing key — get them exactly right or nothing lands. If the
   user isn't sure, have them check what their emitter sets (OTel resource
   attributes).
2. **Logs, metrics, or both?** At least one log lane is required — an
   extension exists to land telemetry somewhere. Metrics shaping is optional.
3. **What index name(s)?** Prefix with something extension-specific
   (`acme-postgres`), never squat on clawker's own indices.
4. **Retention: shared or custom?** `default` joins clawker's shared
   retention policy — right answer for almost everyone. `custom` means the
   extension ships its own ISM policy files, scoped to its own indices only.
5. **Dashboards?** Index patterns / visualizations / dashboards exported
   from OpenSearch Dashboards, shipped as saved objects.

## Layout

```
<name>/
├── monitoring.yaml            # manifest — log lanes + metric shaping
├── index-templates/           # OpenSearch index templates (JSON)
├── ingest-pipelines/          # ingest pipelines (JSON, optional)
├── saved-objects/             # Dashboards objects (NDJSON, optional)
└── ism-policies/              # ONLY when a lane declares retention: custom
```

The `claude-code` extension shipped with clawker is the canonical working
reference for the full directory shape.

## Walkthrough order

1. **`logs` lanes** — per lane: `index` (where records land),
   `service_names` (what the collector routes there), `retention`.
2. **`metrics`** — only if the extension's metrics need collector-side
   reshaping (attribute renames). Empty `service_names` defaults to the
   union of the log lanes'.
3. **`index-templates/`** — one per index so first ingest gets correct field
   mappings. **Include real `mappings`** — a mappings-less template is
   useless and can wedge stack bring-up.
4. **Optional artifacts** — pipelines, saved objects, custom ISM policies.

## Gotchas

- **The `type` label trap:** OpenSearch's Prometheus connector errors on any
  metric series with a label literally named `type`. If the emitter uses
  one, rename it in `metrics.datapoint_renames` (e.g. `from: type`,
  `to: kind`) or the Dashboards Metrics UI can't read the series.
- **Routing is exact-match on `service.name`.** Typos fail silently — the
  records just land nowhere. Verify against the live emitter, not memory.
- **Consumption model:** projects select the extension in
  `monitor.extensions` (whole-value override across config layers, not a
  union). `clawker monitor up` seeds it at bring-up; on an already-running
  stack the apply is `clawker monitor reload`. The collector routes from the
  union of every extension ever seeded; `clawker monitor down --volumes` is
  the reset.
- Testing an extension end-to-end needs the monitoring stack running on the
  host — validation covers the manifest; the seed/routing legs need
  `clawker monitor reload` plus real telemetry flowing.

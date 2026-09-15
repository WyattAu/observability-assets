# observability-assets

Central VictoriaMetrics + Grafana assets for every shipping WyattAu build:
scrape configuration, alert rules, and dashboards — one canonical home so
metric names, alerts, and dashboards stop drifting across repos.

## Why this exists

Each service used to hand-maintain its own dashboards and alert configs
(rankhub `monitoring/`, clawdius gateway, kp_ecom `docs/grafana/`,
SimpleInfrastructureStack, ecom-engine). This repo is the single source of
truth; services keep only their *instrumentation* (metric emission), this
repo owns *consumption* (scrapes, rules, panels).

## Layout

```
victoriametrics/   vmagent scrape configs (Prometheus-compatible)
alerts/            vmalert rules (Prometheus rule format)
dashboards/        Grafana dashboard JSON, one file per service
```

## Shipping-build matrix

| Build | Repo | Metrics endpoint | Auth | Dashboard | Alerts |
|---|---|---|---|---|---|
| vane edge proxy | `WyattAu/vane` | `http://<admin>:<port>/metrics` | network-isolated admin plane | `dashboards/vane-edge.json` | `alerts/rules.yml` → vane group |
| clawdius gateway | `WyattAu/clawdius` | `http://<gateway>/metrics` | network-isolated | `dashboards/generic-rust-service.json` | generic group |
| crawlkit API | `WyattAu/crawlkit` | `:PORT/metrics` | `Authorization: Bearer <METRICS_TOKEN>` | `dashboards/generic-rust-service.json` | generic group |
| Evergreen shims | `WyattAu/EvergreenShims` | `http://<shim>:9101/metrics` | internal network | `dashboards/evergreen-shims.json` | shims group |
| ecom-engine (kp_ecom, homebite) | `git.wyattau.com/Kiln/ecom-engine` | engine `/metrics` route | `METRICS_TOKEN` | `dashboards/generic-rust-service.json` | generic group |
| cryptgpt API | `git.wyattau.com/cryptgpt/cryptgpt_mono` | `/metrics` | `METRICS_TOKEN` header | `dashboards/generic-rust-service.json` | generic group |
| rankhub API | `git.wyattau.com/Rankhub/rankhub_app` | `/api/metrics` | `METRICS_TOKEN` bearer or admin JWT | `dashboards/generic-rust-service.json` | generic group |
| kestrel / desktop clients | `WyattAu/kestrel` | none — telemetry banned by threat model §6 | — | — | — |

## Metric naming convention

`<service>_<unit>_<plural|total|seconds|bytes|info>`

- Counters end `_total`; histograms expose `_bucket/_sum/_count`; gauges are
  bare. Base unit seconds / bytes (Prometheus convention), rendered in
  dashboards with unit overrides.
- Label budget: ≤ 5 labels per series; never put unbounded values
  (IDs, paths, emails) in labels. `metrics-kit` registries enforce a series
  budget; this repo's alerts assume it held.

## Deploying

1. **vmagent**: `victoriametrics/vmagent-scrape.yml` — fill `<token>`
   placeholders from your secrets manager, then run:
   `vmagent -promscrape.config=victoriametrics/vmagent-scrape.yml`
2. **vmalert**:
   `vmalert -rule=alerts/rules.yml -datasource.url=http://victoriametrics:8428 -notifier.url=...`
3. **Grafana**: provision `dashboards/` as a file provider
   (`dashboards.yaml` provider path pointing at this checkout), datasource
   UID `victoriametrics` (see each dashboard's `datasource.uid`).

## CI

`.github/workflows/ci.yml` validates: YAML lint, JSON well-formedness,
dashboard schema sanity (`title`, `panels`, `schemaVersion`, datasource
UID), and that every alert rule references a metric documented in the
matrix above or a `metrics-kit`-emitted name.

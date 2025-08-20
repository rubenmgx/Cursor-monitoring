# Cursor Monitoring

Operational runbooks (SOPs) and Prometheus alert rules for Kubernetes platform monitoring, focused on API Server and namespace health.

## Layout

- `api-server/`
  - `docs/` — API Server dashboard SOP
  - `sops/` — Incident SOPs (CPU/memory/latency/5xx/429/requests-not-processed/TLS)
  - `alerts/` — Prometheus alert rules for API Server
- `namespaces/`
  - `docs/` — Namespace dashboard SOP
  - `sops/` — SOPs for pod restarts, availability, CPU, memory, network, disk
  - `alerts/` — Namespace-level alert rules (one file per topic)

## Dashboards

- API Server dashboard uid: `bep80qv02u6tce`
- Namespace dashboard: see `namespaces/docs/NAMESPACE_DASHBOARD_SOP.md`

## Deploying alert rules

- Prometheus (bare): copy files under `alerts/` trees to your rule_files and reload Prometheus.
- Prometheus Operator: convert each file into a `PrometheusRule`. Keep groups per file for clarity.

## Using SOPs

- Each SOP contains: detection signals, dashboard panels, PromQL pivots, and step-by-step read-only troubleshooting for hand-off.
- Navigate to the folder matching the alert (api-server or namespaces), open the SOP, and follow the steps.

## Contributing

- Follow SOP template guidance; ensure an alert exists for each SOP.
- Prefer direct links (dashboards/logs) where possible; keep steps explicit.

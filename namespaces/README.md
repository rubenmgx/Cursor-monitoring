# Namespaces

Namespace-level dashboards, SOPs, and alerts for workload health per namespace.

## Contents

- `docs/`
  - `NAMESPACE_DASHBOARD_SOP.md`: How to use the namespace dashboard (panels and common investigations)
- `sops/`
  - `NAMESPACE_POD_RESTARTS.md`: Investigate container restarts
  - `NAMESPACE_AVAILABILITY.md`: Investigate availability dips (Konflux_up)
  - `NAMESPACE_CPU_HIGH.md`: CPU above 90% of requests/limits
  - `NAMESPACE_MEMORY_HIGH.md`: Memory above 90% of requests/limits
  - `NAMESPACE_NETWORK.md`: Network anomalies (bytes/packets)
  - `NAMESPACE_DISK.md`: Disk IO anomalies
- `alerts/`
  - `pod_restarts.yaml`, `availability.yaml`, `cpu_high.yaml`, `memory_high.yaml`, `network_spike.yaml`, `disk_io_high.yaml`

## Deploying alerts

- Prometheus: add files under `alerts/` to your rule files and reload.
- Prometheus Operator: wrap each `groups:` list into a `PrometheusRule` resource.

## Using the SOPs

- Each SOP provides panels to check, PromQL pivots, and read-only steps to capture evidence for hand-off.
- Start from the alert name, open the matching SOP under `sops/`, and follow the steps.

## Thresholds

- CPU/Memory thresholds default to 90% of limits; adjust per environment.
- Network/Disk thresholds are example values; tune to your baseline.

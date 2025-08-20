# Namespace Disk IO Anomalies

## Description
Elevated disk read/write throughput at the namespace level indicating IO pressure or contention.

## Detection
- Dashboard: Disk Reads/Writes Per Second
- Metrics:
  - `sum(rate(container_fs_reads_bytes_total{namespace="<ns>"}[5m]))`
  - `sum(rate(container_fs_writes_bytes_total{namespace="<ns>"}[5m]))`

## Resolution (Read-only investigation)
1) Identify top IO consumers
```promql
topk(10, sum by (pod, container) (rate(container_fs_writes_bytes_total{namespace="<ns>"}[5m])))
```
2) Hand-off
- Provide top IO pods/containers and time window; note any concurrent latency in app.

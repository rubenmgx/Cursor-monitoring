### Konflux Namespace Dashboard SOP

This SOP explains how to use and interpret the "rugomez - Konflux namespace dashboard" for namespace-level health, capacity, and performance.

---

## Purpose
- Monitor availability, deployment readiness, stability (restarts), and resource usage per namespace.
- Provide quick triage and comparison across namespaces and clusters.

## Audience
- SREs, platform engineers, application owners.

## Prerequisites
- Grafana access to the dashboard `rugomez - Konflux namespace dashboard` (uid: `eetkzyh60ix34e`).
- Prometheus datasource with Kubernetes metrics (kube-state-metrics, cAdvisor/Kubelet).

## Dashboard Variables
- **datasource**: Prometheus datasource (filtered to `rhtap*`).
- **cluster**: One or more clusters via `source_cluster` (supports multi-select and All).
- **namespace**: One or more namespaces (supports multi-select and All).

Tips:
- Use "All" for a fleet view, then narrow to specific namespaces.
- Compare multiple namespaces to spot outliers quickly.

## Time Range
- Default: last 1h. For trends use 6–24h; for spikes use 5–15m.

---

## Panel Guide

### Availability
- **Availability / Konflux_up Signal (timeseries)**: Monitors the percentage of time `konflux_up{namespace}` is 1 within the window. Use as a quick availability indicator for the namespace service.
- **Number of Desired vs Actual Replicas / Namespace (stat)**: Monitors ratio of desired to available Deployment replicas. Values < 1 indicate pending/unavailable pods.
- **Pod Restarts / Namespace (stat)**: Monitors recent increases in container restarts per namespace to surface instability and crash loops.

### CPU / Memory
- **CPU Utilisation (from requests) (stat)**: Monitors actual CPU usage vs requested CPU across selected namespaces to understand how close workloads are to their requested capacity.
- **CPU Utilisation (from limits) (stat)**: Monitors actual CPU usage vs CPU limits to gauge throttling risk and headroom.
- **CPU Usage Percentage (stat)**: Monitors overall CPU usage percentage for selected namespaces (approximate utilization relative to cluster CPU capacity over time).
- **CPU Usage (timeseries)**: Monitors CPU cores used per namespace over time to visualize trends and bursts.
- **Memory Utilisation (from requests) (stat)**: Monitors memory working set vs requested memory, indicating how close workloads are to requests.
- **Memory Utilisation (from limits) (stat)**: Monitors memory working set vs memory limits to identify risk of OOM kills.
- **Memory Usage (timeseries)**: Monitors bytes of memory used per namespace over time for growth and spikes.

### Network / Disk
- **Network Transmit / Receive Bytes Per Second (timeseries)**: Monitors namespace-level egress/ingress throughput to detect anomalies and traffic surges.
- **Network Inbound / Outbound Packets (timeseries)**: Monitors packet rates to identify packet loss patterns or small-packet floods.
- **Disk Reads/Writes Per Second (timeseries)**: Monitors per-namespace disk IO throughput to spot IO-heavy behavior and potential contention.

---

## Common Investigations (Playbooks)

1) Availability dips
- Check in dashboard: "Availability / Konflux_up Signal" for drops; correlate with "Desired vs Actual Replicas" and "Pod Restarts".
- Relevant metrics: `konflux_up{namespace}`, `kube_deployment_status_replicas_available`, `kube_deployment_spec_replicas`, `kube_pod_container_status_restarts_total`.
- Clues: Dips with replica shortfall → scaling/scheduling issues; dips with restarts → crashes or resource pressure.

2) Under-replicated deployments
- Check in dashboard: "Desired vs Actual Replicas" < 1. Also review CPU/Memory Utilisation to see if capacity limits block scheduling.
- Relevant metrics: `kube_deployment_status_replicas_available`, `kube_deployment_spec_replicas`, request/limit ratios from panels.
- Clues: Adequate resources but replicas missing → image pull/backoff or failing readiness; high utilisation → quotas/requests too low.

3) Crash loops / instability
- Check in dashboard: "Pod Restarts / Namespace" > 0; correlate with CPU/Memory Utilisation and Usage timeseries for spikes.
- Relevant metrics: `kube_pod_container_status_restarts_total`, `container_memory_working_set_bytes`, `node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate`.
- Clues: Restarts with high memory → OOM risk; with high CPU limits ratio → throttling or hot path.

4) CPU saturation from requests
- Check in dashboard: "CPU Utilisation (from requests)" and "CPU Usage (timeseries)" for sustained high values.
- Relevant metrics: `node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate`, `cluster:namespace:pod_cpu:active:kube_pod_container_resource_requests`.
- Clues: > 100% of requests means requests set too low vs actual; adjust requests or optimize workload.

5) CPU at limits / throttling risk
- Check in dashboard: "CPU Utilisation (from limits)" and peaks in "CPU Usage (timeseries)".
- Relevant metrics: `cluster:namespace:pod_cpu:active:kube_pod_container_resource_limits`, `node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate`, and if available `container_cpu_cfs_throttled_seconds_total` (rate).
- Clues: High limit utilisation + throttling counters rising → increase limits or reduce concurrency.

6) Memory pressure
- Check in dashboard: "Memory Utilisation (from limits/requests)" and "Memory Usage (timeseries)" for growth or spikes.
- Relevant metrics: `container_memory_working_set_bytes`, `cluster:namespace:pod_memory:active:kube_pod_container_resource_{requests,limits}`, `namespace:container_memory_usage_bytes:sum`.
- Clues: > 90% of limits with upward trend → OOM risk; right-size limits or fix leaks/caches.

7) Hot namespace identification
- Check in dashboard: "CPU Usage" and "Memory Usage" timeseries to find top consumers; compare across namespaces.
- Relevant metrics: `namespace:container_cpu_usage:sum`, `namespace:container_memory_usage_bytes:sum`.
- Clues: Same namespace also tops Network/Disk panels → broad resource intensity.

8) Network anomalies
- Check in dashboard: "Network Transmit/Receive Bytes" and "Inbound/Outbound Packets" for sudden spikes or asymmetry.
- Relevant metrics: `container_network_{transmit,receive}_bytes_total`, `container_network_{transmit,receive}_packets_total` (rates).
- Clues: High bytes with low packets → large payloads; high packets with modest bytes → many small requests/retries.

9) IO contention
- Check in dashboard: "Disk Reads/Writes Per Second" for step increases during latency complaints.
- Relevant metrics: `container_fs_reads_bytes_total`, `container_fs_writes_bytes_total` (rates) by namespace.
- Clues: Sustained high IO aligns with app slowdowns → investigate storage class/node disk health.

10) Post-deploy regression
- Check in dashboard: widen time range across deploy; compare Utilisation, Usage, Restarts before vs after.
- Relevant metrics: same as above; focus on deltas in CPU/Memory/Network and restarts post-change.
- Clues: Clear step-change after rollout → rollback or hotfix while root-causing.

---

## Operating Notes
- CPU units: cores (timeseries) and percentages vs requests/limits (stat panels).
- Memory units: bytes (timeseries) and percentages vs requests/limits (stat panels).
- Legends: Click to toggle; Shift-click to isolate a series.
- Performance: For slow loads, prefer recorded metrics for aggregation-heavy expressions.

## Troubleshooting
- No data: verify `datasource`, `cluster`, and `namespace` selections and metric presence.
- Noisy charts: widen the time range or focus on one namespace at a time.
- Ratios > 100%: indicates requests/limits set too low vs actual usage.

---

Maintainer: rugomez
Last updated: <set during change>

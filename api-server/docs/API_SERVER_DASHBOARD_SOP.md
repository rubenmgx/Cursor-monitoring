### API Server Dashboard SOP

This SOP explains how to use and interpret the "rugomez - API SERVER testing" dashboard for Kubernetes API Server health, performance, and error analysis.

---

## Purpose
- Provide a fast, consistent way to assess API Server availability, traffic, latency, saturation, and errors across clusters.
- Guide responders through common investigations using the panels provided.

## Audience
- SREs, platform engineers, and on-call responders.

## Prerequisites
- Access to Grafana with the dashboard titled `rugomez - API SERVER testing` (uid: `bep80qv02u6tce`).
- Prometheus datasource with API Server and node metrics.

## Dashboard Variables
- **datasource**: Prometheus datasource (filtered to `rhtap*`).
- **cluster**: One or more clusters (`source_cluster` label). Supports multi-select and "All".

Tips:
- Use multi-select to compare clusters side-by-side.
- When isolating a cluster, de-select others for clearer panels and legends.

## Time Range
Default is last 1h. Adjust based on incident timeline:
- 5–15m for acute issues, 6–24h for trend and regression.

---

## Panel Guide

### Availability
- **Konflux Up Availability Signal (stat)**: Monitors the percentage of time the API is reported up via `konflux_up{service="api-server"}` over the selected window. Use it as a quick availability indicator.

### Load and Saturation
- **Requests being processed by API Server (barchart)**: Monitors concurrent inflight requests (`apiserver_current_inflight_requests`). Highlights spikes and sustained load that can indicate saturation.
- **CPU Usage by Cluster (stat)**: Monitors average CPU consumption of the apiserver job at the cluster level (derived from CPU seconds). Useful for tracking overall load trends.
- **CPU Usage by Instance (timeseries)**: Monitors CPU usage per apiserver instance to identify hot or underutilized replicas.
- **Memory Usage by Cluster (stat)**: Monitors percentage of node memory consumed by apiserver processes across the cluster.
- **Memory Usage by Instance (timeseries)**: Monitors resident memory per apiserver instance over time to spot growth or step changes.

### Latency
- **P95 Latency by Verb (stat)**: Monitors the 95th percentile request latency grouped by `verb` (GET/LIST/WATCH/POST/etc.). Useful to see which operations are slow.
- **API Server latency (stat)**: Monitors overall high-percentile latency across verbs for a top-level health signal.

### Traffic and Errors
- **API Server Request Rate by Instance (timeseries)**: Monitors request rate per apiserver instance to check load distribution and uneven traffic.
- **API Server Request Rate by Status (piechart)**: Monitors share of 2xx/3xx/4xx/5xx responses within the time window.
- **API Server Request Rate by Status (timeseries)**: Monitors the rate of responses over time by HTTP status to see when errors occur.
- **Verb count by Cluster (bar gauge)**: Monitors how many distinct verbs are active in the selected window to understand workload mix.
- **API Server - Request Rate by Resource and Verb (timeseries)**: Monitors request rate by `resource` and `verb` to identify noisy or expensive endpoints.
- **Write / Read Requests (timeseries)**: Monitors read vs write request rates per cluster to understand operation mix and pressure on mutating paths.
- **Rate TLS handshake error (timeseries)**: Monitors rate of TLS handshake errors to surface client TLS/cert configuration problems.

---

## Common Investigations (Playbooks)

### 1) Latency spike
- Check in dashboard: "P95 Latency by Verb" to identify affected verbs; "API Server latency" for overall signal; "Requests being processed" for inflight spikes; CPU/Memory panels for saturation; "Request Rate by Status" for concurrent errors.
- Relevant metrics: `apiserver_request_duration_seconds_bucket` (via `histogram_quantile`), `apiserver_current_inflight_requests`, `apiserver_request_total{code=...}` (rates), `process_cpu_seconds_total` (rate), `process_resident_memory_bytes`.
- Clues: Verb-specific p95/p99 rises, inflight rising, and 5xx growth indicate slow handlers or backend pressure.

### 2) Surge in 5xx/4xx
- Check in dashboard: "Request Rate by Status" (pie + timeseries) to size and time errors; "API Server - Request Rate by Resource and Verb" to see which endpoints are involved; "Rate TLS handshake error" to rule out TLS issues.
- Relevant metrics: `apiserver_request_total{code=~"4..|5.."}` (rates), `code:apiserver_request_total:rate5m`, `resource_verb:apiserver_request_total:rate5m`, `apiserver_tls_handshake_errors_total` (rate).
- Clues: 5xx → backend/admission/storage issues; 429 → throttling; 401/403 → authn/authz changes.

### 3) Suspected saturation
- Check in dashboard: "Requests being processed" for sustained growth; "CPU Usage by Instance" and "Memory Usage by Instance" for hotspots; latency panels for impact.
- Relevant metrics: `apiserver_current_inflight_requests`, `rate(process_cpu_seconds_total[5m])` by instance, `process_resident_memory_bytes` (current/avg_over_time).
- Clues: Inflight rising with latency while RPS constant → slowdowns; CPU hotspot hints at uneven work or heavy requests.

### 4) Cluster comparison
- Check in dashboard: Multi-select `cluster`; compare "API Server Request Rate by Instance", CPU/Memory by instance, and "Request Rate by Status" across clusters.
- Relevant metrics: `instance:apiserver_request_total:rate5m`, `rate(process_cpu_seconds_total[5m])`, `process_resident_memory_bytes`, `code:apiserver_request_total:rate5m`.
- Clues: One cluster dominating errors or latency suggests localized issues (LB, network, etcd).

---

### 5) TLS handshake errors present
- Check in dashboard: "Rate TLS handshake error" for spikes; correlate with "Request Rate by Status".
- Relevant metrics: `apiserver_tls_handshake_errors_total` (rate), `apiserver_request_total{code=~"4..|5.."}` (rates).
- Clues: Spikes around cert rotations or CA changes; expect client retries and elevated 4xx/5xx.

### 6) 429 throttling (priority/fairness)
- Check in dashboard: "Request Rate by Status" focusing on 429; "Write / Read Requests" to confirm write-heavy periods; "Requests being processed" for saturation.
- Relevant metrics: `apiserver_request_total{code="429"}` (rate), `write:apiserver_request_total:rate5m`, `read:apiserver_request_total:rate5m`, `apiserver_current_inflight_requests`. If available: `apiserver_flowcontrol_*`.
- Clues: Write surges plus 429 indicate throttling; consider client backoff or APF tuning.

### 7) CPU hotspot on a single instance
- Check in dashboard: "CPU Usage by Instance" for outliers; cross-check with "API Server Request Rate by Instance".
- Relevant metrics: `rate(process_cpu_seconds_total[5m]) by (instance)`, `instance:apiserver_request_total:rate5m`.
- Clues: High CPU without high RPS → expensive requests; high RPS on one instance → LB skew or endpoint readiness.

### 8) Memory growth or leak
- Check in dashboard: "Memory Usage by Instance" for monotonic growth; cluster-level memory stat for context; latency for impact.
- Relevant metrics: `process_resident_memory_bytes` (current and `avg_over_time`), `namespace:container_memory_usage_bytes:sum` (if available).
- Clues: Step increases after events or sustained growth imply leaks/caches; GC pressure correlates with latency.

### 9) Load imbalance across instances
- Check in dashboard: "API Server Request Rate by Instance" for uneven distribution; CPU/Memory by instance for secondary effects.
- Relevant metrics: `instance:apiserver_request_total:rate5m`, `rate(process_cpu_seconds_total[5m])`.
- Clues: Persistent skew → LB/endpoints issue; sudden skew → pod readiness transitions.

### 10) Auth failures (401/403)
- Check in dashboard: "Request Rate by Status" filtering 401/403; use "API Server - Request Rate by Resource and Verb" to identify impacted endpoints.
- Relevant metrics: `apiserver_request_total{code=~"401|403"}` (rates), `resource_verb:apiserver_request_total:rate5m`.
- Clues: Spikes after RBAC/policy changes or token rotation; often localized to specific resources.

### 11) Inflight stalls without RPS growth
- Check in dashboard: Rising "Requests being processed" with flat "Request Rate by Instance" and deteriorating latency.
- Relevant metrics: `apiserver_current_inflight_requests`, `instance:apiserver_request_total:rate5m` (flat), latency quantiles.
- Clues: Indicates slow handlers/backends (admission webhooks, etcd); look for correlated 5xx or timeouts.

### 12) Post-deploy regression
- Check in dashboard: Increase time range across deploy; compare p95 latency, status shares, and resource/verb rates before vs after.
- Relevant metrics: Latency quantiles from `apiserver_request_duration_seconds_bucket`, `code:apiserver_request_total:rate5m`, `resource_verb:apiserver_request_total:rate5m`.
- Clues: Clear step-change after rollout pinpoints regression surface; roll back while root causing.

### 13) Availability dips with normal metrics
- Check in dashboard: Availability stat dips while latency/errors are stable; inspect TLS error rate and external path health.
- Relevant metrics: `konflux_up{service="api-server"}`, `apiserver_tls_handshake_errors_total` (rate).
- Clues: External reachability or TLS trust issues rather than server-side saturation.

### 14) Read vs write pressure shift
- Check in dashboard: "Write / Read Requests" for write surges; correlate with latency by verb (mutations) and inflight.
- Relevant metrics: `write:apiserver_request_total:rate5m`, `read:apiserver_request_total:rate5m`, latency quantiles.
- Clues: Write-heavy periods increase etcd load and often elevate p95/p99 for mutating verbs.

## Operating Notes
- Units: CPU panels reflect consumption derived from CPU seconds (effectively cores); memory is bytes/percent; latency is seconds.
- Legends: Click entries to toggle; Shift-click to isolate a series.
- Windowing: Some panels use 5m rates/averages. Prefer 15–60m windows for stable comparisons.
- Performance: If charts load slowly, consider adding recording rules for expensive aggregations in Prometheus.

## Troubleshooting
- No data:
  - Confirm `datasource` selection and permissions.
  - Ensure `source_cluster` labels exist and align with variable values.
- Spiky/Noisy signals:
  - Increase time range or rely on averaged views (5m) panels.
- Mismatched trends:
  - Ensure you are comparing the same clusters across panels (multi-select can hide differences).

## References
- Kubernetes API Server metrics: apiserver_* (latency, inflight, request totals)
- TLS errors: `apiserver_tls_handshake_errors_total`
- Process metrics: `process_cpu_seconds_total`, `process_resident_memory_bytes`

---

Maintainer: rugomez
Last updated: <set during change>

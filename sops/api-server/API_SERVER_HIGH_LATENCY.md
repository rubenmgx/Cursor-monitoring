# API Server High Latency

## Developer Communications

- [PROJECT on JIRA](Project URL)
- [#channel on Slack](Channel URL)

## Description

API Server 95th/99th percentile latency elevated beyond SLO (e.g., p95 > 1s or p99 > 2s), impacting client operations and reconcile loops.

## Pre-requisites

- Access to Grafana dashboard `rugomez - API SERVER testing` (uid: `bep80qv02u6tce`)
- Access to Prometheus for ad-hoc queries

## Detection

### User feedback
- Reports of timeouts in kubectl/oc, controllers lagging, or UI slowness.

### Monitoring
- Dashboard panels to check:
  - P95 Latency by Verb; API Server latency; Requests being processed; Request Rate by Status; Request Rate by Resource and Verb
- Key metrics/queries:
  - Latency quantiles:
    - `histogram_quantile(0.95, sum by (le, verb)(rate(apiserver_request_duration_seconds_bucket[5m])))`
    - `histogram_quantile(0.99, sum by (le)(rate(apiserver_request_duration_seconds_bucket[5m])))`
  - Inflight: `apiserver_current_inflight_requests`

## Resolution

1) Identify affected verbs/resources (dashboard)
- Note spikes in P95 by Verb; cross-reference with Request Rate by Resource and Verb to find noisy endpoints.

2) Check for saturation vs endpoint-specific slowness
- If inflight is rising and CPU/memory are high, the issue is global/saturation.
- If only specific verbs/resources are slow, the issue is localized.

3) PromQL pivots
```promql
# Per-verb p95
histogram_quantile(0.95, sum by (le, verb) (rate(apiserver_request_duration_seconds_bucket[5m])))
# Per-resource+verb p95
histogram_quantile(0.95, sum by (le, resource, verb) (rate(apiserver_request_duration_seconds_bucket[5m])))
```

4) Actions based on findings
- Localized (single resource/verb):
  - Inspect admission webhooks bound to that resource; check webhook service latency.
  - Optimize or scale the dependent service; reduce request rate temporarily.
- Global (all verbs):
  - Scale apiserver capacity, reduce client concurrency, or offload heavy list/watch clients.
  - Investigate etcd/storage latency on infra dashboards.

5) Validate recovery
- Observe p95/p99 returning to baseline for at least 30 minutes.

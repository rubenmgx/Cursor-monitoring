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

1. Identify affected verbs/resources from the panels above.
2. Check inflight and CPU/Memory to determine saturation vs endpoint-specific slowness.
3. If isolated to a resource/verb:
   - Inspect admission webhooks and downstream dependencies for that path.
   - Mitigate by reducing request rate or scaling dependent services.
4. If global:
   - Scale apiserver or optimize heavy clients; investigate etcd/storage latency on infra dashboards.
5. Monitor until latency returns to baseline for 30m.

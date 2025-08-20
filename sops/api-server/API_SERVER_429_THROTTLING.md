# API Server 429 Throttling (Priority/Fairness)

## Developer Communications

- [PROJECT on JIRA](Project URL)
- [#channel on Slack](Channel URL)

## Description

HTTP 429 responses indicate requests are being throttled, commonly due to priority and fairness (APF) or client-side concurrency exceeding server capacity.

## Pre-requisites

- Access to Grafana dashboard `rugomez - API SERVER testing` (uid: `bep80qv02u6tce`)
- Access to Prometheus for ad-hoc queries

## Detection

### User feedback
- Clients observe rate-limit errors (429) and increased retries.

### Monitoring
- Dashboard panels to check:
  - Request Rate by Status; Write / Read Requests; Requests being processed
- Key metrics/queries:
  - `sum(rate(apiserver_request_total{code="429"}[5m]))`
  - `write:apiserver_request_total:rate5m`, `read:apiserver_request_total:rate5m`

## Resolution (Read-only investigation)

1) Confirm throttling and workload mix
- In dashboard: confirm elevated 429; check if write-heavy via Write/Read panel.
- PromQL quick checks:
```promql
sum(rate(apiserver_request_total{code="429"}[5m]))
sum by (verb) (rate(apiserver_request_total[5m]))
```

2) APF/flow-control insight (if metrics available)
```promql
# Queue wait time p95 by priority level
histogram_quantile(0.95, sum by (le, priority_level) (rate(apiserver_flowcontrol_request_wait_duration_seconds_bucket[5m])))
# Rejected requests by reason/priority
sum by (reason, priority_level) (rate(apiserver_flowcontrol_rejected_requests_total[5m]))
```

3) Backend context (read-only)
```promql
# Etcd p95 to see if backend contributes to throttling
histogram_quantile(0.95, sum by (le, operation) (rate(etcd_request_duration_seconds_bucket[5m])))
```

4) Hand-off notes
- Provide: 429 rate, verb mix (read vs write), APF queue wait p95 and rejections (if present), and etcd p95 context.

5) Validate recovery
- 429 rate declines and latency stabilizes.

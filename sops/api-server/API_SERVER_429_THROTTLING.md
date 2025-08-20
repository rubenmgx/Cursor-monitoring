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

## Resolution

1) Confirm throttling and workload mix
- In dashboard: confirm elevated 429; check if write-heavy via Write/Read panel.
- PromQL quick checks:
```promql
sum(rate(apiserver_request_total{code="429"}[5m]))
sum by (verb) (rate(apiserver_request_total[5m]))
```

2) Reduce pressure from heavy clients
- Lower concurrency and implement exponential backoff.

3) Consider APF tuning or capacity
- Adjust priority levels/queues or increase apiserver capacity (replicas/resources).

4) Validate recovery
- 429 rate declines and latency stabilizes.

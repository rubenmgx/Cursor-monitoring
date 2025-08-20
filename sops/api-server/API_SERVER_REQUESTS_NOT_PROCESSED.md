# API Server Requests Not Being Processed

## Developer Communications

- [PROJECT on JIRA](Project URL)
- [#channel on Slack](Channel URL)

## Description

API Server appears to stop processing requests (request rate drops to near zero) or requests are stuck inflight for prolonged periods.

## Pre-requisites

- Access to Grafana dashboard `rugomez - API SERVER testing` (uid: `bep80qv02u6tce`)
- Access to Prometheus for ad-hoc queries

## Detection

### User feedback
- Timeouts and stalled kubectl/oc commands; controllers not making progress.

### Monitoring
- Dashboard panels to check:
  - API Server Request Rate by Instance; Requests being processed; API Server latency
- Key metrics/queries:
  - Total request rate:
    - `sum(rate(apiserver_request_total[5m]))`
  - Inflight requests:
    - `sum(apiserver_current_inflight_requests)`

## Resolution

1. If request rate ~0 across instances:
   - Verify apiserver availability signal and pod health; restart unhealthy pods if needed.
   - Check control-plane networking and load balancer.
2. If inflight rising while rate flat/low:
   - Indicates stuck handlers; inspect admission webhooks and storage (etcd) health.
   - Reduce client concurrency; restart the most impacted apiserver pod as a stopgap.
3. Confirm recovery by observing request rate normalization and inflight returning to baseline.

# API Server Elevated 5xx Error Rate

## Developer Communications

- [PROJECT on JIRA](Project URL)
- [#channel on Slack](Channel URL)

## Description

HTTP 5xx responses are elevated, indicating server-side failures or downstream dependency issues (e.g., admission webhooks, storage, or timeouts).

## Pre-requisites

- Access to Grafana dashboard `rugomez - API SERVER testing` (uid: `bep80qv02u6tce`)
- Access to Prometheus for ad-hoc queries

## Detection

### User feedback
- Server errors and failures across clients; retries with 5xx codes.

### Monitoring
- Dashboard panels to check:
  - Request Rate by Status (pie + timeseries); Request Rate by Resource and Verb; API Server latency
- Key metrics/queries:
  - `sum by (code)(rate(apiserver_request_total{code=~"5.."}[5m]))`
  - Top resources/verbs: `topk(10, sum by (resource,verb)(rate(apiserver_request_total{code=~"5.."}[5m])))`

## Resolution

1. Identify top failing resources/verbs; inspect related admission webhooks and backend services.
2. Check latency panels for timeouts vs immediate failures.
3. Roll back recent changes impacting the failing endpoints; adjust timeouts if downstream is slow.
4. Monitor 5xx share returning to baseline and latency normalization.

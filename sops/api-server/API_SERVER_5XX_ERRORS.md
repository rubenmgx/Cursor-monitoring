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

1) Identify failing endpoints
- In dashboard: isolate which resources/verbs lead errors.
- PromQL drill-down:
```promql
sum by (resource,verb) (rate(apiserver_request_total{code=~"5.."}[5m]))
```

2) Correlate with latency and timeouts
- If p95/p99 elevated, errors may be timeout-related.

3) Admission webhooks and dependencies
```bash
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations -A | cat
```
- Check logs of implicated webhooks/services.

4) Roll back or hotfix
- Revert recent changes affecting failing endpoints; increase timeouts if backend is slow.

5) Validate recovery
- 5xx share returns to baseline and latency normalizes.

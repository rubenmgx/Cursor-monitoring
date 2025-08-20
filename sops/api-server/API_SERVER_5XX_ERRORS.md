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

## Resolution (Read-only investigation)

1) Identify failing endpoints
- In dashboard: isolate which resources/verbs lead errors.
- PromQL drill-down:
```promql
sum by (resource,verb) (rate(apiserver_request_total{code=~"5.."}[5m]))
```

2) Admission controller checks (Kyverno, etcd-shield)
```promql
# Webhook rejection counts by name (if exposed)
sum by (name) (rate(apiserver_admission_webhook_rejection_count[5m]))
# Admission controller duration p95 by type/name
histogram_quantile(0.95, sum by (le, type, name) (rate(apiserver_admission_controller_admission_duration_seconds_bucket[5m])))
# Kyverno rules outcome
sum by (policy, rule, result) (rate(kyverno_policy_rule_results_total[5m]))
```
- Read-only cluster info:
```bash
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations -A
kubectl get cpol -A || true; kubectl get pol -A || true
```

3) Backend (etcd) health
```promql
histogram_quantile(0.95, sum by (le, operation) (rate(etcd_request_duration_seconds_bucket[5m])))
max(etcd_server_has_leader)
```

4) Hand-off notes
- Provide: failing resources/verbs, any implicated webhooks/policies, and etcd latency/leader status around the incident window.

5) Validate recovery
- 5xx share returns to baseline and latency normalizes.

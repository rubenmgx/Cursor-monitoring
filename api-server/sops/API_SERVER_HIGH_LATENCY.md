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

## Resolution (Read-only investigation)

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

4) Admission controllers (Kyverno, etcd-shield) checks
- PromQL (K8s admission webhook/controller metrics):
```promql
# Admission controller duration by type/name (look for ValidatingAdmissionWebhook/MutatingAdmissionWebhook)
histogram_quantile(0.95, sum by (le, type, name) (rate(apiserver_admission_controller_admission_duration_seconds_bucket[5m])))
# Webhook rejections (if exposed)
sum by (name) (rate(apiserver_admission_webhook_rejection_count[5m]))
```
- Kyverno metrics (if scraped):
```promql
# Kyverno admission review latency p95
histogram_quantile(0.95, sum by (le) (rate(kyverno_admission_review_duration_seconds_bucket[5m])))
# Rule results by outcome
sum by (policy, rule, result) (rate(kyverno_policy_rule_results_total[5m]))
```
- Read-only cluster info:
```bash
# List admission webhooks
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations -A
# List Kyverno policies (ClusterPolicy/Policy)
kubectl get cpol -A || true
kubectl get pol -A || true
```

5) Backend (etcd) checks
- PromQL:
```promql
# Etcd request p95
histogram_quantile(0.95, sum by (le, operation) (rate(etcd_request_duration_seconds_bucket[5m])))
# Apiserver storage operation latency p95 (older metric name may vary)
histogram_quantile(0.95, sum by (le, verb, resource) (rate(apiserver_storage_operation_duration_seconds_bucket[5m])))
# Leader health and DB size
max(etcd_server_has_leader)
max(etcd_mvcc_db_total_size_in_bytes)
```

6) Actions for responder (hand-off notes)
- If admission is slow (Kyverno/etcd-shield): provide top slow webhook names, affected policies/rules, and p95 values to the owning team.
- If etcd is slow: provide etcd p95 latency, DB size trends, and any spikes around incident window to the platform team.
- If saturation: provide inflight vs RPS and CPU/memory context for capacity actions.

7) Validate recovery
- Observe p95/p99 returning to baseline for at least 30 minutes.

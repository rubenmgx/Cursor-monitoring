# API Server CPU Usage Above 90%

## Developer Communications

- [PROJECT on JIRA](Project URL)
- [#channel on Slack](Channel URL)

## Description

API Server CPU usage is sustained above 90% of configured CPU limits or requested capacity. This leads to throttling, elevated latency, and potential request timeouts.

## Pre-requisites

- Access to Grafana dashboard `rugomez - API SERVER testing` (uid: `bep80qv02u6tce`)
- Access to Prometheus for ad-hoc queries
- `kubectl` access to control-plane namespace hosting kube-apiserver
- Access to logs (Loki/Splunk) if needed

## Detection

### User feedback
- Slow kubectl/oc operations, increasing timeouts, or controllers falling behind.

### Monitoring
- Dashboard panels to check:
  - CPU Usage by Instance; CPU Usage by Cluster; Requests being processed; API Server latency; Request Rate by Instance
- Key metrics/queries:
  - CPU vs limits per instance:
    - `sum by (instance,source_cluster)(rate(container_cpu_usage_seconds_total{pod=~"kube-apiserver-.*"}[5m])) / sum by (instance,source_cluster)(kube_pod_container_resource_limits{resource="cpu", unit="core", pod=~"kube-apiserver-.*"})`
  - Load context:
    - `instance:apiserver_request_total:rate5m`, `apiserver_current_inflight_requests`

## Resolution

1. Identify hot instances on CPU by Instance; confirm with per-instance CPU vs limits query.
2. Check Request Rate by Instance for traffic skew; if skewed, verify endpoints and load balancer health.
3. If CPU-bound:
   - Increase kube-apiserver CPU limits or add replicas (if your control plane supports it).
   - Reduce client concurrency/backoff for heavy controllers/clients.
4. Reduce expensive operations:
   - Use Request Rate by Resource and Verb and latency panels to find slow/noisy endpoints.
   - Audit admission webhooks and LIST/WATCH consumers.
5. Validate recovery: CPU < 80% sustained and latency normalizes over 10–30m.

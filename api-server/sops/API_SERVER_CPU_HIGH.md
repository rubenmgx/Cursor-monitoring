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

1) Locate hot instances and confirm load vs skew
- In dashboard: open "CPU Usage by Instance" and "API Server Request Rate by Instance".
- Shell checks (adjust namespace as needed):
```bash
NS=${APISERVER_NS:-openshift-kube-apiserver}
# If not OpenShift, try kube-system
kubectl get pods -n $NS -l component=kube-apiserver -o wide || kubectl get pods -n kube-system -l component=kube-apiserver -o wide
# If metrics-server is installed
kubectl top pod -n $NS -l component=kube-apiserver || true
# Confirm apiserver backends behind the default service
kubectl -n default get endpoints kubernetes -o wide
```

2) Verify readiness and health
```bash
# API readiness and liveness from inside the cluster
kubectl get --raw='/readyz?verbose' | head -100
kubectl get --raw='/livez?verbose' | head -100
```

3) Quantify CPU vs limits in Prometheus/Grafana Explore
```promql
sum by (instance,source_cluster)(rate(container_cpu_usage_seconds_total{pod=~"kube-apiserver-.*"}[5m]))
/
sum by (instance,source_cluster)(kube_pod_container_resource_limits{resource="cpu", unit="core", pod=~"kube-apiserver-.*"})
```

4) Mitigations
- If skewed traffic: investigate endpoints/LB; unhealthy pod may be excluded.
```bash
# Check pod conditions and recent events
kubectl -n $NS describe pod -l component=kube-apiserver | sed -n '1,200p'
kubectl -n $NS get events --sort-by=.lastTimestamp | tail -n 50
```
- If truly CPU-bound:
  - Increase kube-apiserver CPU limits via your control-plane operator or manifests.
  - Reduce client concurrency/backoff for heavy controllers.

5) Identify expensive endpoints
- In dashboard: "API Server - Request Rate by Resource and Verb" and latency panels.
- PromQL for top heavy endpoints:
```promql
topk(10, sum by (resource, verb) (rate(apiserver_request_total[5m])))
```

6) Validate recovery
- CPU drops < 80% sustained; latency returns to baseline over 10–30m.

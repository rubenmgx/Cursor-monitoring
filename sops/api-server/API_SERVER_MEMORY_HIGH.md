# API Server Memory Usage Above 90%

## Developer Communications

- [PROJECT on JIRA](Project URL)
- [#channel on Slack](Channel URL)

## Description

API Server memory usage is sustained above 90% of configured limits or shows monotonic growth, risking OOM kills and elevated latency.

## Pre-requisites

- Access to Grafana dashboard `rugomez - API SERVER testing` (uid: `bep80qv02u6tce`)
- Access to Prometheus for ad-hoc queries
- `kubectl` access to control-plane namespace

## Detection

### User feedback
- API becomes sluggish or restarts observed; controllers see connection resets.

### Monitoring
- Dashboard panels to check:
  - Memory Usage by Instance; Memory Usage by Cluster; API Server latency; Requests being processed
- Key metrics/queries:
  - Memory vs limits per instance:
    - `process_resident_memory_bytes{job="apiserver"}` vs `kube_pod_container_resource_limits{resource="memory", unit="byte", pod=~"kube-apiserver-.*"}`
  - Smoothed memory per instance:
    - `avg_over_time(process_resident_memory_bytes{job="apiserver"}[5m])`

## Resolution

1) Verify pod health and restart history
```bash
NS=${APISERVER_NS:-openshift-kube-apiserver}
kubectl -n $NS get pods -l component=kube-apiserver -o wide
kubectl -n $NS describe pod -l component=kube-apiserver | sed -n '1,200p'
```

2) Quantify memory vs limits
```promql
sum by (instance,source_cluster) (process_resident_memory_bytes{job="apiserver"})
/
sum by (instance,source_cluster) (kube_pod_container_resource_limits{resource="memory", unit="byte", pod=~"kube-apiserver-.*"})
```

3) Look for growth patterns
```promql
avg_over_time(process_resident_memory_bytes{job="apiserver"}[30m])
```

4) Mitigations
- If close to limits, increase memory limits for kube-apiserver.
- If monotonic growth, restart the most impacted pod as a stopgap while investigating.

5) Investigate likely causes
- Large LIST/WATCH or cache growth:
```promql
topk(10, sum by (resource, verb) (rate(apiserver_request_total[5m])))
```
- Admission webhook overhead: correlate high latency on mutating verbs with webhook logs.

6) Validate recovery
- Memory stabilizes well below limits and latency normalizes for 30–60m.

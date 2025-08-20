# Namespace CPU Usage Above 90%

## Description
Namespace workloads are consuming >90% of requested or limited CPU.

## Detection
- Dashboard: CPU Utilisation (from requests/limits), CPU Usage
- Metrics:
  - `sum(node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate{namespace="<ns>"}) / sum(cluster:namespace:pod_cpu:active:kube_pod_container_resource_requests{namespace="<ns>",resource="cpu"})`
  - `... / sum(cluster:namespace:pod_cpu:active:kube_pod_container_resource_limits{namespace="<ns>",resource="cpu"})`

## Resolution (Read-only investigation)
1) Identify top CPU consumers
```promql
topk(10, sum by (pod, container) (rate(container_cpu_usage_seconds_total{namespace="<ns>"}[5m])))
```
2) Check throttling (if metric present)
```promql
sum by (pod, container) (rate(container_cpu_cfs_throttled_seconds_total{namespace="<ns>"}[5m]))
```
3) Hand-off
- Provide top pods/containers, utilization vs requests/limits, and throttling evidence.

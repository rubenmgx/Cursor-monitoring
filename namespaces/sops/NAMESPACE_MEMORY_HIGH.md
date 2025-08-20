# Namespace Memory Usage Above 90%

## Description
Namespace workloads are consuming >90% of requested or limited memory, risking OOM.

## Detection
- Dashboard: Memory Utilisation (from requests/limits), Memory Usage
- Metrics:
  - `sum(container_memory_working_set_bytes{namespace="<ns>",container!=""}) / sum(cluster:namespace:pod_memory:active:kube_pod_container_resource_requests{namespace="<ns>",resource="memory"})`
  - `... / sum(cluster:namespace:pod_memory:active:kube_pod_container_resource_limits{namespace="<ns>",resource="memory"})`

## Resolution (Read-only investigation)
1) Identify top memory consumers
```promql
topk(10, sum by (pod, container) (container_memory_working_set_bytes{namespace="<ns>",container!=""}))
```
2) Look for growth
```promql
avg_over_time(container_memory_working_set_bytes{namespace="<ns>",container!=""}[30m])
```
3) Hand-off
- Provide top pods/containers and growth patterns; note limits/requests vs usage.

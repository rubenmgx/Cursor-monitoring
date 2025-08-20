# Namespace Availability

## Description
The Konflux_up availability signal for the namespace dipped, indicating service unavailability.

## Detection
- Dashboard: Availability / Konflux_up Signal
- Metric: `count_over_time(konflux_up{namespace="<ns>"}==1[$__range]) / count_over_time(konflux_up{namespace="<ns>"}[$__range])`

## Resolution (Read-only investigation)
1) Confirm the time window and impacted namespaces
2) Check replica ratio
```promql
sum by (namespace) (kube_deployment_status_replicas_available)
/ sum by (namespace) (kube_deployment_spec_replicas)
```
3) Restarts and readiness
```bash
NS=<namespace>
kubectl -n $NS get deploy,rs,po -o wide
kubectl -n $NS get events --sort-by=.lastTimestamp | tail -n 50
```
4) Hand-off
- Provide timestamps of dips, replica shortfall, restart counts, and recent events.

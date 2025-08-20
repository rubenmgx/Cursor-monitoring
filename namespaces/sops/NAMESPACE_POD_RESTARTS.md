# Namespace Pod Restarts

## Description
Pods in the namespace have restarted within the incident window, indicating instability (crashes, OOM, image issues).

## Detection
- Dashboard: Pod Restarts / Namespace stat panel
- Metric: `increase(kube_pod_container_status_restarts_total[15m])`

## Resolution (Read-only investigation)
1) Identify top restarting pods/containers
```promql
sum by (namespace, pod, container) (increase(kube_pod_container_status_restarts_total[15m]))
```
2) Gather pod status and last events
```bash
NS=<namespace>
kubectl -n $NS get pods -o wide
kubectl -n $NS describe pod <pod>
```
3) Hand-off notes
- Provide pod names, containers, restart counts, and last events/log snippets to the owning team.

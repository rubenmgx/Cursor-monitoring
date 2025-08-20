# API Server Requests Not Being Processed

## Developer Communications

- [PROJECT on JIRA](Project URL)
- [#channel on Slack](Channel URL)

## Description

API Server appears to stop processing requests (request rate drops to near zero) or requests are stuck inflight for prolonged periods.

## Pre-requisites

- Access to Grafana dashboard `rugomez - API SERVER testing` (uid: `bep80qv02u6tce`)
- Access to Prometheus for ad-hoc queries

## Detection

### User feedback
- Timeouts and stalled kubectl/oc commands; controllers not making progress.

### Monitoring
- Dashboard panels to check:
  - API Server Request Rate by Instance; Requests being processed; API Server latency
- Key metrics/queries:
  - Total request rate:
    - `sum(rate(apiserver_request_total[5m]))`
  - Inflight requests:
    - `sum(apiserver_current_inflight_requests)`

## Resolution

1) Confirm the symptom
- In dashboard, verify request rate near zero across instances and/or rising inflight with flat RPS.
- PromQL quick checks:
```promql
sum(rate(apiserver_request_total[5m]))
sum(apiserver_current_inflight_requests)
```

2) Check apiserver and control plane health
```bash
NS=${APISERVER_NS:-openshift-kube-apiserver}
kubectl -n $NS get pods -l component=kube-apiserver -o wide
kubectl get --raw='/readyz?verbose' | head -100
```

3) Networking and LB
```bash
kubectl -n default get endpoints kubernetes -o wide
kubectl -n default describe svc kubernetes | sed -n '1,200p'
```

4) If inflight rising but RPS flat (stalls)
- Inspect admission webhooks and downstream dependencies.
```bash
kubectl get validatingwebhookconfigurations,mutatingwebhookconfigurations -A | cat
```

5) Mitigations
- Restart the most impacted apiserver pod as a stopgap.
- Reduce client concurrency/backoff.

6) Validate recovery
- RPS normalizes and inflight returns to baseline.

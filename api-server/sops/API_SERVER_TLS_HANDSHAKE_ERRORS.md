# API Server TLS Handshake Errors

## Developer Communications

- [PROJECT on JIRA](Project URL)
- [#channel on Slack](Channel URL)

## Description

Clients fail to complete TLS handshakes to the API Server, causing connection failures before requests are processed. Often related to certificate rotations, trust issues, or protocol mismatches.

## Pre-requisites

- Access to Grafana dashboard `rugomez - API SERVER testing` (uid: `bep80qv02u6tce`)
- Access to Prometheus for ad-hoc queries

## Detection

### User feedback
- Connection errors before HTTP status codes are returned; widespread client retries.

### Monitoring
- Dashboard panels to check:
  - Rate TLS handshake error; Request Rate by Status (secondary)
- Key metrics/queries:
  - `rate(apiserver_tls_handshake_errors_total[5m])`
  - Errors over time vs codes: `code:apiserver_request_total:rate5m`

## Resolution

1) Confirm spike and time window
- In dashboard: check "Rate TLS handshake error" against incident time.
- PromQL quick check:
```promql
rate(apiserver_tls_handshake_errors_total[5m])
```

2) Validate serving certs and trust
```bash
# Kubernetes service endpoints for API
kubectl -n default get endpoints kubernetes -o wide
# Test TLS directly from a node/pod (adjust host/IP accordingly)
openssl s_client -connect <api-host-or-ip>:6443 -servername <api-hostname> -showcerts </dev/null 2>/dev/null | head -n 50
```

3) Check recent rotations and cert SANs
- Verify that the presented certificate chain matches expected CA and SANs.

4) Client-side trust
- If specific clients fail, refresh their CA bundle or kubeconfig credentials.

5) Validate recovery
- Handshake error rate returns to zero and request success normalizes.

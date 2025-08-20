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

1. If handshake errors spike:
   - Verify recent certificate rotations (serving certs) and CA trust distribution to clients.
   - Check TLS policy/protocol settings on ingress/LB paths.
2. Validate server certificate chain and SANs match advertised endpoints.
3. If specific client group failing, refresh their CA trust or credentials.
4. Confirm decline in handshake errors and normalization of client success rates.

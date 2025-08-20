# API Server Monitoring

This repository contains Kubernetes API Server operational runbooks (SOPs) and Prometheus alert rules to detect and remediate common failure modes.

## Repository structure

- `docs/`
  - `API_SERVER_DASHBOARD_SOP.md`: How to use the API Server dashboard (panels, investigations)
- `sops/api-server/` (one SOP per topic)
  - `API_SERVER_CPU_HIGH.md`
  - `API_SERVER_MEMORY_HIGH.md`
  - `API_SERVER_HIGH_LATENCY.md`
  - `API_SERVER_REQUESTS_NOT_PROCESSED.md`
  - `API_SERVER_TLS_HANDSHAKE_ERRORS.md`
  - `API_SERVER_5XX_ERRORS.md`
  - `API_SERVER_429_THROTTLING.md`
- `alerts/api-server/` (Prometheus alert rules)
  - `api_server_cpu_high.yaml`
  - `api_server_memory_high.yaml`
  - `api_server_latency_high.yaml`
  - `api_server_requests_not_processed.yaml`
  - `api_server_tls_handshake_errors.yaml`
  - `api_server_errors_5xx.yaml`
  - `api_server_429_throttling.yaml`

## Dashboards

- API Server dashboard uid: `bep80qv02u6tce`
  - The SOPs reference panels on this dashboard for triage and validation.

## Deploying alert rules

Options depend on your stack:

- Prometheus (bare): copy files from `alerts/api-server/` into your Prometheus `rule_files` path and reload Prometheus.
- Prometheus Operator:
  - Wrap each rules file into a `PrometheusRule` or combine as needed.
  - Example:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: api-server-alerts
  namespace: monitoring
spec:
  groups: [] # paste the groups: [...] from one of the YAMLs here
```

Apply:
```bash
kubectl apply -f api-server-alerts.yaml
```

## Using the SOPs

- Each SOP describes detection, dashboard panels to inspect, PromQL pivots, and actionable steps.
- Navigate to `sops/api-server/` and follow the steps for the alert that fired.

## Contributing

- Follow the SOP template guidelines:
  - Ensure the SOP has an associated alert in `alerts/api-server/`
  - Provide direct links to dashboards/logs where possible
  - Assume minimal prior knowledge; make steps explicit and actionable
- Add new SOPs under `sops/api-server/` and corresponding rules under `alerts/api-server/`.

## License

TBD (add a license if required).

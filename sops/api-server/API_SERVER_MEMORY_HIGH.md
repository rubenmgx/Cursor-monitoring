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

1. Identify instances with steady growth or >90% of limit.
2. If limits too low, raise memory limits; otherwise restart the most impacted apiserver to relieve pressure (temporary).
3. Investigate causes:
   - Large LIST responses, watch fan-out, or cache growth.
   - Admission webhooks buffering/serialization overhead.
4. Reduce memory footprint or increase limits; monitor for stability over 30–60m.

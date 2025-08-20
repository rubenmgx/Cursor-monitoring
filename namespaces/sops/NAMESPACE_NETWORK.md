# Namespace Network Anomalies

## Description
Elevated network throughput or packet rates for a namespace indicating surges or anomalies.

## Detection
- Dashboard: Network Transmit/Receive Bytes, Inbound/Outbound Packets
- Metrics:
  - `sum(rate(container_network_transmit_bytes_total{namespace="<ns>",container!=""}[5m]))`
  - `sum(rate(container_network_receive_bytes_total{namespace="<ns>",container!=""}[5m]))`
  - Packet rates with `..._packets_total`

## Resolution (Read-only investigation)
1) Identify top talkers
```promql
topk(10, sum by (pod, container) (rate(container_network_transmit_bytes_total{namespace="<ns>",container!=""}[5m])))
```
2) Packet vs bytes asymmetry
- High packets with low bytes → many small requests/retries
- High bytes with low packets → large payloads

3) Hand-off
- Provide top pods/containers and the time window; correlate with service endpoints.

# Monitoring & Security Auditing

This section covers the tools used in the ECN2026 infrastructure for system monitoring, metrics collection, and security auditing.

---

## Overview

| Tool | Purpose |
|---|---|
| [Prometheus](prometheus.md) | Collects and stores system metrics as time-series data |
| [Grafana](grafana.md) | Visualizes metrics from Prometheus through dashboards |
| [Lynis](lynis.md) | Performs security audits and hardening checks on the VPS |
| [OpenVAS](openvas.md) | Scans the infrastructure for known vulnerabilities |

---

## How It Fits Together

Prometheus and Grafana form the monitoring stack. Node Exporter collects system metrics on the VPS, Prometheus scrapes and stores them, and Grafana provides a visual dashboard accessible through the WireGuard VPN.

Lynis and OpenVAS serve as security auditing tools. Lynis runs directly on the VPS to assess system configuration and hardening. OpenVAS performs network-based vulnerability scans against the infrastructure.

```
Monitoring:    Node Exporter → Prometheus → Grafana

Security:      Lynis        → local audit on VPS
               OpenVAS      → network vulnerability scan
```

# Monitoring Stack (Prometheus + Grafana)

As part of the **ECN2026 infrastructure**, a lightweight monitoring stack was deployed on the VPS to observe system health and performance metrics.

The setup uses a common open-source stack:

Node Exporter → Prometheus → Grafana

---

# Architecture

**Node Exporter** collects system metrics directly from the host:

- CPU usage
- memory usage
- disk usage
- network traffic
- filesystem metrics
- system load

**Prometheus** periodically scrapes these metrics and stores them as time-series data.

**Grafana** provides visualization through dashboards and graphs.

---

# Deployment

The stack is deployed using **Docker Compose**.

### Services:

- **Prometheus** – metrics collection and time-series database
- **Node Exporter** – host metrics exporter
- **Grafana** – monitoring dashboards

Prometheus scrapes Node Exporter locally:

targets:
```yaml
"127.0.0.1:9100"
```
---

# Security Design

Monitoring services are not publicly exposed.

Access is restricted to the **WireGuard VPN network**.

Internet
   │
   └── blocked
        │
        ▼
WireGuard VPN
        │
        ├── Prometheus
        └── Grafana

This prevents bots or external scanners from accessing internal monitoring data.

---

# Dashboards

Grafana dashboards were imported from the community repository.

Example dashboard:
```
Node Exporter Full
Grafana Dashboard ID: 1860
```

This provides a complete overview of:

- CPU utilization

- memory usage

- disk I/O

- network traffic

- system load

---

# Purpose within ECN2026

The monitoring stack helps to:

- observe server performance

- detect resource bottlenecks

- verify system stability

- build practical monitoring experience with Prometheus and Grafana

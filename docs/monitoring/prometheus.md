# Prometheus Deployment Documentation

**Project:** ECN2026 – Monitoring Stack
**Component:** Prometheus (Metrics Collection & Monitoring)

---

## 1. Objective

The goal of this deployment was to introduce infrastructure-level monitoring into the ECN2026 lab environment. Prometheus was implemented to collect, store, and query time-series metrics from Linux servers and selected services.

The monitoring stack is designed to support:

- Infrastructure observability
- Service health validation
- Baseline performance analysis
- Future alerting integration
- Security-relevant anomaly detection support

Prometheus is part of the monitoring and security layer alongside OpenVAS (vulnerability scanning) and Wazuh (SIEM).

---

## 2. Environment

| Parameter         | Value                          |
| ----------------- | ------------------------------ |
| Server Role       | Monitoring Node                |
| OS                | Ubuntu 24.04 LTS              |
| Network           | IPv4 only                      |
| Deployment Method | Docker (containerized deployment) |

Prometheus runs in a dedicated Docker container to ensure:

- Isolation from host system
- Reproducibility
- Simplified updates
- Clean configuration management

---

## 3. Installation Method

Prometheus was deployed using Docker Compose.

### 3.1 Directory Structure

```
/opt/prometheus/
 ├── docker-compose.yml
 └── prometheus.yml
```

### 3.2 Docker Compose Configuration

Prometheus container configuration:

- Official Prometheus image
- Persistent volume for time-series data
- Config file mounted as read-only
- Port 9090 exposed internally

Key characteristics:

- Restart policy enabled
- Data stored outside container
- Clean separation between config and runtime data

### 3.3 Prometheus Configuration (prometheus.yml)

The configuration defines:

- Global scrape interval
- Target endpoints
- Job definitions

Example logical structure:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets: ["<server-ip>:9100"]
```

---

## 4. Exporters Deployed

To collect system metrics, Node Exporter was installed on monitored hosts.

### Node Exporter

**Purpose:**

- CPU metrics
- Memory usage
- Disk I/O
- Filesystem utilization
- Network statistics

**Port:** `9100/TCP`

Node Exporter runs as a service on monitored Linux machines and exposes metrics in Prometheus format.

---

## 5. Verification Steps

After deployment:

1. Container status verified via:

   ```bash
   docker ps
   ```

2. Prometheus UI accessed via:

   ```
   http://<monitoring-server-ip>:9090
   ```

3. Targets page checked:

   ```
   Status → Targets
   ```

4. Verified:
   - Targets in "UP" state
   - Scrape intervals working
   - Metrics available in expression browser

---

## 6. Metrics Collected

Examples of validated metrics:

| Metric                              | Description                |
| ----------------------------------- | -------------------------- |
| `node_cpu_seconds_total`            | CPU usage                  |
| `node_memory_MemAvailable_bytes`    | Available memory           |
| `node_filesystem_avail_bytes`       | Filesystem availability    |
| `node_network_receive_bytes_total`  | Network receive traffic    |
| `up`                                | Target availability status |

These metrics allow:

- Capacity planning
- Performance baselining
- Infrastructure health tracking
- Detection of abnormal resource consumption

---

## 7. Security Considerations

- Prometheus UI not exposed publicly
- Access limited via firewall rules
- No anonymous internet exposure
- Monitoring traffic restricted to internal VPN network (WireGuard)

**Future hardening:**

- Reverse proxy with authentication
- TLS termination
- Role-based access
- Alertmanager integration

---

## 8. Architectural Role in ECN2026

Prometheus serves as the **Infrastructure Metrics Engine**:

- Feeds visualization (Grafana – optional)
- Supports security monitoring (Wazuh correlation)
- Complements OpenVAS scanning results

It provides quantitative observability, while:

- **OpenVAS** provides vulnerability visibility
- **Wazuh** provides security event detection

---

## 9. Lessons Learned

- Containerized deployment simplifies reproducibility
- Exporter configuration must match firewall policies
- Scrape intervals directly affect storage growth
- Metric naming consistency is critical for queries
- Monitoring must remain internal to reduce attack surface

---

## 10. Future Enhancements

- Alertmanager integration
- Long-term storage tuning
- Multi-node federation
- Dashboard integration (Grafana)
- Integration with incident response workflow

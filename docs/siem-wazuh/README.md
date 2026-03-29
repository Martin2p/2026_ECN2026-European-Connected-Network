# SIEM - Wazuh

## Overview

This project implements Wazuh as a SIEM (Security Information and Event Management) solution
to monitor and protect a web server. All central Wazuh components run as
Docker containers on the same host, while the Wazuh Agent is installed natively to ensure
full visibility into the host system.

## Architecture

| Component | Deployment | Purpose |
|---|---|---|
| Wazuh Server | Docker Container | Analyzes agent data, processes rules & decoders, detects threats |
| Wazuh Indexer | Docker Container | Indexes and stores alerts, enables real-time search & analytics |
| Wazuh Dashboard | Docker Container | Web UI for visualization, alert management, and compliance |
| Wazuh Agent | Native on Host | Collects logs, monitors file integrity, reports to Wazuh Server |

### Architecture Decision: Single-Host Docker Deployment

**Context:** The project runs on a single VPS (8 GB RAM, 3 CPU Cores) that also
hosts the web server. A dedicated SIEM server is not available.

**Decision:** Deploy Wazuh Server, Indexer, and Dashboard as Docker containers on the same
host. Install the Wazuh Agent natively.

**Reasoning:**
- Docker provides process isolation between the SIEM stack and the web server
- The Wazuh Agent must run natively because a containerized agent cannot directly
  access or monitor the host system (logs, file integrity, processes)
- A single-host deployment is sufficient for monitoring one endpoint in a learning
  and portfolio context

**Trade-offs:**
- If an attacker gains root access on the host, Docker isolation will not protect the
  SIEM data — this is an accepted risk for this project scope
- Resource constraints require careful memory limits on all containers

## Resource Allocation

Total available: **8 GB RAM / 3 CPU Cores**

| Component | RAM Limit | Notes |
|---|---|---|
| Wazuh Indexer | 2 GB | Largest consumer (OpenSearch), heap limited via `OPENSEARCH_JAVA_OPTS` |
| Wazuh Server | 1 GB | Handles analysis engine, API, agent connection |
| Wazuh Dashboard | 1 GB | Web UI, queries Indexer API |
| Wazuh Agent | ~50 MB | Lightweight, runs natively |
| Host OS + Web Server + Docker | ~4 GB | Remaining resources for all other services |

## Network Access

All management interfaces are accessible exclusively through the WireGuard VPN tunnel.
No management ports are exposed to the public internet. This follows the same access
policy applied to Grafana and OpenVAS on this server.

| Service | Port | Access |
|---|---|---|
| Wazuh Agent Connection | 1514/TCP | localhost only (Agent runs on same host) |
| Wazuh Server API | 55000/TCP | VPN only (WireGuard) |
| Wazuh Indexer | 9200/TCP | localhost only (internal communication) |
| Wazuh Dashboard | 443/TCP | **VPN only** (WireGuard) |

### Access Control — Defense in Depth

Access restriction is enforced at two levels:

**1. Docker Port Binding**
Dashboard and API bind exclusively to the WireGuard interface IP:
```yaml
services:
  wazuh.dashboard:
    ports:
      - "10.x.x.1:443:5601"    # VPN interface only
  wazuh.manager:
    ports:
      - "10.x.x.1:55000:55000" # VPN interface only
      - "127.0.0.1:1514:1514"  # localhost only (agent connection)
```

**2. nftables Firewall Rules**
Management ports are explicitly allowed only on the WireGuard interface (`wg0`):
```
iifname "wg0" tcp dport 443 accept
iifname "wg0" tcp dport 55000 accept
```

For the complete firewall ruleset, see [`docs/firewall-nftables/`](../firewall-nftables/).

## Documentation

| Document | Description |
|---|---|
| [Installation](installation.md) | Step-by-step setup of Docker stack and native Agent |
| [Agent Configuration](agent-config.md) | ossec.conf, monitored log sources, enabled modules |
| [Rules & Decoders](rules-decoders.md) | Custom rules and decoder configurations |
| [Alerting](alerting.md) | Alert thresholds, notification setup, dashboard configuration |

## Related

- Firewall configuration: [`docs/firewall-nftables/`](../firewall-nftables/)
- Server nftables config: [`Projekt/configs/firewall/`](../../Projekt/configs/firewall/)

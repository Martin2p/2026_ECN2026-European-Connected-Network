# Resilience & Service Recovery

## Purpose within ECN2026

This document covers observed restart and recovery behaviour across the ECN2026 stack following unplanned reboots, manual container restarts, and service-level failures. The goal is to understand which components recover autonomously and where manual intervention is required — a realistic operational baseline for a single-node lab environment.

---

## Scope

| Test type | Description |
|---|---|
| Server reboot | Full VPS reboot via Contabo control panel or `sudo reboot` |
| Container restart | Individual `docker compose restart <service>` |
| Compose stack restart | `docker compose down && docker compose up -d` for a full stack |

All tests performed on `vmi2680499` (Ubuntu 24.04, Docker with `"iptables": false`, nftables managed manually).

---

## Results

| Component | Recovery | Automatic | Notes |
|---|---|---|---|
| **WireGuard** | Full | ✅ Yes | Tunnel re-establishes on boot via `systemd`; peer reconnects within seconds |
| **nftables** | Full | ✅ Yes | Rules persist across reboots via `systemctl enable nftables`; `inet meinefirewall` table reloads cleanly |
| **Nginx** | Full | ✅ Yes | All vhosts (`.com`, `.de`, `.se`) recover without intervention |
| **Grafana** | Full | ✅ Yes | Dashboard available on WireGuard interface (`10.100.0.1`) after stack restart |
| **Prometheus** | Full | ✅ Yes | Scrape targets resume automatically |
| **NetBox** | Full | ✅ Yes | All containers (`netbox`, `postgres`, `redis`) come up in correct order via `depends_on` |
| **Wazuh** | Partial | ⚠️ No | See finding below |
| **OpenVAS/Greenbone** | N/A | ➖ Deactivated | See finding below |

---

## Finding: Wazuh Startup Failure After Reboot

### Symptom

After a server reboot or full Wazuh stack restart, one or more Wazuh components fail to reach a healthy state. The dashboard and/or manager remain unavailable until manually restarted via console.

### Root Cause

The Wazuh stack consists of three tightly coupled services: `wazuh.indexer` (OpenSearch), `wazuh.manager`, and `wazuh.dashboard`. OpenSearch requires significant initialisation time before it can accept connections. If the manager or dashboard starts before the indexer is fully ready, healthchecks fail and the dependent containers exit or stall.

This is a known timing issue inherent to single-node Wazuh deployments in Docker — `depends_on` with `condition: service_healthy` helps but does not fully eliminate the race condition, as OpenSearch initialisation duration varies under load.

### Workaround (Current)

Manual restart of the affected service(s) via console after the indexer has fully initialised:

```bash
docker compose -f /path/to/wazuh/docker-compose.yml restart wazuh.manager
# or, if the dashboard is also affected:
docker compose -f /path/to/wazuh/docker-compose.yml restart wazuh.dashboard
```

### Status

**Open — deferred.** Accepted as a known limitation of the current single-node architecture. A startup delay or retry wrapper could mitigate this, but is not prioritised at this stage.

### Architectural Note

This behaviour reinforces the case for separating the SIEM layer onto a dedicated node in a future multi-node ECN2026 topology. On a standalone host, Wazuh would not compete for memory during initialisation and would be less susceptible to timing-related failures.

---

## Finding: OpenVAS/Greenbone Deactivated — Resource Constraints

### Symptom

Running OpenVAS/Greenbone alongside the existing stack (Wazuh, NetBox, Prometheus, Grafana) pushed the VPS to its resource limits. Feed synchronisation and scan processes caused sustained high CPU and memory usage, degrading the stability of other services.

### Decision

OpenVAS Docker volumes were removed and the service was deactivated. Approximately 27 GB of disk space was reclaimed.

### Status

**Permanently deactivated on this node.** Accepted as an architectural trade-off.

### Architectural Note

A vulnerability scanner and its targets should not run on the same host — scans against localhost produce misleading results and do not reflect realistic network attack surfaces. OpenVAS is better suited to a dedicated node that scans other ECN2026 components externally. This is noted as a future consideration for the multi-node topology.

---

## Lessons Learned

- **Resource-heavy services require dedicated nodes.** OpenVAS/Greenbone was deactivated after sustained resource pressure on the single VPS. Vulnerability scanners belong on isolated nodes — both for realistic scan results and to avoid impacting production-equivalent services. Services with long initialisation times (OpenSearch, PostgreSQL) require careful `depends_on` / healthcheck tuning that goes beyond default Compose behaviour.
- **nftables persistence is reliable.** With `"iptables": false` in Docker daemon config and rules managed via `/etc/nftables.conf`, the firewall state survives reboots cleanly — no rules are lost or duplicated.
- **WireGuard-first access model holds under failure.** After every reboot tested, the WireGuard interface came up before any service became reachable externally, meaning the security perimeter was never bypassed during recovery.

# SIEM - Wazuh

## Overview

This document describes the deployment of Wazuh as a SIEM (Security Information and Event
Management) solution within the ECN2026 project. All central Wazuh components run as Docker
containers on the Contabo VPS, while the Wazuh Agent is installed natively on the host to
ensure full visibility into the host system.

The Wazuh Dashboard is accessible exclusively via WireGuard VPN at `https://10.100.0.1:8443`,
consistent with the security model applied to Grafana and OpenVAS within ECN2026.

---

## Purpose within ECN2026

Wazuh serves as the SIEM layer of ECN2026, complementing the existing monitoring stack
(Prometheus, Grafana) with security-focused capabilities: log analysis, file integrity
monitoring, threat detection, and compliance reporting. It provides visibility into security
events across the simulated European network infrastructure.

---

## Architecture

```
+---------------------------------------------------+
|              Contabo VPS (Ubuntu 24.04)           |
|                                                   |
|  +---------------+   +------------------------+  |
|  | Wazuh Manager |   |     Wazuh Indexer      |  |
|  | (Docker)      |<->| (Docker / OpenSearch)  |  |
|  +---------------+   +------------------------+  |
|          ^                      |                 |
|          |            +---------+--------+        |
|          |            | Wazuh Dashboard  |        |
|          |            | (Docker)         |        |
|          |            | 10.100.0.1:8443  |        |
|          |            +------------------+        |
|          |                                        |
|  +-------+-------+                               |
|  |  Wazuh Agent  |  (native on host)             |
|  +---------------+                               |
+---------------------------------------------------+
         ^
         | WireGuard VPN only
         |
   [ Admin Client ]
```

| Component       | Deployment       | Purpose                                                        |
|----------------|------------------|----------------------------------------------------------------|
| Wazuh Manager  | Docker Container | Analyzes agent data, processes rules & decoders, detects threats |
| Wazuh Indexer  | Docker Container | Indexes and stores alerts, enables real-time search & analytics |
| Wazuh Dashboard| Docker Container | Web UI for visualization, alert management, and compliance      |
| Wazuh Agent    | Native on Host   | Collects logs, monitors file integrity, reports to Wazuh Manager |

---

## Architecture Decisions

### Single-Host Docker Deployment

**Context:** The ECN2026 project runs on a single Contabo VPS (8 GB RAM, 3 CPU Cores) that
also hosts the web server, Grafana, Prometheus, and OpenVAS. A dedicated SIEM server is not
available.

**Decision:** Deploy Wazuh Manager, Indexer, and Dashboard as Docker containers using the
official `wazuh-docker` single-node setup. Install the Wazuh Agent natively on the host.

**Reasoning:** The single-node Docker deployment is the most resource-efficient option for a
single-server environment. Native agent installation (rather than containerized) provides
complete host visibility including access to all system logs, file system events, and kernel
activity — which a containerized agent cannot fully achieve.

**Trade-offs:** All components share the same host resources. In a production environment,
Manager, Indexer, and Dashboard would run on dedicated infrastructure.

---

### VPN-Only Access for All Wazuh Services

**Context:** The Wazuh Dashboard exposes sensitive security data. Manager ports (1514, 1515,
55000) must not be reachable from the public internet. The default `wazuh-docker` single-node
setup binds all ports to `0.0.0.0`.

**Decision:** Bind all externally exposed ports explicitly to the WireGuard interface IP
(`10.100.0.1`). Disable the external port binding for the Indexer (port 9200) entirely, as
Manager and Dashboard communicate with it via the internal Docker network.

**Reasoning:** Consistent with the existing security model for Grafana and OpenVAS. Two
independent enforcement layers: Docker port binding and nftables. Even if nftables has a
misconfiguration, the Docker binding prevents exposure.

**Port binding after hardening (`docker-compose.yml`):**
```yaml
wazuh.dashboard:
  ports:
    - "10.100.0.1:8443:5601"

wazuh.manager:
  ports:
    - "10.100.0.1:1514:1514"
    - "10.100.0.1:1515:1515"
    - "10.100.0.1:514:514/udp"
    - "10.100.0.1:55000:55000"

wazuh.indexer:
  # port 9200 not exposed externally — internal Docker network only
```

**Trade-offs:**  The Wazuh Agent must connect to `10.100.0.1:1514` instead of `127.0.0.1:1514`,
requiring an explicit configuration change in `ossec.conf` (see Lessons Learned #3).

---

## Deployment

### Prerequisites

- Docker and Docker Compose installed
- WireGuard VPN operational (`wg0` interface at `10.100.0.1`)
- nftables configured with forward chain for Docker subnets
- `/etc/resolv.conf` symlinked to `/run/systemd/resolve/resolv.conf` (not the stub resolver)

### Repository

```bash
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker
git checkout v4.14.0
cd single-node/
```

### Certificate Generation

```bash
sudo docker compose -f generate-indexer-certs.yml run --rm generator
```

Certificates are written to `config/wazuh_indexer_ssl_certs/`.

### Stack Start

```bash
sudo docker compose up -d
```

Dashboard available at `https://10.100.0.1:8443` via WireGuard.

### Agent Installation

```bash
curl -s https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.0-1_amd64.deb \
  -o /tmp/wazuh-agent.deb
sudo WAZUH_MANAGER='127.0.0.1' dpkg -i /tmp/wazuh-agent.deb
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

---

## Operations

### Alert Triage — FIM Events

Wazuh File Integrity Monitoring (FIM) generates Level 7 alerts (rule 550) whenever a
monitored file is modified. Not every FIM alert indicates a threat — the standard triage
workflow is:

1. Identify the modified file from `syscheck.path` in the alert details
2. Check the modification timestamp (`syscheck.mtime_after`)
3. Cross-reference with package manager logs to confirm whether a system update caused
   the change

**Example:** Alert for `/usr/bin/rotatelogs` (part of `apache2-utils`):
```bash
grep "apache2" /var/log/apt/history.log
# Output confirmed apache2 upgraded on the same date → alert classified as legitimate
```

If the modification timestamp does not match any known update or deployment, the alert
should be treated as a potential incident and investigated further.

---

### SSH Brute-Force Monitoring

The server receives a continuous stream of SSH brute-force attempts from automated scanners
(observed: ~1,700 attempts per day). Wazuh detects these via rule group
`authentication_failed` and rule IDs 5710/5711/5716.

Fail2Ban runs alongside Wazuh and automatically bans IPs that exceed the failure threshold.
Both tools must read from the same log source to be effective — see Lessons Learned #4.

To view active bans:
```bash
sudo fail2ban-client status sshd
```

To search SSH brute-force alerts in the Wazuh Dashboard:
```
Threat Intelligence → Events → filter: rule.groups: authentication_failed
```

---

## Lessons Learned

### 1. Docker containers had no internet access

**Problem:** The certificate generator container failed with `The tool to create the
certificates does not exist in any bucket`. The container script downloads `wazuh-certs-tool.sh`
from `packages.wazuh.com` at runtime, but DNS resolution failed inside the container.

**Root cause:** Two separate issues combined:

- `/etc/resolv.conf` pointed to `127.0.0.53` (systemd-resolved stub), which is unreachable
  from inside Docker containers.
- The nftables `forward` chain had `policy drop` and only allowed outbound traffic for
  specific Docker subnets (`172.19.0.0/16`). The Docker bridge network (`172.17.0.0/16`)
  and Docker Compose networks (`172.16.0.0/12`) were missing.

**Fix 1 — resolv.conf:**
```bash
sudo unlink /etc/resolv.conf
sudo ln -s /run/systemd/resolve/resolv.conf /etc/resolv.conf
```
This replaces the stub resolver with the actual upstream DNS servers from the provider.

**Fix 2 — nftables forward chain (added to `/etc/nftables.conf`):**
```
# Docker bridge DNS
ip saddr 172.17.0.0/16 udp dport 53 accept comment "docker bridge DNS"
ip saddr 172.16.0.0/12 udp dport 53 accept comment "docker compose DNS"

# Docker bridge outbound
ip saddr 172.17.0.0/16 tcp dport { 80, 443 } accept comment "docker bridge outbound"
ip saddr 172.16.0.0/12 tcp dport { 80, 443 } accept comment "docker compose outbound"
```

**Fix 3 — nftables NAT/masquerading (added to `table ip nat POSTROUTING`):**
```
ip saddr 172.17.0.0/16 oifname "eth0" masquerade comment "docker bridge outbound"
ip saddr 172.16.0.0/12 oifname "eth0" masquerade comment "docker compose outbound"
```

**Note:** `"iptables": false` is set in `/etc/docker/daemon.json` because the host uses
nftables instead of iptables. This means Docker does not set up its own NAT or FORWARD rules
— these must be maintained manually in nftables.

---

### 2. Dashboard port conflict with Nginx

**Problem:** `docker compose up` failed because port 443 was already allocated by the host
Nginx web server.

**Fix:** Changed the Dashboard port binding in `docker-compose.yml`:
```yaml
# Before:
ports:
  - "443:5601"

# After:
ports:
  - "10.100.0.1:8443:5601"
```

---

### 3. Wazuh Agent disconnected after port binding hardening

**Problem:** After binding all Manager ports to `10.100.0.1` (WireGuard interface), the
natively installed Wazuh Agent could no longer connect. The agent was configured to reach
the Manager at `127.0.0.1:1514`, which was no longer bound.

**Error in `/var/ossec/logs/ossec.log`:**
```
wazuh-agentd: ERROR: (1216): Unable to connect to '[127.0.0.1]:1514/tcp':
'Transport endpoint is not connected'.
```

**Fix:** Updated the Manager address in `/var/ossec/etc/ossec.conf`:
```xml
<server>
  <address>10.100.0.1</address>
  <port>1514</port>
  <protocol>tcp</protocol>
</server>
```

```bash
sudo systemctl restart wazuh-agent
```

**Reasoning:** The agent communicates with the Manager via the WireGuard interface, consistent
with all other administrative access in ECN2026. This also means the agent connection is
subject to the same nftables rules as other WireGuard traffic.

---

### 4. Fail2Ban not reading SSH logs on Ubuntu 24.04

**Problem:** Fail2Ban reported `Currently failed: 0` despite thousands of SSH brute-force
attempts visible in Wazuh. Fail2Ban was configured to read `/var/log/auth.log`, but Ubuntu
24.04 uses `systemd-journald` by default — making `auth.log` nearly empty.

**Diagnosis:**
```bash
sudo fail2ban-client get sshd logpath
# Output: /var/log/auth.log  ← wrong source
```

**Fix:** Changed the backend to `systemd` in `/etc/fail2ban/jail.local`:
```ini
[sshd]
enabled = true
port = ssh
filter = sshd
backend = systemd
```

Removed the `logpath` line entirely — not needed with `backend = systemd`.

**Verification:**
```bash
sudo fail2ban-client status sshd
# Output: Journal matches: _SYSTEMD_UNIT=sshd.service + _COMM=sshd
```

Fail2Ban now reads from the same source as Wazuh, ensuring consistent detection and
blocking across both tools.

---

## Version

| Component     | Version |
|--------------|---------|
| Wazuh Stack  | 4.14.0  |
| Wazuh Agent  | 4.14.0  |
| Deployment   | single-node Docker |

---

*Part of ECN2026 — European Connected Network | tastlernetworks.com*

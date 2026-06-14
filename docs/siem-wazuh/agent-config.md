# Wazuh Agent Configuration

## Purpose within ECN2026

The Wazuh Agent runs natively on the Contabo VPS host and acts as the primary sensor
for the SIEM layer. It collects security-relevant data from the host system and forwards
it to the Wazuh Manager running inside Docker. The agent must run natively — a
containerized agent cannot directly access host logs, processes, or the filesystem.

## Agent Communication

| Parameter | Value |
|---|---|
| Manager address | `10.100.0.1` (WireGuard interface) |
| Port | `1514/tcp` |
| Encryption | AES |
| Reconnect interval | 60 seconds |

The agent communicates exclusively via the WireGuard interface, consistent with the
VPN-only access model applied to all administrative services in ECN2026.

## Active Modules

### Rootcheck

Scans for rootkits, trojans, suspicious processes, open ports, and network interfaces.
Runs every 12 hours.

Docker overlay directories are excluded to avoid false positives:

```xml
<ignore>/var/lib/containerd</ignore>
<ignore>/var/lib/docker/overlay2</ignore>
```

### Syscollector (System Inventory)

Collects a full inventory of the host every hour and on startup:

| Data collected |
|---|
| Hardware information |
| OS details |
| Installed packages |
| Open network ports |
| Running processes |
| Local users and groups |
| Active services |

This data is visible in the Wazuh Dashboard under **Inventory** and provides a baseline
for detecting unauthorized changes to the system state.

### Security Configuration Assessment (SCA)

Evaluates the host configuration against CIS Benchmark policies every 12 hours and on
startup. Results are visible in the Wazuh Dashboard under **Configuration Assessment**
and show which hardening checks pass or fail.

### File Integrity Monitoring (FIM / Syscheck)

Monitors critical directories for unauthorized file changes. Default scan interval is
12 hours; the web root uses real-time monitoring.

| Path | Mode | Notes |
|---|---|---|
| `/etc` | scheduled | System configuration files |
| `/usr/bin`, `/usr/sbin` | scheduled | System binaries |
| `/bin`, `/sbin`, `/boot` | scheduled | Core binaries and bootloader |
| `/var/www/html` | **realtime** | Web root — immediate detection of defacement or unauthorized file changes |

The `/var/www/html` entry was added explicitly with `realtime="yes"` and
`report_changes="yes"` to detect web defacement or unauthorized file drops as they
happen, rather than waiting for the next scheduled scan:

```xml
<directories realtime="yes" report_changes="yes">/var/www/html</directories>
```

**Security rationale:** The web root is publicly accessible and the highest-risk
directory on this server. Real-time FIM ensures any unauthorized modification triggers
an immediate alert in Wazuh rather than going undetected for up to 12 hours.

## Log Sources

### systemd Journal

```xml
<localfile>
  <log_format>journald</log_format>
  <location>journald</location>
</localfile>
```

Captures all systemd-managed services including SSH, Fail2Ban, nftables, Docker, and
the Wazuh Agent itself.

### Nginx

```xml
<localfile>
  <log_format>apache</log_format>
  <location>/var/log/nginx/access.log</location>
</localfile>

<localfile>
  <log_format>apache</log_format>
  <location>/var/log/nginx/error.log</location>
</localfile>
```

Wazuh uses the `apache` log format parser for Nginx, as both use the Combined Log Format.
Access and error logs are monitored separately.

### Package Management

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/dpkg.log</location>
</localfile>
```

Tracks all package installations, upgrades, and removals. Useful for correlating system
changes with FIM alerts.

### Linux Audit Framework

```xml
<localfile>
  <log_format>audit</log_format>
  <location>/var/log/audit/audit.log</location>
</localfile>
```

### Active Response Log

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/ossec/logs/active-responses.log</location>
</localfile>
```

Records any automated responses triggered by Wazuh rules (e.g. IP blocks).

### Periodic Commands

| Command | Interval | Purpose |
|---|---|---|
| `df -P` | 6 minutes | Disk usage monitoring |
| `netstat -tulpn` (filtered) | 6 minutes | Snapshot of listening ports |
| `last -n 20` | 6 minutes | Recent login sessions |

These commands run on the agent and their output is analyzed by the Wazuh Manager for
anomalies such as unexpected new listening ports or unusual login activity.

## Inactive Modules

| Module | Status | Reason |
|---|---|---|
| `cis-cat` | disabled | Requires external CIS-CAT tool (not installed) |
| `osquery` | disabled | Requires osquery installation (not in scope) |

Apache2 log paths are present in the configuration as placeholders for potential future
use. Wazuh silently ignores log sources where the file does not exist.

## Configuration File

```
/var/ossec/etc/ossec.conf
```

After any change, restart the agent:

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

---

*Part of ECN2026 — European Connected Network | tastlernetworks.com*

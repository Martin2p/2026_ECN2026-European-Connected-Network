# Wazuh Alerting — ECN2026

## Overview

Wazuh generates alerts by correlating events collected from the agent against a ruleset.
Each alert carries a severity level (1–15), a rule ID, and — where applicable — a MITRE
ATT&CK technique mapping. This document covers the alert categories observed in ECN2026,
the primary dashboard used for triage, and the two concrete scenarios investigated during
Phase 3.

---

## Primary Alert View: Threat Hunting Dashboard

**Navigation:** `≡ Menu → Threat Intelligence → Threat Hunting`

The Threat Hunting dashboard is the main alert interface in Wazuh 4.14. It provides:

- Total alert count with severity breakdown
- Level 12+ alert counter (critical threshold)
- Authentication failure / success counters
- Top 10 Alert level evolution (time series)
- MITRE ATT&CK technique distribution
- Per-agent alert breakdown

The time range filter is available in the top-right corner (e.g. Last 24h / Last 90 days).
The `Events` tab within the same view provides a filterable raw event stream.

**Observed data — Last 90 days (as of 2026-06-14):**

| Metric                    | Value  |
|---------------------------|--------|
| Total alerts              | 38,224 |
| Level 12+ alerts          | 0      |
| Authentication failures   | 36,691 |
| Authentication successes  | 153    |

96% of all alert volume is SSH brute-force traffic from automated internet scanners.
A significant spike was observed around 2026-05-03, consistent with a scanner campaign
rather than a targeted attack (no corresponding Level 12+ alerts, no successful
authentication from unknown sources).

![Threat Hunting Dashboard — Last 90 days](/docs/images/threat-hunting-90days.png)

---

## MITRE ATT&CK Mapping

Wazuh automatically maps detected events to MITRE ATT&CK techniques. In ECN2026, the
dominant techniques over the 90-day observation period were:

| Technique               | Context                              |
|-------------------------|--------------------------------------|
| Password Guessing       | SSH brute-force attempts             |
| Brute Force             | SSH brute-force attempts             |
| SSH                     | SSH-based attack vector              |
| Valid Accounts          | Successful authentication events     |
| Sudo and Sudo Caching   | Privilege escalation monitoring      |

No lateral movement, persistence, or exfiltration techniques were detected.

---

## Alert Scenarios

### 1. SSH Brute-Force (Ongoing)

The server is continuously targeted by automated SSH scanners — a normal condition for
any internet-facing Linux host.

**Detection:**
- Rule group: `authentication_failed`
- Primary rule IDs: 5710, 5711, 5716
- Severity: Level 5–10 depending on frequency and pattern

**Wazuh + Fail2Ban integration:**
Fail2Ban runs alongside Wazuh and bans IPs that exceed the failure threshold. Both tools
must read from the same log source. On Ubuntu 24.04 with journald, `/var/log/auth.log`
is nearly empty — Fail2Ban must be configured to use the systemd journal backend:

```ini
# /etc/fail2ban/jail.local
[sshd]
backend = systemd
```

To check active bans:

```bash
sudo fail2ban-client status sshd
```

To filter SSH brute-force events in the dashboard:

```
Threat Hunting → Events → Add filter: rule.groups: authentication_failed
```

---

### 2. FIM Alert — `/usr/bin/rotatelogs`

**Rule:** 550 — Integrity checksum changed  
**Severity:** Level 7  
**Module:** File Integrity Monitoring (`Endpoint Security → File Integrity Monitoring`)

Wazuh FIM monitors critical system binaries and generates a Level 7 alert whenever a
checksum changes. Not every FIM alert indicates a compromise — system package updates
routinely modify monitored files.

**Triage workflow:**

1. Identify the modified file from `syscheck.path` in the alert detail
2. Note the modification timestamp (`syscheck.mtime_after`)
3. Cross-reference with package manager history:

```bash
grep "apache2" /var/log/apt/history.log
```

**Result:** The alert for `/usr/bin/rotatelogs` (part of `apache2-utils`) matched an
apache2 package upgrade on the same date → classified as **legitimate, no action required**.

If the modification timestamp does not correspond to any known update or deployment, the
alert should be treated as a potential incident and investigated further.

---

## Alert Severity Reference

Wazuh uses a 1–15 severity scale. Relevant thresholds for ECN2026:

| Level | Meaning                                 | Examples in ECN2026          |
|-------|-----------------------------------------|------------------------------|
| 3–4   | System / informational events           | Service starts, cron jobs    |
| 5–7   | Authentication events, FIM changes      | SSH attempts, rotatelogs FIM |
| 9–10  | Multiple authentication failures        | Brute-force patterns         |
| 12+   | Critical — requires immediate attention | Not observed in 90-day period|

The Threat Hunting dashboard highlights Level 12+ alerts separately as the primary
escalation indicator.

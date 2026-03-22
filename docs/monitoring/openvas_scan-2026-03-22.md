# OpenVAS Scan Report — March 22, 2026

## Scan Overview

| Property            | Value                                     |
|----------------------|-------------------------------------------|
| **Task**            | First intensive Scan of VPS Server        |
| **Target IP**       | 194.163.151.84 (tastlernetworks.se)       |
| **Scan Start**      | 2026-03-22, 10:22 AM (CET)               |
| **Scan End**        | 2026-03-22, 11:59 AM (CET)               |
| **Duration**        | approx. 1h 37min                          |
| **Scanner**         | OpenVAS / Greenbone                       |

## Result Summary

| Severity  | Count  |
|-----------|--------|
| Critical  | 0      |
| **High**  | **11** |
| **Medium**| **7**  |
| Low       | 2      |
| Log       | 0      |
| **Total** | **20** |

---

## Detailed Findings

### HIGH — phpinfo() Publicly Accessible Across Multiple Domains (CVSS 7.5)

**NVT:** EasyPHP Webserver <= 12.1 Multiple Vulnerabilities — Active Check  
**OID:** 1.3.6.1.4.1.25623.1.0.803189  
**QoD:** 99%

**Description:**  
The file `phpinfo.php` was publicly accessible via HTTPS, exposing detailed server information including PHP version, configuration paths, and PHP variables. OpenVAS classifies this as an EasyPHP vulnerability, though the actual finding is the publicly exposed `phpinfo()` output.

**Affected URLs (11× High):**

| #  | URL                                              |
|----|--------------------------------------------------|
| 1  | `https://www.tastlernetworks.de/phpinfo.php`     |
| 2  | `https://www.tastlernetworks.se/phpinfo.php`     |
| 3  | `https://vmi2680499.contaboserver.net/phpinfo.php`|
| 4  | `https://tastlernetwork.se/phpinfo.php`          |
| 5  | `https://tastlernetworks.ch/phpinfo.php`         |
| 6  | `https://tastlernetworks.com/phpinfo.php`        |
| 7  | `https://tastlernetworks.de/phpinfo.php`         |
| 8  | `https://tastlernetworks.se/phpinfo.php`         |
| 9  | `https://www.tastlernetwork.se/phpinfo.php`      |
| 10 | `https://www.tastlernetworks.ch/phpinfo.php`     |
| 11 | `https://www.tastlernetworks.com/phpinfo.php`    |

**Disclosed Information:**

- PHP version: 8.2.29
- Configuration path: `/usr/local/etc/php`
- PHP variables, loaded modules, environment variables

---

### MEDIUM — phpinfo() Output Reporting (CVSS 5.3)

**NVT:** phpinfo() Output Reporting (HTTP)  
**OID:** 1.3.6.1.4.1.25623.1.0.11229  
**QoD:** 80%

**Description:**  
Supplementary finding to the High-severity issue above. Confirms the public accessibility of `phpinfo()` output across 7 domains/subdomains. Enables attackers to retrieve usernames, IP addresses, web server version, operating system, and webroot directory.

**Referenced CVEs:** CVE-2008-0149, CVE-2023-49282, CVE-2023-49283, CVE-2024-10486

---

### LOW — ICMP Timestamp Reply (CVSS 2.1)

**NVT:** ICMP Timestamp Reply Information Disclosure  
**OID:** 1.3.6.1.4.1.25623.1.0.103190  
**QoD:** 80%

**Description:**  
The server responds to ICMP Timestamp requests (Type 13) with Timestamp Replies (Type 14). This could theoretically be used to exploit weak time-based random number generators in other services.

**Referenced CVE:** CVE-1999-0524

---

### LOW — TCP Timestamps Information Disclosure (CVSS 2.6)

**NVT:** TCP Timestamps Information Disclosure  
**OID:** 1.3.6.1.4.1.25623.1.0.80091  
**QoD:** 80%

**Description:**  
The server implements TCP timestamps as defined by RFC 1323/7323. As a side effect, the uptime of the remote host can be computed.

---

## Remediation Actions Taken

### ✅ Deleted phpinfo.php

- **Date:** 2026-03-22
- **Action:** Located and deleted `phpinfo.php` inside the `nginx-docker` container
- **Commands:**
  ```bash
  docker exec nginx-docker find / -name "phpinfo.php" 2>/dev/null
  docker exec nginx-docker rm /path/to/phpinfo.php
  ```
- **Status:** Completed — resolves all 11 High and 7 Medium findings
- **Verification:** `curl -I` no longer shows an `X-Powered-By` header

### ✅ Hidden Nginx Version Number

- **Date:** 2026-03-22
- **Action:** Added `server_tokens off;` to the `http` block in `nginx.conf`
- **Status:** Completed — HTTP response header now shows `Server: nginx` without version number
- **Verification:**
  ```
  curl -I https://tastlernetworks.com
  → Server: nginx  (no version number)
  ```

### ✅ Disabled TCP Timestamps

- **Date:** 2026-03-22
- **Action:** Added `net.ipv4.tcp_timestamps = 0` to `/etc/sysctl.conf` on the host
- **Commands:**
  ```bash
  echo "net.ipv4.tcp_timestamps = 0" | sudo tee -a /etc/sysctl.conf
  sudo sysctl -p
  ```
- **Status:** Completed

---

## Open Items

| Issue                        | Priority    | Status    |
|------------------------------|-------------|-----------|
| ICMP Timestamp Reply         | Low         | Open      |
| Set `expose_php = Off`       | Recommended | Open      |
| Verification rescan          | —           | Pending   |

---

## Next Steps

1. **Run a verification rescan** — Confirm all remediations via a new full scan
2. **Set `expose_php = Off`** — Configure in php.ini inside the container and mount via volume
3. **Block ICMP Timestamps** — Optionally block ICMP Type 13 via firewall (iptables/nftables)
4. **Establish regular scan schedule** — Set up monthly scan cycle

# OpenVAS Scan Comparison: Scan 1 vs. Scan 2

## Overview

| | Scan 1 | Scan 2 |
|---|---|---|
| **Date** | March 22, 2026, 10:22–11:59 CET | March 22, 2026, 15:21–16:59 CET |
| **Target** | 194.163.151.84 (tastlernetworks.se) | 194.163.151.84 (tastlernetwork.se) |
| **Scan Config** | Full and Fast | Full and Fast |
| **Results (before filter)** | 220 | 152 |
| **Critical** | 0 | 0 |
| **High** | 11 | 0 ✅ |
| **Medium** | 7 | 0 ✅ |
| **Low** | 2 | 1 |
| **Total (filtered)** | 20 | 1 |

---

## Findings Scan 1 (March 22, 2026, 10:22 CET)

### 🔴 High (CVSS: 7.5) — phpinfo.php publicly accessible

**NVT:** EasyPHP Webserver <= 12.1 Multiple Vulnerabilities - Active Check  
**Port:** 443/tcp  
**QoD:** 99%

**Affected URLs:**
- https://www.tastlernetworks.de/phpinfo.php
- https://www.tastlernetworks.se/phpinfo.php
- https://www.tastlernetworks.ch/phpinfo.php
- https://www.tastlernetworks.com/phpinfo.php
- https://www.tastlernetwork.se/phpinfo.php
- https://tastlernetworks.de/phpinfo.php
- https://tastlernetworks.se/phpinfo.php
- https://tastlernetworks.ch/phpinfo.php
- https://tastlernetworks.com/phpinfo.php
- https://tastlernetwork.se/phpinfo.php
- https://vmi2680499.contaboserver.net/phpinfo.php

**Issue:** The file `phpinfo.php` was publicly accessible across all domains, exposing detailed system information including PHP version 8.2.29, configuration paths and server variables.

**Risk:** Attackers could use this information to prepare targeted attacks including PHP code injection and authentication bypass.

**Action taken:** `phpinfo.php` removed from all domains.

---

### 🟠 Medium (CVSS: 5.3) — phpinfo() Output Reporting

**NVT:** phpinfo() Output Reporting (HTTP)  
**Port:** 443/tcp  
**QoD:** 80%  
**CVEs:** CVE-2008-0149, CVE-2023-49282, CVE-2023-49283, CVE-2024-10486

**Issue:** Supplementary finding to the phpinfo.php exposure — sensitive details such as the PHP process username, IP address, web server version and root directory were readable.

**Action taken:** Resolved by removing phpinfo.php.

---

### 🟡 Low (CVSS: 2.1) — ICMP Timestamp Reply

**NVT:** ICMP Timestamp Reply Information Disclosure  
**CVE:** CVE-1999-0524

**Issue:** The host responded to ICMP Timestamp requests (Type 13/14).  
**Risk:** Could theoretically be used to exploit weak time-based random number generators in other services.  
**Status:** Open / Accepted residual risk (Low severity)

---

### 🟡 Low (CVSS: 2.6) — TCP Timestamps

**NVT:** TCP Timestamps Information Disclosure  
**Port:** general/tcp

**Issue:** The host implements TCP Timestamps (RFC1323/RFC7323), allowing computation of system uptime.  
**Fix:** Add `net.ipv4.tcp_timestamps = 0` to `/etc/sysctl.conf`, then run `sysctl -p`  
**Status:** Open / Accepted residual risk (Low severity)

---

## Findings Scan 2 (March 22, 2026, 15:21 CET)

### 🟡 Low (CVSS: 2.1) — ICMP Timestamp Reply

**NVT:** ICMP Timestamp Reply Information Disclosure  
**CVE:** CVE-1999-0524  
**Status:** Remaining residual risk — no action required at this time

---

## Improvements Between Scan 1 and Scan 2

| Finding | Scan 1 | Scan 2 | Action Taken |
|---|---|---|---|
| phpinfo.php publicly accessible (High) | ✗ Found 11x | ✅ Resolved | phpinfo.php removed from all domains |
| phpinfo() Output Reporting (Medium) | ✗ Found 7x | ✅ Resolved | phpinfo.php removed from all domains |
| ICMP Timestamp Reply (Low) | ✗ Present | ⚠️ Still present | Accepted residual risk |
| TCP Timestamps (Low) | ✗ Present | ✅ No longer reported | Resolved or below filter threshold |

---

## Conclusion

All **High** and **Medium** findings were successfully remediated between Scan 1 and Scan 2.  
The most critical vulnerability — the publicly accessible `phpinfo.php` across all domains — has been removed.  
The only remaining finding is **ICMP Timestamp Reply** (Low, CVSS 2.1), which is classified as an accepted residual risk.

---

*Generated on March 22, 2026 | Scanner: OpenVAS / Greenbone Community Edition | Tool Version: GVM 26.18.1*

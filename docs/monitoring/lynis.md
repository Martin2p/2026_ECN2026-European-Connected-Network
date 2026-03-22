# Lynis Security Auditing

Lynis is used in the ECN2026 infrastructure to perform security audits and hardening checks on the VPS located in Germany.

The tool scans system configurations, identifies vulnerabilities, and provides actionable suggestions to improve the overall security posture.

---

# Deployment

Lynis is installed directly on the VPS via the system package manager.

```
sudo apt install lynis
```

No additional dependencies or containers are required. Lynis runs as a standalone shell-based tool.

---

# Usage

Audits are performed manually over a direct SSH connection to the VPS.

```
Admin workstation → SSH → VPS → lynis audit system
```

To run a full system audit:

```
sudo lynis audit system
```

For a quick scan without interactive pauses:

```
sudo lynis audit system --quick
```

---

# Output

Lynis generates three types of output:

```
Terminal        Color-coded results during the scan
Log file        /var/log/lynis.log
Report file     /var/log/lynis-report.dat
```

The report file can be parsed for specific findings:

```
grep "^warning" /var/log/lynis-report.dat
grep "^suggestion" /var/log/lynis-report.dat
grep "^hardening_index" /var/log/lynis-report.dat
```

After each scan, Lynis calculates a hardening index between 0 and 100. A higher score indicates a stronger security configuration.

---

# Scan Categories

During an audit, Lynis checks the following areas on the VPS:

* Kernel parameters and loaded modules
* User accounts, authentication and sudo configuration
* SSH daemon configuration
* Firewall rules (iptables / nftables)
* File permissions and mount options
* Running services and open ports
* Software packages and known vulnerabilities
* Logging and audit daemon status
* Cryptography and SSL/TLS certificates
* Malware scanners (ClamAV, rkhunter)

---

# Purpose within ECN2026

Lynis is used to audit and harden the VPS that serves as the backbone node of the ECN2026 network. It helps to:

* identify security weaknesses on the VPS after deployment
* verify firewall and SSH hardening measures
* track the hardening index over time
* gain hands-on experience with security auditing tools used in real infrastructure environments

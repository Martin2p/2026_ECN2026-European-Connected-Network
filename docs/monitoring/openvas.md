# OpenVAS / Greenbone Community Edition

OpenVAS (Open Vulnerability Assessment System) is used in the ECN2026 infrastructure
to perform vulnerability scans against network targets and identify security weaknesses.

It is part of the Greenbone Community Edition (GCE), which consists of several
interconnected components:

```
ospd-openvas → gvmd → gsad → Browser
```

ospd-openvas performs the actual scanning, gvmd manages scan configurations and results,
and gsad provides the web-based user interface.

---

# Deployment

OpenVAS runs as a set of Docker containers on the Germany VPS (Ubuntu 24.04),
using the official Greenbone Community Edition Docker Compose setup.

Default web UI port (remapped):

```
8443
```

Access to the OpenVAS interface is restricted to the WireGuard VPN network.

```
Internet
│
└── blocked
│
▼
WireGuard VPN (10.100.0.x)
│
└── OpenVAS Web UI (https://10.100.0.1:8443)
```

---

# Installation

OpenVAS was deployed using the official Greenbone setup script:

```bash
curl -f -L https://greenbone.github.io/docs/latest/_static/setup-and-start-greenbone-community-edition.sh \
  -o setup-and-start-greenbone-community-edition.sh

bash setup-and-start-greenbone-community-edition.sh
```

The script pulls all required container images and starts the full Greenbone stack
via Docker Compose.

> **Note:** APT-based installation (`apt install openvas`) was attempted first but
> abandoned due to missing `gvm` components in the standard repositories.
> The official Docker image is the recommended and supported installation method.

---

# Port Configuration

The Germany VPS already runs an Nginx web server on port `443`.
To avoid a port conflict, the Greenbone Security Assistant (gsad) was remapped
to port `8443`, bound exclusively to the WireGuard interface:

```yaml
services:
  gsa:
    ports:
      - "10.100.0.1:8443:443"
```

This ensures:

* No conflict with Nginx on port 443
* OpenVAS is not reachable from the public internet
* Access is only possible via WireGuard VPN

---

# Scan Configuration

### Target Setup

| Setting | Value |
|---|---|
| Host | Target IP address |
| Max. concurrent hosts | 1 |
| Concurrent scanning across IPs | Yes |
| Reverse lookup | No |
| Alive test | Consider Alive |
| Port list | All IANA assigned TCP and UDP |

> **Note:** Setting Alive Test to `Consider Alive` ensures the scan runs even if
> ICMP ping is blocked on the target. This is important for hardened servers
> that drop ping requests.

### Scan Policies

OpenVAS uses scan configurations to define the depth and type of checks performed.
The recommended configuration for a full assessment is:

```
Full and Fast
```

This applies all available Network Vulnerability Tests (NVTs) while keeping
scan duration reasonable.

---

# Purpose within ECN2026

OpenVAS enables active security assessment of the ECN2026 infrastructure and helps to:

* identify known vulnerabilities on exposed services
* verify the effectiveness of firewall rules and hardening measures
* practice vulnerability management workflows used in professional security operations
* complement passive monitoring (Grafana / Prometheus) with active security scanning
* prepare for later SIEM integration (Wazuh) in Phase 3

---

# Related Components

* **WireGuard** – required for secure access to the OpenVAS web UI
* **Nginx** – runs on port 443 alongside OpenVAS (port 8443)
* **Fail2Ban** – active on the VPS, complements OpenVAS findings with intrusion detection
* **Grafana / Prometheus** – system monitoring stack running on the same host

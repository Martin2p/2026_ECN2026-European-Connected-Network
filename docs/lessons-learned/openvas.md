# Lessons Learned – OpenVAS / Greenbone Installation

**Project:** ECN2026 – European Connected Network
**Phase:** 3 – Security & Operations
**Date:** 2026-03
**Author:** Martin

---

## Overview

This document captures the problems encountered during the installation and
configuration of OpenVAS (Greenbone Community Edition) on the ECN2026 Germany
VPS node, along with the solutions applied and key takeaways.

---

## Problem 1 – APT Package Installation: Missing `gvm` Package

### Context
Attempted to install OpenVAS via the standard APT package manager on the
Debian/Ubuntu-based VPS.

### Problem
The APT installation completed partially but the `gvm` (Greenbone Vulnerability
Manager) component was missing or not correctly included in the available
package repositories. The installation could not be completed successfully via
APT alone.

### Error Indication
After running:
```bash
apt install openvas
```
The `gvm` command was not available post-installation, making it impossible to
initialize and manage the vulnerability scanner properly.

### Solution
Switched from the APT package installation to the **official Greenbone
Community Container image** (Docker-based deployment).

```bash
# Using the official Greenbone docker-compose setup
curl -f -L https://greenbone.github.io/docs/latest/_static/setup-and-start-greenbone-community-edition.sh -o setup-and-start-greenbone-community-edition.sh
bash setup-and-start-greenbone-community-edition.sh
```

The Docker image includes all required components (`gvmd`, `gsad`, `ospd-openvas`)
pre-configured and maintained by Greenbone.

### Lesson Learned
> For complex security tools with multiple interdependent components, **official
> container images are more reliable than APT packages**, especially on
> non-standard or older repository versions. Always check the official
> documentation first before using distribution packages.

---

## Problem 2 – Port Conflict on 443: Nginx Already Running

### Context
The ECN2026 Germany node already runs an Nginx web server handling HTTPS traffic
on the standard port `443`. OpenVAS (Greenbone Security Assistant / `gsad`) also
defaults to port `443` for its web interface.

### Problem
Both Nginx and the Greenbone Security Assistant (`gsad`) attempted to bind to
port `443`, causing a conflict. OpenVAS could not start its web UI because the
port was already occupied by Nginx.

### Solution
Reconfigured the Greenbone Security Assistant to use port **`8443`** instead of
the default `443`.

In the Docker Compose configuration, the port mapping was adjusted:

```yaml
services:
  gsa:
    ports:
      - "10.100.0.1:8443:443"   # Bind to WireGuard interface IP only, port 8443 → container port 443
```

The OpenVAS web interface is now accessible via:
```
https://10.100.0.1:8443
```

Nginx continues to serve the main web server on port `443` without conflict.

### Lesson Learned
> In a **multi-service environment**, always audit which ports are already in use
> before deploying a new service. Use `ss -tlnp` or `netstat -tlnp` to check
> active listeners.
>
> Standard tool defaults (like port 443) **will conflict** in environments where
> a web server is already running. Plan port assignments in advance and document
> them centrally.

---

## Summary Table

| # | Problem | Root Cause | Solution |
|---|---------|------------|----------|
| 1 | `gvm` missing after APT install | Incomplete APT package | Switched to Greenbone Docker image |
| 2 | Port 443 conflict with Nginx | Both services default to 443 | Remapped OpenVAS to port 8443 |

---

## Related Configuration Notes

- OpenVAS runs fully containerized via Docker Compose
- Nginx handles all public-facing HTTPS traffic on port 443
- OpenVAS web UI accessible internally via port 8443
- WireGuard VPN recommended for accessing the OpenVAS interface securely

---

## References

- [Greenbone Community Edition Docs](https://greenbone.github.io/docs/latest/)
- [Greenbone Docker Setup Script](https://greenbone.github.io/docs/latest/22.4/container/index.html)

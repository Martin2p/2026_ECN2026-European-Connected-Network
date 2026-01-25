# WireGuard Security Hardening
*ECN2026 – VPN Security Baseline (VPS + Raspberry Pi Gateway)*

This document describes the security measures applied to the WireGuard setup to make it **stable, auditable, and production-oriented**.

---

## Scope

**In scope**
- WireGuard VPN (VPS hub + Raspberry Pi site gateway)
- SSH access model and restrictions
- Firewall policy (ingress/egress)
- Key handling and system-level hardening
- Basic monitoring signals relevant for security

**Out of scope**
- Advanced IDS/IPS
- SIEM integration
- Zero Trust / device posture checks

---

## Threat Model (Practical)

| Threat | Example | Mitigation |
|------|---------|------------|
| Internet scanning & brute force | SSH password attempts on VPS | **Key-only SSH**, firewall restrictions, Fail2Ban |
| Unauthorized VPN peer usage | Stolen client config | **Least-privilege AllowedIPs**, key rotation process |
| Misrouting / network exposure | LAN routes advertised too broadly | Documented routing design, explicit AllowedIPs |
| Lateral movement | Compromised client reaching sensitive LAN services | Segmentation roadmap, service-specific firewall rules |

---

## Baseline Security Controls

### 1) SSH Hardening (VPS + Raspberry Pi)

- **Password authentication disabled** (key-based login only)
- SSH access restricted as much as possible (source IP / VPN-only model where applicable)
- Only required users allowed to log in via SSH
- Strong key types preferred (e.g., Ed25519)

**Operational note:** SSH is treated as an administrative interface and must be protected at least as strictly as the VPN.

---

### 2) Firewall Policy (Ingress First)

**VPS (public-facing)**
- Allow WireGuard UDP port (e.g., `51820/udp`) from the Internet
- Allow SSH only from trusted sources (preferred: VPN / allowlist)
- Deny everything else by default

**Raspberry Pi (site gateway)**
- WireGuard interface allowed
- LAN forwarding controlled explicitly
- Default deny inbound from untrusted networks

> Principle: **Expose the minimum required surface area** on the public VPS.

---

### 3) Fail2Ban (VPS)

- Fail2Ban enabled to reduce the impact of repeated authentication attempts
- Focus on SSH protection (common attack surface)
- Logging retained to support incident review / troubleshooting

---

### 4) WireGuard Configuration Hardening

- **Least-privilege routing** via `AllowedIPs`
  - Site gateway peer: only transit IP (`/32`) + required LAN subnets
  - Full-tunnel client: `0.0.0.0/0` only if explicitly required
- Peer keys stored with correct file permissions
- No “catch-all” routes unless intended and documented

**Rationale:** In WireGuard, `AllowedIPs` act as both:
- a routing table entry, and
- an authorization boundary

---

### 5) System Hardening (VPS / Pi)

- Regular security updates applied
- Services minimized (only necessary daemons running)
- Backups exist for critical configs (WireGuard + SSH)

---

## Recommended Configuration Conventions

### Naming & File Layout

- Source configs stored in repo as **templates** (no secrets)
- Secrets (private keys) never committed

Suggested repo layout:
docs/architecture/wireguard-security-hardening.md
docs/architecture/wireguard-site-to-site.md

docs/diagrams/vpn/wireguard-site-to-site.png

---

## Security Validation Checklist

Use this checklist after changes or maintenance:

- [ ] SSH password login disabled
- [ ] SSH reachable only from trusted sources (preferably VPN)
- [ ] WireGuard UDP port reachable and limited to required interface
- [ ] `AllowedIPs` follows least privilege (no unintended subnets)
- [ ] IP forwarding only enabled where required (site gateway)
- [ ] Firewall default deny rules in place
- [ ] Fail2Ban running and logging
- [ ] Config backups exist (without private keys in the repo)

---

## Notes / Future Improvements

- Add per-service firewall rules between VPN and LAN (segmentation)
- Add lightweight monitoring (handshake age, endpoint changes)
- Document key rotation procedure and emergency revocation steps

- ---

## Key Rotation & Incident Response

This section describes the **operational procedures** to follow in case of
key compromise, suspected misuse, or regular maintenance.

The goal is to ensure **rapid containment**, **minimal downtime**, and
**auditable recovery**.

---

## When to Rotate Keys

Key rotation is required in the following situations:

- Suspected or confirmed exposure of a WireGuard configuration file
- Loss or theft of a client device
- Unauthorized access attempt detected in logs
- Role change (device repurposed or decommissioned)
- Periodic maintenance (recommended: every 6–12 months)

---

## Incident Response – Compromised Peer

### 1) Immediate Containment

- Identify the affected peer by **public key**
- Remove or comment out the peer from the WireGuard configuration
- Apply the configuration:
  - Restart WireGuard interface
  - Or use `wg set` for live removal

**Result:**  
The compromised peer is instantly disconnected.

---

### 2) Key Rotation Procedure

Perform a full key rotation for the affected peer:

1. Generate a new WireGuard key pair on the affected device
2. Update the peer configuration on the VPS / gateway
3. Review and re-apply **least-privilege `AllowedIPs`**
4. Bring the tunnel back up and verify handshake

> Old keys must be considered permanently invalid.

---

### 3) Validation After Rotation

- Verify active peers using `wg show`
- Confirm correct endpoint and handshake timestamp
- Test:
  - ICMP connectivity
  - SSH access (if applicable)
  - LAN reachability

---

## Regular Key Rotation (Preventive)

For preventive maintenance:

- Rotate keys during a defined maintenance window
- Rotate **one peer at a time** to avoid loss of connectivity
- Document:
  - date
  - affected peer
  - reason for rotation

---

## Logging & Audit Trail

For incident analysis and auditability:

- Keep WireGuard status outputs (`wg show`) during incidents
- Retain SSH and firewall logs for correlation
- Document actions taken and timestamps

> No private keys or sensitive material should ever be stored in the repository.

---

## Emergency Recovery Scenario

If the VPS is suspected to be compromised:

1. Revoke **all WireGuard peers**
2. Rotate all keys
3. Rebuild the VPS from a clean image if required
4. Reapply firewall and hardening baseline
5. Reintroduce peers incrementally

This ensures **trust re-establishment from a known-good state**.

---

## Summary

- WireGuard peers are treated as **independent trust units**
- Compromise of one peer does not affect others
- Rapid revocation and key rotation minimize impact
- Procedures are documented to avoid ad-hoc decisions under pressure


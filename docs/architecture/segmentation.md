# Network Segmentation – Phase 2 (CH Core Site)

## Context

This document describes the **initial network segmentation design** implemented
as part of **Phase 2** of the *ECN2026 – European Connected Network* roadmap.

At the current stage, **full VLAN control is only available at the Switzerland (CH) core site**.
The Sweden (SE) site is operated behind an **ISP-managed provider router (CPE)** which does not allow VLAN configuration.

---

## Design Decision

- Network segmentation is implemented **exclusively at the CH core site**
- The SE site remains a **flat Layer-2 access network**
- Inter-VLAN routing and policy enforcement are centralized at the CH router

This approach enables incremental expansion without requiring architectural redesign.

---

## VLAN Overview (CH Site)

| VLAN ID | Name    | Purpose                         | Subnet             |
|-------:|---------|---------------------------------|--------------------|
| 10     | CLIENTS | Trusted client devices          | 192.168.101.0/24   |
| 20     | GUEST   | Untrusted / guest devices       | 192.168.120.0/24   |

The intentionally small number of VLANs is sufficient to demonstrate:

- IEEE 802.1Q VLAN tagging
- Access vs trunk port configuration
- Inter-VLAN routing
- Policy-based network segmentation

---

## Switching Design (CH)

- **Router uplink**: trunk port carrying all VLANs (tagged)
- **Client ports**: access ports assigned to VLAN 10
- **Guest ports**: access ports assigned to VLAN 20

No wireless access points are deployed at this stage.
Segmentation is implemented exclusively via wired switch ports.

---

## Inter-VLAN Policy Model

Default security model: **deny by default**, allow explicitly.

### Allowed Traffic

- VLAN 10 → Internet
- VLAN 10 → Site-to-Site VPN (SE)
- VLAN 20 → Internet

### Denied Traffic

- VLAN 20 → VLAN 10
- VLAN 20 → Site-to-Site VPN
- VLAN 20 → Router management interfaces

This ensures a clear separation between trusted and untrusted devices.

---

## Site-to-Site VPN Integration

- Only **selected VLAN subnets** are advertised via WireGuard
- Guest traffic is explicitly excluded from VPN routing
- Firewall rules on the gateway enforce per-VLAN VPN access

This prevents lateral movement from untrusted segments across sites.

---

## Sweden Site (SE) Limitation

The Sweden site currently operates behind an **ISP-managed router** with no VLAN capability.

Design implications:

- Single flat LAN
- No local segmentation possible
- Routed access via Site-to-Site VPN only

This limitation is documented intentionally and treated as a design constraint
rather than a misconfiguration.

---

## Future Expansion

The current design is fully extensible:

- Additional VLANs (MGMT, IoT, LAB) can be added without restructuring
- Replacement of the SE router enables mirrored segmentation
- Wireless access points can map SSIDs directly to existing VLANs

---

## Summary

This segmentation step demonstrates:

- Practical VLAN deployment with limited hardware
- Centralized security enforcement
- Handling of real-world constraints
- Incremental and scalable network architecture

The design prioritizes clarity, documentation and architectural correctness over complexity.

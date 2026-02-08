## WireGuard on MikroTik RouterOS – Explicit Routing Required

### Context
In the ECN2026 setup, WireGuard is used in a hub-and-spoke topology with a Linux VPS as the central hub and a MikroTik RB5009 as the site router for the CH network.

During testing, a critical RouterOS-specific behavior was identified that differs fundamentally from Linux-based WireGuard implementations.

---

### Key Finding (Important)
On **MikroTik RouterOS**, WireGuard **does not automatically install kernel routes** for remote peers.

> **AllowedIPs ≠ routing entry on RouterOS**

As a result, WireGuard traffic may arrive at the router, but replies are silently dropped **before firewall processing**, because RouterOS has no explicit return route.

---

### Symptoms
- WireGuard tunnel is up (`last-handshake` updates, RX/TX counters increase)
- VPS receives ICMP requests from remote peers (verified via `tcpdump`)
- Remote peers can reach the VPS (`10.100.0.1`)
- **Remote peers cannot reach the RB5009 (`10.100.0.19`) or CH-LAN**
- No firewall logs on the RB5009
- No visible packet drops in firewall or sniffer tools

---

### Root Cause
RouterOS requires **explicit routing entries** for each remote WireGuard peer (or subnet).  
Without a route, reply packets are discarded at the routing stage.

---

### Solution
For every WireGuard peer **outside of the local CH-LAN**, add an explicit route on the RB5009.

#### Example: Smartphone (Road Warrior)
```mikrotik
/ip route add dst-address=10.100.0.22/32 gateway=wg0 comment="WG Smartphone"
```

#### Example: Raspberry Pi (SE site)
```mikrotik
/ip route add dst-address=10.100.0.23/32 gateway=wg0 comment="WG Smartphone"
```

After adding the route, connectivity is immediately restored:

- Ping to RB5009 WireGuard IP works
- Access to CH-LAN works
- SSH over VPN works

---

### Alternative (Not Recommended for Large Setups)

A single aggregate route is possible:

```mikrotik
/ip route add dst-address=10.100.0.0/24 gateway=wg0
```

However, this reduces clarity and makes debugging harder.
For ECN2026, explicit /32 routes per peer are preferred.

### Design Rule (ECN2026)

- WireGuard peers are not routes

- Each remote peer requires an explicit route

- Linux (wg-quick) installs routes automatically

- MikroTik RouterOS does not

### Practical Rule of Thumb

> **If a WireGuard peer is not physically behind the RB5009, it needs a route.**

This applies to:

- Smartphones

- Laptops on the road

- Remote Raspberry Pi sites

- Any external WireGuard client

### Lessons Learned

- The issue is not firewall- or NAT-related

- The issue is not visible via firewall logging

- The behavior is RouterOS-specific and non-obvious

- Explicit routing is mandatory for WireGuard on MikroTik

### Status

✔ Issue identified

✔ Root cause confirmed

✔ Solution implemented

✔ Documented for ECN2026

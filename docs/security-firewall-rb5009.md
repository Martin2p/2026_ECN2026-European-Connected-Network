# RB5009 – Firewall Configuration (MikroTik RouterOS)

## Overview

This document describes the firewall ruleset applied to the MikroTik RB5009 router,
which acts as the central security boundary in the home lab setup.

**Network position:**
```
ISP Router → RB5009 → Managed Switch → LAN Devices
                ↕
            WireGuard VPN (wg0)
                ↕
         Remote Site / VPS Hub
```

**Interfaces used:**

| Interface | Role |
|-----------|------|
| `ether2` | WAN – uplink to ISP router |
| `bridge-lan` | LAN – internal network segment |
| `wg0` | WireGuard VPN tunnel |

> ⚠️ **Note:** Real IP addresses, subnet ranges and device names have been
> intentionally omitted from this document. Use your own values from your
> network plan when applying these rules.

---

## Design Principles

- **Default-deny** approach: all traffic not explicitly permitted is dropped
- **Stateful filtering**: established/related connections are always allowed
- **Management access** is restricted to a single trusted workstation (by IP) and only accessible via LAN or VPN — never from WAN
- **VPN segmentation**: remote site traffic is not permitted to roam freely into the local LAN; only specific ports/services are allowed across the tunnel
- **WAN access to the router itself** is fully blocked (no exposed services)

---

## Lessons Learned

During initial setup, the forward rule for LAN internet access was mistakenly
set to `out-interface=bridge-lan` (the internal interface) instead of
`out-interface=ether2` (the WAN interface). This caused all outbound internet
traffic from LAN clients to be dropped silently.

**Diagnosis steps used:**
```routeros
/interface print
/ip route print
```
The default route (`0.0.0.0/0`) pointed to the WAN gateway via `ether2`,
which identified the correct interface to use in the forward rule.

---

## Input Chain

Protects the router itself from unauthorized access.

```routeros
/ip firewall filter

# Allow already-established and related connections
add chain=input connection-state=established,related action=accept \
    comment="Allow established/related input"

# Drop invalid packets
add chain=input connection-state=invalid action=drop \
    comment="Drop invalid packets"

# Allow ICMP (ping) from LAN only
add chain=input protocol=icmp src-address=<LAN-SUBNET> action=accept \
    comment="Allow ping from LAN"

# Allow Winbox management only from trusted workstation (LAN IP)
add chain=input src-address=<TRUSTED-WORKSTATION-LAN-IP> protocol=tcp \
    dst-port=8291 action=accept \
    comment="Winbox from trusted workstation only"

# Allow Winbox management from trusted workstation via VPN
add chain=input src-address=<TRUSTED-WORKSTATION-VPN-IP> protocol=tcp \
    dst-port=8291 action=accept \
    comment="Winbox via VPN"

# Allow SSH from LAN
add chain=input src-address=<LAN-SUBNET> protocol=tcp dst-port=22 \
    action=accept comment="SSH from LAN"

# Allow SSH from VPN
add chain=input src-address=<VPN-SUBNET> protocol=tcp dst-port=22 \
    action=accept comment="SSH via VPN"

# Allow WireGuard VPN port
add chain=input protocol=udp dst-port=<WIREGUARD-PORT> action=accept \
    comment="WireGuard VPN inbound"

# Allow DHCP from LAN
add chain=input src-address=<LAN-SUBNET> protocol=udp dst-port=67 \
    action=accept comment="DHCP server"

# Allow DNS from LAN
add chain=input src-address=<LAN-SUBNET> protocol=udp dst-port=53 \
    action=accept comment="DNS from LAN UDP"
add chain=input src-address=<LAN-SUBNET> protocol=tcp dst-port=53 \
    action=accept comment="DNS from LAN TCP"

# Drop everything else — must remain the last rule
add chain=input action=drop comment="Drop everything else"
```

---

## Forward Chain

Controls traffic passing *through* the router between network segments.

```routeros
# Allow established/related forwarded traffic
add chain=forward connection-state=established,related action=accept \
    comment="Allow established/related forward"

# Drop invalid forwarded packets
add chain=forward connection-state=invalid action=drop \
    comment="Drop invalid forward"

# Allow LAN to reach the internet (via WAN interface)
add chain=forward src-address=<LAN-SUBNET> out-interface=ether2 \
    action=accept comment="LAN to internet"

# Allow VPN tunnel traffic to reach the internet
add chain=forward in-interface=wg0 out-interface=ether2 action=accept \
    comment="VPN to internet"

# Allow return traffic from WAN back to LAN
add chain=forward in-interface=ether2 connection-state=established,related \
    action=accept comment="Return traffic to LAN"

# Allow only specific service from remote site to trusted LAN host
# Example: remote desktop to one specific workstation
add chain=forward src-address=<REMOTE-SITE-VPN-IP> \
    dst-address=<TARGET-LAN-HOST-IP> protocol=tcp dst-port=<ALLOWED-PORT> \
    action=accept comment="Specific access remote to LAN"

# Block VPS/hub from accessing the LAN directly
add chain=forward src-address=<VPS-VPN-IP> dst-address=<LAN-SUBNET> \
    action=drop comment="Block VPS direct LAN access"

# Block remote site from freely roaming the LAN
add chain=forward src-address=<VPN-SUBNET> dst-address=<LAN-SUBNET> \
    action=drop comment="Block remote free LAN access"

# Default deny — must remain the last rule
add chain=forward action=drop comment="Forward default drop"
```

---

## Management Access Restriction (Winbox Service)

Restricts the Winbox service so it is only reachable from the LAN and VPN —
never from the public internet.

```routeros
/ip service set winbox address=<LAN-SUBNET>,<VPN-SUBNET>
```

---

## Why Not MAC Address Filtering?

MAC address filtering was considered but rejected for the following reasons:

- MAC addresses are only visible within the local Layer-2 segment
- They can be trivially spoofed on any OS (Windows, Linux, macOS)
- An attacker already inside the LAN can clone any MAC address within seconds

**Conclusion:** IP-based restrictions combined with VPN-only access provide
meaningfully stronger guarantees than MAC filtering.

---

## Related Documentation

- [MikroTik Firewall Basics](https://help.mikrotik.com/docs/display/ROS/Firewall)
- [WireGuard on RouterOS](https://help.mikrotik.com/docs/display/ROS/WireGuard)
- [RouterOS Safe Mode](https://help.mikrotik.com/docs/display/ROS/Safe+Mode)

---

*Document maintained as part of the ECN2026 home lab project.*

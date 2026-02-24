# 🇸🇪 SE Site – Raspberry Pi WireGuard Gateway (ECN2026)

## Overview

This document describes the hardened Raspberry Pi acting as the **Sweden Site Gateway** in the ECN2026 project.

The Raspberry Pi provides:

- WireGuard site-to-site connectivity (SE ↔ VPS ↔ CH)
- Routing between SE-LAN and VPN
- nftables firewall (default deny)
- Hardened SSH access (key-based only)
- Secured WireGuard key material

The SE site operates in **site-to-site mode**, not full tunnel for LAN clients.

---

# 1. Network Architecture
Smartphone (WG Client)
│
▼
VPS (Hub)
│
▼
Raspberry Pi (SE Gateway)
│
▼
WLAN (192.168.50.0/24)
│
▼
Netbook

### Addressing

| Component        | Network               |
|------------------|----------------------|
| WireGuard Net    | 10.100.0.0/24        |
| SE-LAN           | 192.168.50.0/24      |
| CH-LAN           | 192.168.88.0/24      |

---

# 2. WireGuard Configuration (Raspberry)

`/etc/wireguard/wg0.conf`

```ini
[Interface]
Address = 10.100.0.21/24
PrivateKey = <hidden>


[Peer]
PublicKey = <VPS public key>
Endpoint = <VPS-IP>:51820
AllowedIPs = 10.100.0.0/24, 192.168.88.0/24
PersistentKeepalive = 25
```
Important Fix

The subnet 10.100.0.0/24 was initially missing from AllowedIPs, causing asymmetric routing.
Adding the full WireGuard subnet restored proper return paths and peer connectivity.

---

# 3. nftables Firewall (Default Deny)

`/etc/nftables.conf`

```
flush ruleset

table inet filter {

    chain input {
        type filter hook input priority filter;
        policy drop;

        iif lo accept;
        ct state established,related accept;

        # WireGuard management
        iif "wg0" accept;

        # SSH (LAN)
        iif "wlan0" tcp dport 22 accept;

        # DHCP
        iif "wlan0" udp dport 67 accept;

        # DNS
        iif "wlan0" udp dport 53 accept;
        iif "wlan0" tcp dport 53 accept;

        # ICMP
        ip protocol icmp accept;
    }

    chain forward {
        type filter hook forward priority filter;
        policy drop;

        ct state established,related accept;

        # VPN <-> WLAN
        iif "wg0"  oif "wlan0" accept;
        iif "wlan0" oif "wg0"  accept;
    }

    chain output {
        type filter hook output priority filter;
        policy accept;
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat;
        oif "wg0" masquerade;
    }
}
```
Security principles applied:

- Default DROP (input + forward)

- Explicit service allow rules

- Stateful inspection via conntrack

- NAT restricted to WireGuard interface

---

# 4. Raspberry Pi Hardening

### 4.1 SSH Hardening

`/etc/ssh/sshd_config`
```
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
ChallengeResponseAuthentication no
UsePAM yes
```

Measures implemented:

- Password login disabled

- Public key authentication enforced

- Root login prohibited

- SSH accessible only via trusted interfaces

- Service restarted and validated

### 4.2 WireGuard File Protection

All WireGuard configuration and key material secured:

```
sudo chown root:root /etc/wireguard/wg0.conf
sudo chmod 600 /etc/wireguard/wg0.conf
```

Directory protection:
```
sudo chown -R root:root /etc/wireguard
sudo chmod -R 600 /etc/wireguard/*
```

Result:

- Private keys readable only by root

- No world-readable configuration files

- Reduced attack surface

### 4.3 Removal of Unnecessary Components

- Unused packages removed

- No unnecessary services enabled

- No exposed WAN services

- Minimal attack surface maintained

### 4.4 Kernel Hardening

Applied sysctl parameters:
```
net.ipv4.conf.all.rp_filter=1
net.ipv4.conf.default.rp_filter=1
net.ipv4.icmp_echo_ignore_broadcasts=1
net.ipv4.icmp_ignore_bogus_error_responses=1
net.ipv4.conf.all.accept_redirects=0
net.ipv4.conf.all.send_redirects=0
```

Purpose:

- Reverse path filtering (anti-spoofing)

- Disable ICMP redirects

- Harden network stack behavior

---

# 5. Validation Tests
Connectivity

Smartphone → SE Netbook ✔

Netbook → VPS ✔

Netbook → Internet via WireGuard ✔

SE ↔ CH LAN communication ✔

Verification Commands
wg show
ip route
nft list ruleset

---

# 6. Security Posture Summary

The SE site gateway now operates with:

- Encrypted site-to-site tunnel

- Stateful firewall (default deny)

- Hardened SSH

- Protected key material

- Controlled routing domains

- Minimal exposed services

-> This configuration reflects a production-style small-site VPN gateway design.

---

# 7. Known Limitations

Single Point of Failure: VPS Hub

- No redundancy implemented

- DNS currently external (own recursive resolver planned)

- No IDS/IPS yet (OpenVAS / monitoring phase upcoming)

-> Project Status

✔ Site-to-Site WireGuard operational
✔ nftables firewall implemented
✔ SSH hardened
✔ Kernel hardened
✔ WireGuard key material secured
✔ Routing validated

Next Phase: Security validation and vulnerability assessment (OpenVAS).

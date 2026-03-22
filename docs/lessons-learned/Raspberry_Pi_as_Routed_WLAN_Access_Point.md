# Lessons Learned – Raspberry Pi as Routed WLAN Access Point (Sweden Site)

## 1. Clear Interface Role Separation Is Critical

The Raspberry Pi was configured with:

- **eth0** → WAN uplink (connected via cable to upstream router / internet)
- **wlan0** → LAN interface (acting as Wi-Fi Access Point)

The most important architectural principle was strict separation of interface roles:

- `eth0` must remain DHCP client (or statically routed toward upstream gateway).
- `wlan0` must operate with a static private IP address and serve as gateway for wireless clients.

Misconfiguration (e.g., both interfaces using DHCP or incorrect gateway assignment) immediately resulted in routing conflicts.

## 2. Static IP Configuration for AP Interface

`wlan0` required:

- Static IP (e.g., `192.168.x.1/24`)
- No default gateway
- No DNS resolver override

The default route must only exist on `eth0`.

Adding a gateway to `wlan0` caused asymmetric routing and broke internet connectivity for clients.

> **Key insight:** Only one default route in a simple NAT topology.

## 3. IP Forwarding Is Not Optional

Enabling kernel IP forwarding was mandatory:

```
net.ipv4.ip_forward=1
```

Without IP forwarding:

- Clients could connect to Wi-Fi
- Clients received IP addresses
- But no traffic was routed to `eth0`

> **Lesson:** Layer 3 forwarding must be explicitly enabled on Linux.

## 4. NAT (Masquerading) Is Required for Internet Access

Since the wireless clients were in a private subnet, source NAT using iptables was required:

```bash
iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

Without NAT:

- Routing worked internally
- No return traffic from the internet

> **Lesson:** Routing alone is insufficient. Private-to-public traffic requires masquerading.

## 5. hostapd and dnsmasq Integration

Two services were required:

- **hostapd** → Wi-Fi AP functionality
- **dnsmasq** → DHCP server for WLAN clients

Important observations:

- `hostapd` fails silently if regulatory domain or driver mode is incorrect.
- `dnsmasq` must listen only on `wlan0`.
- Overlapping DHCP scopes with upstream router must be avoided.

> **Lesson:** Service binding to the correct interface prevents cross-network contamination.

## 6. Regulatory Domain Configuration Matters

Wi-Fi channel selection failed initially because the country code was not set properly.

Correct configuration:

```
country=SE
```

> **Lesson:** Wireless behavior depends on regulatory settings. Always define country code explicitly.

## 7. Firewall Rules Must Be Persistent

Temporary iptables rules worked during testing but were lost after reboot.

**Solution:** Use persistent rule storage (e.g., `iptables-persistent` or `netfilter-save` approach).

> **Lesson:** Always validate configuration after reboot. Lab success does not equal production readiness.

## 8. Default Gateway Logic on Clients

Clients must:

- Receive default gateway = Raspberry Pi (`wlan0` IP)
- Receive DNS (either upstream or local forwarder)

If DHCP did not distribute correct gateway information, internet access failed even though routing and NAT were correct.

> **Lesson:** DHCP correctness directly impacts routing behavior.

## 9. Simplicity Beats Overengineering

Initial attempts included advanced firewall policies, multiple routing experiments, and additional filtering rules — these increased troubleshooting complexity.

Final working architecture:

- Single NAT rule
- Basic FORWARD accept rules
- Clear subnet separation
- Minimal services

> **Lesson:** Build minimal working configuration first. Harden later.

## 10. Architectural Outcome

Final topology:

```
[Internet]
     |
 Upstream Router
     |
   (eth0)
 Raspberry Pi
   (wlan0)
     |
  Wi-Fi Clients
```

The Raspberry Pi successfully:

- Acted as Wi-Fi access point
- Provided DHCP services
- Routed traffic to WAN
- Performed NAT
- Maintained stable connectivity after reboot

---

## Strategic Takeaway

This exercise reinforced practical understanding of Linux routing, NAT mechanics, interface binding, DHCP behavior, wireless regulatory constraints, and service-layer troubleshooting methodology.

It demonstrated that a Raspberry Pi can function as a fully routed edge device when properly configured, provided that:

1. Interface roles are clearly defined
2. Forwarding and NAT are correctly implemented
3. Services are bound explicitly to intended interfaces
4. Persistence is validated across reboots

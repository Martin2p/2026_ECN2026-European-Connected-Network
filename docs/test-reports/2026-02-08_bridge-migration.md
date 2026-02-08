## 📅 2026-02-08 — RB5009 Bridge, SFP+ and WireGuard Configuration

### Summary
Today’s work focused on stabilizing the physical and logical connectivity between router and switch and further consolidating the core network architecture.

---

### Changes & Actions

- **SFP+ DAC cable deployed between router and switch**  
  → Copper Ethernet (Cat) link negotiation was unreliable  
  → SFP+ link established immediately and remained stable

- **Bridge (`bridge-lan`) created on the MikroTik RB5009**
  - LAN ports and SFP+ uplink integrated
  - Hardware offloading enabled
  - Foundation for future VLAN and VRF segmentation

- **DHCP configuration moved to the router**
  - DHCP server bound to `bridge-lan`
  - DNS distributed via the router
  - Clients behind the switch receive correct IP, gateway, and DNS

- **WireGuard configured**
  - Tunnel successfully established
  - Routing and name resolution verified
  - Prepared for integration into the ECN2026 site-to-site topology

---

### Result

- ✅ Stable physical connectivity (SFP+)
- ✅ Functional bridge-based LAN architecture
- ✅ DHCP and DNS operating correctly via router
- ✅ WireGuard operational
- ✅ Clean foundation for VLANs and VRFs

---

### ECN2026 Context
This step marks the transition from a port-based test setup to a **scalable campus/core design**, aligned with real-world enterprise network architectures.

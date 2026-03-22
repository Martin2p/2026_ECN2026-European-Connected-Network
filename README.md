🌍 Language: [English](README.md) | [Svenska](README.sv.md)

# ECN2026 – European Connected Network

## Technologies Used

- Linux
- Docker
- WireGuard
- Prometheus
- Grafana
- OpenVAS
- nftables
- Nginx


## Overview
ECN2026 is a personal network engineering project simulating a multi-site
European network across Sweden, Switzerland and Germany.

The goal is to design, implement and document a realistic campus and backbone
architecture using limited hardware and enterprise-grade concepts.

## Sites
- Sweden (Access Site)
- Switzerland (Core / Campus Site)
- Germany (Cloud / VPS Backbone)

## Key Topics
- Routing & subnetting
- Site-to-site VPN (WireGuard)
- Firewalling & basic hardening
- Monitoring & logging concepts

## Repository Structure
ECN2026/
├── docs/        # Architecture, concepts, explanations
├── sites/       # Per-site documentation
├── configs/     # Sanitized configuration examples
├── diagrams/    # Network diagrams (PNG / SVG)
└── notes/       # Learning notes & observations

## Status
🚧 Work in progress (2026)

## Project Roadmap

**Phase 1 – Core Connectivity**
- Basic connectivity ✔
- Routing fundamentals ✔
- Baseline firewall rules ✔

**Phase 2 – Segmentation & Multi-Site**
- Subnet-based network segmentation ✔
- Site-to-site VPN (WireGuard) ✔
- Traffic isolation and access control ✔

**Phase 3 – Security & Operations**
- Firewall and system hardening ✔
- Vulnerability scanning (OpenVAS, Lynis) ✔
- Monitoring & Observability (Prometheus, Grafana) ✔
- Logging / SIEM (Wazuh) 🚧
- Failover and resilience testing 🚧


## Network Overview Diagram

![ECN2026 Network Overview](diagrams/ECN2026_overview.png)

*This diagram is a simplified, abstract representation created for educational
and documentation purposes. It does not disclose real infrastructure details.*

*Diagram created using the Filius network simulator.*

## Disclaimer
This repository contains no real credentials, IP addresses or provider details.
All diagrams are simplified representations created using simulation tools.
They do not represent a production environment.

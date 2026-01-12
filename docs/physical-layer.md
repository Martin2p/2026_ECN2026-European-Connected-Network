> **TL;DR**
> - Copper Ethernet is used across the project
> - Self-crimped cables were validated under load
> - Flat cables showed instability at higher speeds
> - Monitoring relies on error counters and throughput
> - Validation is practical, not certification-grade


## Physical Layer Scope

This document describes the physical layer design principles, validation approach,
and limitations used within the ECN2026 project.

The physical layer covers all passive and active components responsible for signal transmission:

- Ethernet cabling (copper)
- Connectors and terminations
- Network interfaces (NICs)
- Switch ports and transceivers
- Physical link characteristics (speed, duplex, error counters)

Wireless technologies are out of scope for this document.

## Cabling Standards & Design Decisions

- Twisted-pair copper Ethernet is used throughout the project
- Focus on Cat5e / Cat6-class cabling  to balance realism, availability, and cost
- Self-crimped cables are intentionally used to validate installation quality
- Flat Ethernet cables are evaluated but treated as non-preferred for high-speed links

## Error Metrics & Monitoring

Physical layer quality is evaluated using the following indicators:

- CRC / FCS error counters
- RX / TX error statistics
- Link flaps and renegotiations
- Throughput stability under load
- Duplex and speed consistency

Monitoring is performed on both switch and host level using interface statistics,
error counters, and throughput measurements.

## Known Limitations

- No certified cable measurement (NEXT, FEXT, insertion loss)
- No professional cable certifier used
- Results represent practical validation, not formal certification

## Relation to OSI Model

Although primarily focused on OSI Layer 1, physical layer issues are evaluated in conjunction with:

- Layer 2 (Ethernet framing, CRC errors)
- Link negotiation behavior (auto-negotiation)

This reflects real-world troubleshooting, where Layer 1 and Layer 2 are tightly coupled.

## Validation & Testing

Practical cable validation tests are documented separately:
- [Link Instability Analysis (Layer 1/2) – 2026-01-11](test-reports/2026-01-11_link-instability-analysis.md)
- [Cable Validation Report – 2026-01-11](test-reports/2026-01-11_cable-validation.md)

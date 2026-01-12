# Ethernet Link Instability Analysis at 2.5 GbE

## Context
During normal lab operation, intermittent link instability was observed
when operating Ethernet connections at 2.5 GbE using flat Ethernet cables.

This document focuses on operational behavior and does not represent
a pure cable certification test.

## Test Environment
- Switch: TP-Link Omada SG3210X-M2
- Client NIC: 2.5 GbE Ethernet adapter
- Cable Type: Flat Ethernet cable (Cat5e)
- Alternative Cable: Round, shielded Ethernet cable
- Operating Mode: Auto-negotiation enabled

## Observed Behavior
- Intermittent link drops and reconnections at 2.5 GbE
- Temporary loss of network connectivity during operation
- No persistent CRC, RX or TX error counters observed
- Issue reproducible with flat cable, not observed with round cable

## Analysis
The observed instability did not present itself as classic physical-layer
errors (CRC, RX, TX), but rather as link negotiation and stability issues.

Possible contributing factors include:
- Cable construction unsuitable for higher-frequency signaling
- Reduced shielding and pair separation in flat cable design
- Increased sensitivity at 2.5 GbE compared to 1 GbE operation

## Mitigation
- Replacement of flat cable with round, shielded Ethernet cable
- Stable link operation observed after cable replacement

## Conclusion
While the flat Ethernet cable operated reliably at 1 GbE, it showed limitations
when used at 2.5 GbE. Cable construction and shielding play a significant role
in link stability at higher data rates.

For multi-gigabit Ethernet deployments, round, shielded cables are recommended.

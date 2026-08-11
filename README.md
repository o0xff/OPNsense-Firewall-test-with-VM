# OPNsense Firewall Configuration & Testing Project

## Overview

This project documents the setup and testing of a firewall using **OPNsense** in a virtualized lab environment. The goal is to demonstrate practical, hands-on understanding of firewall rule configuration, traffic filtering logic, and systematic testing methodology.

## Lab Environment

- **Firewall**: OPNsense (virtualized)
- **Client**: Windows VM on the LAN network
- **Testing tools**: `ping`, PowerShell `Test-NetConnection`
- **Network**: LAN / WAN interfaces configured on OPNsense

## Objectives

- Configure and validate firewall rules at increasing levels of granularity
- Move from basic allow/deny testing (ICMP) to protocol- and port-specific filtering (TCP)
- Document methodology, results, and troubleshooting for each test
- Build a reproducible reference for firewall rule behavior (default deny, rule ordering, logging)

## Tests

| # | Test | Focus | Status |
|---|------|-------|--------|
| 1 | [ICMP Block/Allow Test](tests/test-01-icmp-blocking.md) | Basic allow/deny filtering with `ping` | ✅ Completed |
| 2 | [TCP Port Filtering Test](tests/test-02-tcp-port-filtering.md) | Granular filtering — block Telnet (TCP/23) while allowing HTTPS (TCP/443) | ✅ Completed |
| 3 | NAT / Port Forwarding | Planned |
| 4 | VLAN Segmentation (LAN/DMZ) | Planned |

## Key Concepts Demonstrated

- **Stateful packet filtering** — rules matched by protocol, source, destination, and port
- **Default deny principle** — traffic is blocked unless explicitly allowed by a Pass rule
- **Rule ordering and specificity** — how the `Quick` flag and rule order affect which rule applies
- **Firewall logging** — using Live View logs to confirm rule matches
- **Control testing** — validating that unrelated traffic is unaffected by a new rule

## Repository Structure

```
opnsense-firewall-project/
├── README.md
└── tests/
    ├── test-01-icmp-blocking.md
    └── test-02-tcp-port-filtering.md
```

# Test 2: Granular Traffic Filtering by Protocol and Port (Blocking Telnet)

## Objective

Verify that the firewall can selectively block a specific protocol/port (TCP/23 — Telnet) without affecting the rest of the traffic (HTTPS, ICMP). This demonstrates granular, protocol-aware filtering rather than a blanket allow/deny.

## Test Scenario

- **Target host**: `8.8.8.8`
- **Blocked traffic**: TCP port 23 (Telnet) — an insecure protocol that transmits data in plaintext
- **Control traffic**: TCP port 443 (HTTPS) and ICMP (ping) — expected to remain unaffected

## Rule Configuration

Created in **Firewall → Rules → LAN**:

| Field | Value |
|---|---|
| Action | Block |
| Direction | In |
| Version | IPv4 |
| Protocol | TCP |
| Source | LAN network |
| Source Port | any |
| Destination | any |
| Destination Port | 23 (Telnet) |
| Log | Enabled |
| Description | "Test - block outbound telnet" |

## Test Method

From the LAN client VM (Windows), using PowerShell:

```powershell
# Should be blocked
Test-NetConnection -ComputerName 8.8.8.8 -Port 23

# Control: should succeed
Test-NetConnection -ComputerName 8.8.8.8 -Port 443

# Control: should succeed
ping 8.8.8.8
```

`Test-NetConnection` was used as the native PowerShell equivalent for TCP port testing.

## Results

| Port / Protocol | Expected | Result |
|---|---|---|
| TCP/23 (Telnet) | Blocked | ✅ `TcpTestSucceeded: False` |
| TCP/443 (HTTPS) | Allowed | ✅ `TcpTestSucceeded: True` |
| ICMP (ping) | Allowed | ✅ `PingSucceeded: True` |

## Verification via Logs

Rule matches were checked in **Firewall → Log Files → Live View**, filtered by `interface contains "LAN"`.

## Issues Encountered and Resolved

### 1. `nc` not available on Windows

**Symptom**: `'nc' is not recognized as an internal or external command`

**Cause**: netcat is a Linux/Unix utility and is not installed on Windows by default.

**Resolution**: used the built-in PowerShell cmdlet `Test-NetConnection -ComputerName <IP> -Port <port>`, which performs the equivalent TCP port check natively.

**Takeaway**: verify which OS a set of test commands is written for before running them — Linux and Windows expose different native tools for the same task.

### 2. No matching entry visible in Live View logs

**Symptom**: the rule was clearly working (confirmed by test results), but no corresponding entry appeared in the firewall log.

**Cause**: logs were initially checked on the wrong interface (WAN instead of LAN).

**Resolution**: applied a filter in Live View (`interface` → `contains` → `LAN`) to isolate relevant traffic.

**Takeaway**: firewall logs contain a large volume of background/system entries — filtering by interface, protocol, port, or rule label is essential to finding the relevant match.

### 3. All traffic stopped unexpectedly (including ping and HTTPS)

**Symptom**: after further rule inspection, both `Test-NetConnection -Port 443` and `ping` failed, even though neither had been targeted by any new rule.

**Cause**: two base "Pass" rules on the LAN interface (allowing general outbound IPv4/IPv6 traffic) had been accidentally disabled while reviewing rule settings.

**Resolution**: re-enabled the base Pass rules — all traffic was restored immediately.

**Takeaway (key concept)**: this was a direct, practical demonstration of the **default deny** principle — in the absence of an explicit Pass rule, all traffic is blocked by default, even without any specific Block rule targeting it. This is a common real-world cause of unexpected network outages and is worth documenting as a standalone lesson.

## Conclusion

The firewall rule successfully filtered traffic at a granular level — blocking only the targeted protocol and port (TCP/23) while leaving unrelated traffic (TCP/443, ICMP) unaffected. This confirms a working understanding of:

- Protocol- and port-based rule matching in OPNsense
- The importance of control tests to validate that a rule's scope is as narrow as intended
- The default deny principle and its practical impact on troubleshooting

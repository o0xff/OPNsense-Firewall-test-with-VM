# Test 1: Basic ICMP Block/Allow Filtering

## Objective

Verify that OPNsense can allow or block basic ICMP (ping) traffic between the LAN client and an external host, confirming that the firewall is correctly positioned in the traffic path and that basic Pass/Block rules take effect.

## Test Scenario

- **Target host**: `8.8.8.8` (Google Public DNS)
- **Traffic type**: ICMP (ping)
- **Expected behavior**: traffic is allowed or blocked depending on the active firewall rule for ICMP on the LAN interface

## Configuration

| Field | Value |
|---|---|
| Interface | LAN |
| Action | Block / Pass (toggled for testing) |
| Direction | In |
| Protocol | ICMP |
| ICMP type | Echo Request |
| Source | LAN network |
| Destination | any |

### Rule Configuration


<img width="895" height="483" alt="image" src="https://github.com/user-attachments/assets/f44056f3-f1b0-42ef-bb2b-e1451e5042aa" />


## Test Method

From the LAN client VM:

```powershell
ping 8.8.8.8
```

## Results

| Rule state | Expected | Result |
|---|---|---|
| ICMP rule: Block | Ping fails (timeout) | ✅ Confirmed — 100% packet loss |
| ICMP rule: Pass / disabled | Ping succeeds | ✅ Confirmed — replies received |

### Ping Result (Blocked)

![Ping blocked - 100% packet loss](../screenshots/test-01/ping-result-blocked.png)

### Firewall Log — Default Deny Behavior

While reviewing the Live View logs, background UDP traffic was also observed being dropped by the interface's **default deny** rule (not the custom ICMP rule itself). This is included as supporting evidence that the firewall actively enforces a deny-by-default posture on unmatched traffic, in addition to the explicit ICMP rule above.

![Live log showing default deny matches](../screenshots/test-01/live-log-default-deny.png)

## Conclusion

The firewall correctly filters ICMP traffic based on the active rule. This confirmed that:

- OPNsense is correctly inline on the network path between the LAN client and external hosts
- Basic Block/Pass actions take effect as expected
- The lab environment is correctly set up for further, more granular testing

This test served as the foundation for more advanced protocol- and port-specific filtering (see [Test 2](test-02-tcp-port-filtering.md)).

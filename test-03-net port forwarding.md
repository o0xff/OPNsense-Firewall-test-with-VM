# Test 3: NAT / Port Forwarding (Destination NAT)

## Objective

Configure and validate **Destination NAT (Port Forwarding)** in OPNsense — a mechanism that allows an internal LAN service to be reached from outside the network through the firewall's WAN address.

## Concept

A firewall typically has a single public (WAN) IP address, while every device inside the LAN has its own private IP address. When an external request arrives at the WAN address, the firewall does not automatically know which internal device it belongs to.

Port Forwarding solves this by mapping a specific **port** on the WAN address to a specific **internal IP:port**, similar to how an office building with one street address relies on room numbers to route mail to the correct company inside.

| Concept | Real-world analogy |
|---|---|
| Firewall's WAN IP | Building's street address |
| Port | Room number |
| Internal server | Specific company inside the building |
| Destination NAT rule | Reception's routing instructions |

## Lab Setup

- **Internal service**: a simple PowerShell-based HTTP server (`System.Net.HttpListener`) running on port `8080` on the LAN client VM (`192.168.1.106`)
- **Firewall**: OPNsense, WAN address `10.0.2.15` (VirtualBox NAT network)
- **Goal**: reach the internal server via the firewall's WAN address instead of its private LAN address

## Configuration Steps

### 1. Test server (LAN client VM)

```powershell
$http = New-Object System.Net.HttpListener
$http.Prefixes.Add("http://+:8080/")
$http.Start()
while ($http.IsListening) {
    $ctx = $http.GetContext()
    $r = $ctx.Response
    $h = "<html><body><h1>Test server is working!</h1></body></html>"
    $b = [System.Text.Encoding]::UTF8.GetBytes($h)
    $r.ContentLength64 = $b.Length
    $r.OutputStream.Write($b, 0, $b.Length)
    $r.OutputStream.Close()
}
```

Verified locally:
```powershell
Invoke-WebRequest -Uri http://localhost:8080
# StatusCode: 200
```

### 2. Port alias — Firewall → Aliases → Add

| Field | Value |
|---|---|
| Enabled | ✅ |
| Name | `test_port_8080` |
| Type | Port(s) |
| Content | `8080` |
| Description | "Test web server port" |

Saved, then **Apply Changes**.

### 3. Destination NAT rule — Firewall → NAT → Destination NAT → Add

**Interface section**

| Field | Value |
|---|---|
| Interface | WAN |
| Version | IPv4 |
| Protocol | TCP |

**Destination section**

| Field | Value |
|---|---|
| Destination Address | WAN address |
| Destination Port | `test_port_8080` |

**Translation section**

| Field | Value |
|---|---|
| Redirect Target IP | `192.168.1.106` |
| Redirect Target Port | `test_port_8080` |

**Options**

| Field | Value |
|---|---|
| Log | ✅ enabled |
| Description | "Test - port forward to internal web server" |

Saved, then **Apply**.

### 4. NAT Reflection — Firewall → Settings → Advanced

Required specifically because the test was performed from a client located *inside* the same LAN the target service resides in (see [Issue 3](#issue-3-loopback-test-failed-nat-hairpinning) below).

| Field | Value |
|---|---|
| Reflection for destination NAT | ✅ enabled |
| Automatic outbound NAT for Reflection | ✅ enabled |

### 5. Windows Firewall rule (on the server VM)

```powershell
New-NetFirewallRule -DisplayName "Allow8080" -Direction Inbound -LocalPort 8080 -Protocol TCP -Action Allow
```

## Test Method

From the LAN client VM (same machine running the test server):

```powershell
Invoke-WebRequest -Uri http://10.0.2.15:8080
```

## Result

```
StatusCode        : 200
StatusDescription : OK
Server             : Microsoft-HTTPAPI/2.0
```

Confirmed end-to-end path:

```
Client → OPNsense WAN (10.0.2.15:8080)
       → Destination NAT rule matched (logged as "rdr")
       → Redirected to LAN server (192.168.1.106:8080)
       → Windows Firewall allowed inbound connection
       → Server responded
       → 200 OK
```

## Verification via Logs

Checked in **Firewall → Log Files → Live View**, filtered by `8080`:

```
Interface: LAN | Direction: In | Protocol: TCP
Source: 192.168.1.106 → Destination: 10.0.2.15:8080
Action: rdr | Label: rdr rule
```

The `rdr` (redirect) action confirmed the Destination NAT rule was actively matching and translating traffic — the key piece of evidence that the rule was functioning correctly, obtained before the end-to-end test succeeded.

## Issues Encountered and Resolved

### Issue 1: Port field rejected manual entry ("8080")

**Symptom**: typing `8080` into the Destination Port field returned "No results matched".

**Cause**: this field is not free text — it searches a list of predefined services and aliases only.

**Resolution**: created a port alias (`test_port_8080`) under Firewall → Aliases, which then appeared as a selectable option in the NAT form.

**Takeaway**: OPNsense relies heavily on aliases to reuse values (ports, IPs, ranges) across different rule types instead of typing raw values everywhere.

### Issue 2: Destination Port and Redirect Target IP fields mixed up

**Symptom**: an IP address was mistakenly entered into the Destination Port field.

**Cause**: form has many similarly-structured fields, easy to misplace values without pausing to check the underlying logic.

**Resolution**: re-mapped the fields to their actual meaning — Destination Port = "what's being requested from outside," Redirect Target IP = "where it's forwarded internally."

**Takeaway**: before saving a NAT rule, it helps to explicitly restate the "from outside → to inside" mapping rather than filling fields mechanically.

### Issue 3: Loopback test failed (NAT Hairpinning)

**Symptom**: `Invoke-WebRequest http://10.0.2.15:8080`, run from the same VM hosting the server, failed with "Unable to connect to the remote server."

**Cause**: the client and the target server are in the same LAN. The request targets the firewall's own external (WAN) address, expecting it to loop the connection back inside — a scenario known as **NAT Hairpinning / NAT Reflection**, which is disabled in OPNsense by default.

**Resolution**: enabled "Reflection for destination NAT" and "Automatic outbound NAT for Reflection" under Firewall → Settings → Advanced.

**Takeaway**: testing a port-forward from inside the same network it targets is not how it would normally be used in production (a real external client wouldn't hit this issue) — reflection is essentially a lab-testing accommodation.

### Issue 4: Still failing after enabling NAT Reflection

**Symptom**: connection still failed even with reflection enabled.

**Diagnosis**: checked Firewall → Log Files → Live View, filtered by `8080` — found the traffic was reaching the firewall and the NAT rule was matching (`Action: rdr`), proving the OPNsense configuration itself was correct and the failure was occurring further down the path.

**Cause**: the Windows Firewall on the server VM was blocking the inbound connection, since — due to the hairpin path — it appeared to arrive from outside rather than as a purely local connection.

**Resolution**: added an explicit inbound allow rule in Windows Firewall for TCP port 8080.

**Takeaway**: a packet's path can cross multiple independent firewalls (network-level and host-level). Confirming the network firewall is configured correctly doesn't guarantee the full path works — each hop needs to be checked individually, and logs are the fastest way to isolate which hop is failing.

## Conclusion

Successfully configured and validated a Destination NAT (Port Forwarding) rule in OPNsense, redirecting external WAN traffic on port 8080 to an internal LAN server. The test surfaced several real-world troubleshooting scenarios — alias-based field entry, NAT hairpinning, and multi-layer firewall filtering — all of which are common in production network administration.

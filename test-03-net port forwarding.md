# Test 3: NAT / Port Forwarding (Destination NAT)

## Objective

Configure and validate **Destination NAT (Port Forwarding)** in OPNsense — a mechanism that allows an internal LAN service to be reached from outside the network through the firewall's WAN address.


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

Alias Configuration:

<img width="897" height="317" alt="image" src="https://github.com/user-attachments/assets/c40aaf9f-6c86-46d0-8a40-58ff992596de" />


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

NAT Rule Configuration:

<img width="597" height="535" alt="image" src="https://github.com/user-attachments/assets/e2d67ac9-81e1-4600-a1e3-9fc97b12c3bc" />



### 4. NAT Reflection — Firewall → Settings → Advanced

Required specifically because the test was performed from a client located *inside* the same LAN the target service resides in (see [Issue 3](#issue-3-loopback-test-failed-nat-hairpinning) below).

| Field | Value |
|---|---|
| Reflection for destination NAT | ✅ enabled |
| Automatic outbound NAT for Reflection | ✅ enabled |

Firewall Settings:

<img width="416" height="170" alt="image" src="https://github.com/user-attachments/assets/2d4e9ec7-d0a3-42c9-81b9-13000fa25d1d" />


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

PowerShell Real-Time Proof:

<img width="921" height="392" alt="image" src="https://github.com/user-attachments/assets/f12b1a35-e953-4de6-a3e1-0c16696fb272" />




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


## Conclusion

Successfully configured and validated a Destination NAT (Port Forwarding) rule in OPNsense, redirecting external WAN traffic on port 8080 to an internal LAN server. The test surfaced several real-world troubleshooting scenarios — alias-based field entry, NAT hairpinning, and multi-layer firewall filtering — all of which are common in production network administration.

# Lab 06: DHCP & DNS Configuration

## Objective
Demonstrate how to configure a Cisco router as a DHCP server to automatically assign IP addresses to clients, and configure DNS so clients can resolve hostnames. Builds on Lab 05's routed topology — instead of manually assigning static IPs to PCs, they now receive addressing automatically.

## Topology

```
        192.168.1.0/30
   R1 -------------------- R2
   Gi0/0                Gi0/1  Gi0/0
   |                                  |
LAN: 192.168.10.0/24         LAN: 192.168.20.0/24
(DHCP Pool: .10-.100)        (DHCP Pool: .10-.100)
   |                                  |
  PC0                                PC1
(DHCP client)                  (DHCP client)
```

**Devices:**
| Device | Interface | IP Address        | Role                          |
|--------|-----------|---------------------|--------------------------------|
| R1     | Gi0/0     | 192.168.10.1/24     | Gateway + DHCP server, VLAN A |
| R1     | Gi0/1     | 192.168.1.1/30      | Link to R2                    |
| R2     | Gi0/0     | 192.168.1.2/30      | Link to R1                    |
| R2     | Gi0/1     | 192.168.20.1/24     | Gateway + DHCP server, VLAN B |
| PC0    | —         | DHCP (auto)         | Client on R1's LAN            |
| PC1    | —         | DHCP (auto)         | Client on R2's LAN            |

## Configuration

**R1 (DHCP pool for its own LAN):**
```
enable
configure terminal

ip dhcp excluded-address 192.168.10.1 192.168.10.9

ip dhcp pool LAN-R1
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 8.8.8.8
 lease 7
exit

end
write memory
```

**R2 (DHCP pool for its own LAN):**
```
enable
configure terminal

ip dhcp excluded-address 192.168.20.1 192.168.20.9

ip dhcp pool LAN-R2
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 8.8.8.8
 lease 7
exit

end
write memory
```

*(`excluded-address` reserves .1–.9 for static devices like router interfaces/servers so DHCP never hands them out. `dns-server` points clients to an upstream DNS resolver — in this lab, a simulated DNS server or 8.8.8.8 stand-in.)*

**PC0 / PC1 configuration:** set IP Configuration to **DHCP** in each PC's Desktop settings (instead of Static).

## Verification

```
R1# show ip dhcp binding

IP address       Client-ID/    Lease expiration    Type
                  Hardware address
192.168.10.10     0100.5079.6...   Jul 27 2026 09:14 AM   Automatic
```

```
PC0> ipconfig /all

IP Address......................: 192.168.10.10
Subnet Mask......................: 255.255.255.0
Default Gateway..................: 192.168.10.1
DNS Server........................: 8.8.8.8
```

PC0 received an address from the correct pool, excluding the reserved .1–.9 range, with the correct gateway and DNS server automatically applied.

**Ping test:** PC0 → PC1 succeeds using the DHCP-assigned addresses, confirming routing (from Lab 05) still works correctly alongside dynamic addressing.

## Troubleshooting Scenario

**Symptom:** PC1 on R2's LAN was not receiving an IP address — `ipconfig` showed `169.254.x.x` (APIPA), indicating a failed DHCP request.

**Diagnostic steps:**
```
R2# show ip dhcp pool
```
Output showed pool `LAN-R2` had 0 addresses leased and a `Current index` stuck at the pool's starting address — suggesting the request wasn't reaching the DHCP process at all.

```
R2# show run interface Gi0/1
```
Revealed the interface still had `shutdown` applied — it had never been brought up after the pool was configured, so PC1 had no Layer 1 connectivity to even reach R2.

**Root cause:** `no shutdown` was never issued on R2's Gi0/1 during initial interface setup, so despite the DHCP pool being correctly configured, PC1's DHCPDISCOVER broadcast never reached the router.

**Fix:**
```
R2(config)# interface GigabitEthernet0/1
R2(config-if)# no shutdown
```
Ran `ipconfig /release` then `ipconfig /renew` on PC1, confirmed via `show ip dhcp binding` on R2 that PC1 received 192.168.20.10.

## What I Learned
DHCP troubleshooting often isn't about the DHCP configuration itself — `show ip dhcp pool` showing zero leases pointed to a connectivity problem, not a pool misconfiguration. Checking basic interface status (`show ip interface brief` / `show run interface`) early saves time versus assuming the DHCP service itself is broken.

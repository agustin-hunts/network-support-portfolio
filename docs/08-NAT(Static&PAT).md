# Lab 08: NAT (Static & PAT)

## Objective
Demonstrate how Network Address Translation allows private internal IP addresses to communicate on a public network. Covers **Static NAT** (one-to-one mapping, typically for a server that needs a consistent public-facing address) and **PAT/NAT Overload** (many internal hosts sharing a single public IP — the standard method for general internet access).

## Topology

```
LAN: 192.168.10.0/24                 ISP Cloud
   |                                     |
  PC0 (.10.10)                    ISP Router
  PC1 (.10.11)                    (203.0.113.1)
Server0 (.10.50) ---- R1 (NAT router) ----+
                    Gi0/0 (inside)   Gi0/1 (outside)
                    192.168.10.1     203.0.113.10/30
```

**Devices:**
| Device   | Interface | IP Address           | Role                      |
|----------|-----------|-------------------------|----------------------------|
| R1       | Gi0/0     | 192.168.10.1/24         | Inside (LAN gateway)      |
| R1       | Gi0/1     | 203.0.113.10/30         | Outside (to ISP)          |
| PC0/PC1  | —         | 192.168.10.10 / .11     | Internal clients (PAT)    |
| Server0  | —         | 192.168.10.50            | Internal server (Static NAT → 203.0.113.50) |

**Scenario:**
- PC0 and PC1 share the router's outside interface address for general outbound traffic (**PAT**)
- Server0 gets a fixed, dedicated public IP (**Static NAT**) so external users can always reach it at the same address

## Configuration

**R1 — interface roles:**
```
enable
configure terminal

interface GigabitEthernet0/0
 ip address 192.168.10.1 255.255.255.0
 ip nat inside
 no shutdown
exit

interface GigabitEthernet0/1
 ip address 203.0.113.10 255.255.255.252
 ip nat outside
 no shutdown
exit
```

**Static NAT (Server0 — permanent one-to-one mapping):**
```
ip nat inside source static 192.168.10.50 203.0.113.50
```

**PAT / NAT Overload (PC0, PC1 — many-to-one using the router's outside interface):**
```
access-list 1 permit 192.168.10.0 0.0.0.255

ip nat inside source list 1 interface GigabitEthernet0/1 overload

end
write memory
```

*(The ACL defines which internal addresses are allowed to be translated; `overload` enables PAT so multiple hosts share the single outside IP, differentiated by port number.)*

## Verification

```
R1# show ip nat translations

Pro  Inside global        Inside local         Outside local     Outside global
---  203.0.113.50          192.168.10.50         ---               ---
tcp  203.0.113.10:1025     192.168.10.10:1025    203.0.113.1:80    203.0.113.1:80
tcp  203.0.113.10:1026     192.168.10.11:1026    203.0.113.1:80    203.0.113.1:80
```

- Server0's static entry is always present, even with no active traffic — permanent mapping
- PC0 and PC1 both translate through the same outside IP (203.0.113.10), distinguished only by port number — confirming PAT is working

```
R1# show ip nat statistics

Total active translations: 3 (1 static, 2 dynamic; 2 extended)
Outside interfaces: GigabitEthernet0/1
Inside interfaces: GigabitEthernet0/0
```

**Test results:**
- PC0 → simulated external web server — succeeds, translated via PAT
- External host → 203.0.113.50 (Server0's public IP) — succeeds, translated via static NAT to 192.168.10.50

## Troubleshooting Scenario

**Symptom:** PC0 and PC1 could not reach the external network at all — `show ip nat translations` showed no dynamic entries being created, only the static one for Server0.

**Diagnostic steps:**
```
R1# show ip nat statistics
```
Showed `Total active translations: 1 (1 static, 0 dynamic)` — confirming PAT was never triggering for PC0/PC1's traffic.

```
R1# show access-lists
```
Revealed ACL 1 was written as:
```
access-list 1 permit 192.168.1.0 0.0.0.255
```
— the wrong subnet (192.168.**1**.0 instead of 192.168.**10**.0), so it never matched traffic from the actual LAN.

**Root cause:** A typo in the NAT-associated ACL meant the access list never matched the real internal subnet, so `ip nat inside source list 1 ... overload` had nothing to translate.

**Fix:**
```
no access-list 1 permit 192.168.1.0 0.0.0.255
access-list 1 permit 192.168.10.0 0.0.0.255
```
Verified with `show access-lists` that the corrected subnet was in place, then confirmed via `show ip nat translations` that PC0 and PC1 traffic began generating dynamic PAT entries.

## What I Learned
NAT troubleshooting almost always comes down to checking three things in order: interface roles (`ip nat inside`/`outside`), the ACL defining what gets translated, and the NAT statement itself — in that order, since a working NAT config with a broken or mismatched ACL will fail silently with no obvious error, just an absence of translations in `show ip nat statistics`.

# Lab 01: VLANs & Port Assignment

## Objective
Demonstrate the creation of VLANs on a Cisco switch and the assignment of access ports to segment a network by department/function. VLANs reduce broadcast domains, improve security, and organize traffic logically rather than physically — a foundational skill for any network support role.

## Topology

```
                +-------------------+
                |     Switch1       |
                |   (Cisco 2960)    |
                +-------------------+
                 Fa0/1   Fa0/2   Fa0/3
                   |       |       |
                 PC0      PC1     PC2
               (Sales)   (IT)    (HR)
```

**Devices:**
| Device | Interface | VLAN | IP Address     |
|--------|-----------|------|-----------------|
| PC0    | Fa0/1     | 10 (Sales) | 192.168.10.10/24 |
| PC1    | Fa0/2     | 20 (IT)    | 192.168.20.10/24 |
| PC2    | Fa0/3     | 30 (HR)    | 192.168.30.10/24 |

## Configuration

**Switch1:**
```
enable
configure terminal

vlan 10
 name Sales
vlan 20
 name IT
vlan 30
 name HR
exit

interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
 no shutdown
exit

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 20
 no shutdown
exit

interface FastEthernet0/3
 switchport mode access
 switchport access vlan 30
 no shutdown
exit

```

## Verification

```
Switch1# show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Fa0/4, Fa0/5, Fa0/6, Fa0/7
10   Sales                            active    Fa0/1
20   IT                               active    Fa0/2
30   HR                               active    Fa0/3
```

```
Switch1# show interfaces status

Port      Name               Status       Vlan       Duplex  Speed Type
Fa0/1                        connected    10         a-full  a-100 10/100BaseTX
Fa0/2                        connected    20         a-full  a-100 10/100BaseTX
Fa0/3                        connected    30         a-full  a-100 10/100BaseTX
```

**Ping test:** PC0 (192.168.10.10) → PC1 (192.168.20.10) — fails as expected, confirming VLANs are isolated broadcast domains with no inter-VLAN routing configured yet (covered in Lab 02).

## Troubleshooting Scenario

**Symptom:** PC1 cannot communicate with any other device, including devices in the same VLAN 20.

**Diagnostic steps:**
```
Switch1# show interfaces Fa0/2 switchport
```
Output showed `Administrative Mode: static access` but `Access Mode VLAN: 1 (default)` — the port had not actually been moved into VLAN 20 despite the config appearing correct in `running-config`.

**Root cause:** The `switchport access vlan 20` command was entered before the VLAN itself was created, so the switch silently rejected the assignment and reverted the port to VLAN 1.

**Fix:** Recreated VLAN 20 first, then reapplied the port assignment:
```
vlan 20
 name IT
exit
interface Fa0/2
 switchport access vlan 20
exit
```
Confirmed with `show vlan brief` that Fa0/2 now appeared under VLAN 20.

## What I Learned
VLANs must exist before ports can be assigned to them — Packet Tracer (and real IOS) won't always throw an obvious error if the order is wrong, it just silently falls back to VLAN 1. Always verify with `show vlan brief` and `show interfaces switchport` rather than trusting `running-config` alone.

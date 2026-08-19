# Lab 04: EtherChannel & Link Redundancy

## Objective
Demonstrate how to bundle multiple physical links between two switches into a single logical EtherChannel using LACP, providing both increased bandwidth and redundancy without triggering STP to block redundant ports. Builds on Lab 03 — instead of letting STP block one of the two links between Switch1 and Switch2, both links are combined into one active logical link.

## Topology

```
                    +----------+                       +----------+
                    | Switch1  |==== Port-channel 1 ===| Switch2  |
                    +----------+   (Fa0/2 + Fa0/3)     +----------+
                     Fa0/1    Fa0/2   Fa0/3            Fa0/1  Fa0/2  Fa0/3
                       |         \_____/                  \____/       |
                     PC0        (bundled)                (bundled)    PC1
                   VLAN 10                                           VLAN 10
```

**Devices:**
| Device  | Interface       | Connects To         | Role                        |
|---------|------------------|----------------------|------------------------------|
| Switch1 | Fa0/1            | PC0                  | Access port, VLAN 10        |
| Switch1 | Fa0/2, Fa0/3     | Switch2 Fa0/2, Fa0/3 | Port-channel 1 members (LACP) |
| Switch2 | Fa0/1            | PC1                  | Access port, VLAN 10        |

Both physical links between Switch1 and Switch2 are bundled into **Port-channel 1**, so they act as a single logical trunk — no STP blocking needed, and traffic load-balances across both.

## Configuration

**Switch1:**
```
enable
configure terminal

interface range FastEthernet0/2 - 3
 switchport mode trunk
 switchport trunk allowed vlan 10
 channel-group 1 mode active
exit

interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10
exit

```

**Switch2:**
```
enable
configure terminal

interface range FastEthernet0/2 - 3
 switchport mode trunk
 switchport trunk allowed vlan 10
 channel-group 1 mode active
exit

interface Port-channel1
 switchport mode trunk
 switchport trunk allowed vlan 10
exit

```
*(`channel-group 1 mode active` on both sides enables LACP — both ends negotiate actively, as opposed to `passive` mode which only responds.)*

## Verification

```
Switch1# show etherchannel summary

Flags:  D - down        P - bundled in port-channel
        I - stand-alone s - suspended
        R - Layer3      S - Layer2

Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SU)         LACP      Fa0/2(P)  Fa0/3(P)
```

```
Switch1# show spanning-tree vlan 10

Interface           Role Sts Cost      Prio.Nbr Type
-------------------- ---- --- --------- -------- --------------
Po1                  Desg FWD 12        128.4    P2p
```

Only **one** STP entry appears — `Po1` — instead of two separate blocked/forwarding entries for Fa0/2 and Fa0/3. STP now treats the bundle as a single logical link, so both physical members forward traffic simultaneously.

**Ping test:** PC0 (VLAN 10) → PC1 (VLAN 10) — succeeds. Unplugging one of the two bundled links (e.g., Fa0/3) mid-ping test causes no packet loss, confirming redundancy.

## Troubleshooting Scenario

**Symptom:** After configuring `channel-group 1 mode active` on both switches, the Port-channel showed as suspended, and PC0 could not reach PC1.

**Diagnostic steps:**
```
Switch1# show etherchannel summary
```
Output showed:
```
Group  Port-channel  Protocol    Ports
------+-------------+-----------+-----------------------------------------------
1      Po1(SD)         LACP      Fa0/2(D)  Fa0/3(s)
```
The `(s)` flag on Fa0/3 indicated a suspended port — mismatched configuration between bundle members.

**Root cause:** Fa0/2 and Fa0/3 on Switch1 had different `switchport trunk allowed vlan` lists — Fa0/3 had accidentally been left with the default (all VLANs) instead of matching Fa0/2's restricted list. EtherChannel requires all bundled ports to share identical Layer 2 configuration (mode, trunk settings, allowed VLANs) or LACP suspends the mismatched port.

**Fix:**
```
interface FastEthernet0/3
 switchport trunk allowed vlan 10
```
Reapplied matching trunk config to Fa0/3, then verified with `show etherchannel summary` that both ports showed `(P)` bundled status and `Po1(SU)` up.

## What I Learned
EtherChannel member ports must be configured identically before bundling — even a small mismatch like a different allowed-VLAN list is enough for LACP to suspend a port rather than bundle it, and the switch won't always make this obvious without checking `show etherchannel summary` directly. This also reinforced why EtherChannel is often preferable to relying on STP alone for redundancy — both links stay active and contribute bandwidth instead of one sitting idle in a blocking state.

# Lab 03: STP Behavior & Loop Troubleshooting

## Objective
Demonstrate how Spanning Tree Protocol (STP) prevents Layer 2 loops when redundant links exist between switches, and how to diagnose a network outage caused by a loop when STP is misconfigured or disabled. Builds on Labs 01–02 by adding a second switch with a redundant link back to Switch1.

## Topology

```
                    +----------+           +----------+
                    | Switch1  |===========| Switch2  |
                    +----------+  (Link A) +----------+
                     Fa0/1  Fa0/2            Fa0/1  Fa0/2
                       |                       |      |
                     PC0                     PC1    (Link B - redundant,
                   VLAN 10                 VLAN 10    connects back to Switch1)
                                                          |
                                                       Switch1 Fa0/3
```

**Devices:**
| Device  | Interface | Connects To      | Role                  |
|---------|-----------|------------------|------------------------|
| Switch1 | Fa0/1     | PC0              | Access port, VLAN 10  |
| Switch1 | Fa0/2     | Switch2 Fa0/1    | Trunk (Link A)         |
| Switch1 | Fa0/3     | Switch2 Fa0/2    | Trunk (Link B — redundant) |
| Switch2 | Fa0/3     | PC1              | Access port, VLAN 10  |

Two links between Switch1 and Switch2 create an intentional Layer 2 loop — STP is required to block one of them.

## Configuration

**Switch1 & Switch2 (trunk links between switches):**
```
enable
configure terminal

interface range FastEthernet0/2 - 3
 switchport mode trunk
 switchport trunk allowed vlan 10
 no shutdown
exit

spanning-tree mode pvst
spanning-tree vlan 10 priority 4096

end
write memory
```
*(Priority 4096 set on Switch1 to force it as the root bridge for VLAN 10 — lower priority wins root election.)*

## Verification

```
Switch1# show spanning-tree vlan 10

VLAN0010
  Spanning tree enabled protocol ieee
  Root ID    Priority    4096
             Address     0001.42AB.1200
             This bridge is the root

  Bridge ID  Priority    4096
             Address     0001.42AB.1200

Interface           Role Sts Cost      Prio.Nbr Type
-------------------- ---- --- --------- -------- --------------
Fa0/2                Desg FWD 19        128.2    P2p
Fa0/3                Desg FWD 19        128.3    P2p
```

```
Switch2# show spanning-tree vlan 10

Interface           Role Sts Cost      Prio.Nbr Type
-------------------- ---- --- --------- -------- --------------
Fa0/1                Root FWD 19        128.1    P2p
Fa0/2                Altn BLK 19        128.2    P2p
```

Fa0/2 on Switch2 shows **BLK (blocking)** — confirming STP successfully identified the redundant path and blocked it to prevent a loop, while Fa0/1 forwards as the root port.

## Troubleshooting Scenario

**Symptom:** After connecting the second (redundant) link between Switch1 and Switch2, the network became unusable — PC0 and PC1 lost connectivity entirely, and both switches' CPU/activity LEDs showed constant heavy traffic.

**Diagnostic steps:**
```
Switch1# show spanning-tree vlan 10
```
Output showed `Spanning tree enabled protocol ieee` was missing — spanning-tree had been globally disabled on Switch1 during initial setup (`no spanning-tree vlan 10` had been run by mistake while testing).

Also checked:
```
Switch1# show interfaces Fa0/2 counters
Switch1# show interfaces Fa0/3 counters
```
Both interfaces showed rapidly climbing broadcast/multicast packet counts — consistent with a broadcast storm caused by the loop, since no port was being blocked.

**Root cause:** STP was disabled on VLAN 10 on Switch1, so when the second redundant link (Fa0/3) was added, both links stayed in forwarding state simultaneously — creating a Layer 2 loop and broadcast storm.

**Fix:**
```
Switch1(config)# spanning-tree vlan 10
```
Re-enabled STP for VLAN 10, confirmed root bridge election completed, and verified Fa0/3 on Switch2 transitioned to `BLK` state via `show spanning-tree vlan 10`. Connectivity restored immediately.

## What I Learned
STP isn't just "on by default and forget about it" — a single disabled VLAN instance or a misconfigured priority can silently remove loop protection while both redundant links stay active, causing a broadcast storm that looks like a total outage. `show spanning-tree vlan X` and interface packet counters were the fastest way to confirm STP wasn't running, rather than assuming the physical links themselves were the problem.

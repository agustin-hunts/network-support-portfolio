# Lab 05: OSPF Single-Area Routing

## Objective
Demonstrate dynamic routing between multiple routers using OSPF (Open Shortest Path First) in a single area (Area 0). Unlike static routes, OSPF automatically discovers neighbors, exchanges routes, and recalculates paths if a link fails — core knowledge for any support role troubleshooting enterprise routing.

## Topology

```
        192.168.1.0/30                192.168.2.0/30
   R1 ----------------------- R2 ---------------------- R3
   Gi0/1                 Gi0/0  Gi0/1                Gi0/1
   |                                                  |
LAN: 192.168.10.0/24                          LAN: 192.168.30.0/24
   |                                                  |
  PC0                                                PC1
```

**Devices:**
| Device | Interface | IP Address         | Network            |
|--------|-----------|----------------------|----------------------|
| R1     | Gi0/0     | 192.168.1.1/30       | Link to R2          |
| R1     | Gi0/1     | 192.168.10.1/24      | LAN (PC0)           |
| R2     | Gi0/0     | 192.168.1.2/30       | Link to R1          |
| R2     | Gi0/1     | 192.168.2.1/30       | Link to R3          |
| R3     | Gi0/0     | 192.168.2.2/30       | Link to R2          |
| R3     | Gi0/1     | 192.168.30.1/24      | LAN (PC1)           |

All routers and links belong to **OSPF Area 0** (single area — no inter-area complexity yet).

## Configuration

**R1:**
```
enable
configure terminal

interface GigabitEthernet0/0
 ip address 192.168.1.1 255.255.255.252
 no shutdown
exit

interface GigabitEthernet0/1.10
 ip address 192.168.10.1 255.255.255.0
 no shutdown
exit

router ospf 1
 network 192.168.10.0 0.0.0.255 area 0
 network 192.168.1.0 0.0.0.3 area 0
exit

end
write memory
```

**R2:**
```
enable
configure terminal

interface GigabitEthernet0/0
 ip address 192.168.1.2 255.255.255.252
 no shutdown
exit

interface GigabitEthernet0/1
 ip address 192.168.2.1 255.255.255.252
 no shutdown
exit

router ospf 1
 network 192.168.1.0 0.0.0.3 area 0
 network 192.168.2.0 0.0.0.3 area 0
exit

end
write memory
```

**R3:**
```
enable
configure terminal

interface GigabitEthernet0/0
 ip address 192.168.2.2 255.255.255.252
 no shutdown
exit

interface GigabitEthernet0/1
 ip address 192.168.30.1 255.255.255.0
 no shutdown
exit

router ospf 1
 network 192.168.2.0 0.0.0.3 area 0
 network 192.168.30.0 0.0.0.255 area 0
exit

end
write memory
```

## Verification

```
R2# show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
192.168.10.1    1     FULL/DR         00:00:39    192.168.1.1     GigabitEthernet0/0
192.168.30.1    1     FULL/DR         00:00:32    192.168.2.2     GigabitEthernet0/1
```

```
R1# show ip route ospf

O    192.168.2.0/30 [110/2] via 192.168.1.2, 00:12:04, GigabitEthernet0/1
O    192.168.30.0/24 [110/3] via 192.168.1.2, 00:12:04, GigabitEthernet0/1
```

R1 has learned the route to R3's LAN (192.168.30.0/24) entirely through OSPF — no static routes configured.

**Ping test:** PC0 (192.168.10.x) → PC1 (192.168.30.x) — succeeds, traffic routed automatically across R1 → R2 → R3.

## Troubleshooting Scenario

**Symptom:** R1 and R2 show as OSPF neighbors, but R3's LAN (192.168.30.0/24) never appears in R1's routing table.

**Diagnostic steps:**
```
R2# show ip ospf neighbor
```
Only showed R1 as a neighbor — R3 was missing entirely, meaning the adjacency between R2 and R3 never formed.

```
R3# show ip protocols
```
Confirmed OSPF process was running, but the network statement for the R2–R3 link was missing — checked `running-config` and found:
```
router ospf 1
 network 192.168.30.0 0.0.0.255 area 0
```
Only the LAN network had been added; the point-to-point link to R2 (192.168.2.0/30) was never included.

**Root cause:** Incomplete `network` statement under `router ospf 1` on R3 — without advertising the 192.168.2.0/30 link into OSPF, R3 never sent Hello packets out Gi0/0, so no adjacency formed with R2.

**Fix:**
```
R3(config)# router ospf 1
R3(config-router)# network 192.168.2.0 0.0.0.3 area 0
```
Verified with `show ip ospf neighbor` on R2 that R3 appeared as `FULL`, then confirmed R1 received the 192.168.30.0/24 route via `show ip route ospf`.

## What I Learned
OSPF neighbor adjacencies depend entirely on correct `network` statements on **every** interface that should participate — missing even one link (even if the router's LAN is advertised fine) silently breaks the topology with no error message. `show ip ospf neighbor` was the fastest way to spot exactly which adjacency was missing before digging into `running-config`.

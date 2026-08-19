# Lab 02: Trunking & Inter-VLAN Routing

## Objective
Demonstrate how to carry multiple VLANs across a single switch-to-router (or switch-to-switch) link using 802.1Q trunking, and how to route traffic between VLANs using router-on-a-stick. Builds directly on Lab 01 — VLANs that were previously isolated can now communicate through a Layer 3 device.

## Topology

```
                 Router0 (Router-on-a-Stick)
                      Gi0/0
                        |
                        | (Trunk - 802.1Q)
                        |
                    +----------+
                    | Switch1  |
                    +----------+
                 Fa0/1   Fa0/2   Fa0/3
                   |       |       |
                 PC0     PC1     PC2
              VLAN 10   VLAN 20  VLAN 30
              (Sales)   (IT)     (HR)
```

**Devices:**
| Device   | Interface       | VLAN | IP Address        |
|----------|-----------------|------|--------------------|
| PC0      | Fa0/1           | 10   | 192.168.10.10/24   |
| PC1      | Fa0/2           | 20   | 192.168.20.10/24   |
| PC2      | Fa0/3           | 30   | 192.168.30.10/24   |
| Router0  | Gi0/0.10        | 10   | 192.168.10.1/24 (gateway) |
| Router0  | Gi0/0.20        | 20   | 192.168.20.1/24 (gateway) |
| Router0  | Gi0/0.30        | 30   | 192.168.30.1/24 (gateway) |
| Switch1  | Gi0/1 (to Router)| trunk | — |

## Configuration

**Switch1 (trunk port to router):**
```
enable
configure terminal

interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
 no shutdown
exit

end
write memory
```

**Router0 (router-on-a-stick — subinterfaces):**
```
enable
configure terminal

interface GigabitEthernet0/0
 no shutdown
exit

interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
exit

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
exit

interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
exit

end
write memory
```

**PC default gateways:** set PC0/PC1/PC2 gateway to their respective VLAN's router subinterface IP (.10.1, .20.1, .30.1).

## Verification

```
Switch1# show interfaces trunk

Port        Mode         Encapsulation  Status        Native vlan
Gi0/1       on           802.1q         trunking      1

Port        Vlans allowed on trunk
Gi0/1       10,20,30
```

```
Router0# show ip interface brief

Interface                  IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0         unassigned      YES manual up                    up
GigabitEthernet0/0.10      192.168.10.1    YES manual up                    up
GigabitEthernet0/0.20      192.168.20.1    YES manual up                    up
GigabitEthernet0/0.30      192.168.30.1    YES manual up                    up
```

**Ping test:** PC0 (192.168.10.10) → PC1 (192.168.20.10) — now succeeds, confirming inter-VLAN routing is working through Router0.

## Troubleshooting Scenario

**Symptom:** PC0 can ping its own gateway (192.168.10.1) but cannot reach PC1 in VLAN 20.

**Diagnostic steps:**
```
Router0# show ip interface brief
```
`GigabitEthernet0/0.20` showed status `up` but `Method` blank and no IP listed — the subinterface had been created but the `ip address` command was never applied.

**Root cause:** Typo in the encapsulation command (`encapsulation dot1q 2` instead of `20`) caused the subinterface to bind to the wrong VLAN tag, so the IP address command that followed was applied to a mismatched subinterface context.

**Fix:** Removed and recreated the subinterface with the correct VLAN tag:
```
no interface GigabitEthernet0/0.20
interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
```
Verified with `show ip interface brief` and a successful ping from PC0 to PC1.

## What I Learned
Router-on-a-stick relies entirely on matching VLAN tags between the switch trunk and the router's subinterfaces — a single typo in the `encapsulation dot1Q` number silently breaks routing for that VLAN with no error message. Always cross-check `show interfaces trunk` on the switch against `show ip interface brief` on the router to confirm both sides agree on VLAN numbering.

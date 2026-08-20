# Lab 10: Port Security & Hardening

## Objective
Demonstrate how to secure switch access ports against unauthorized devices using port security, and apply basic device hardening — SSH remote access instead of insecure Telnet, and restricting console/VTY access. These are baseline security practices expected on any production switch or router a support engineer would touch.

## Topology

```
                    +----------+
                    | Switch1  |
                    +----------+
                     Fa0/1    Fa0/2
                       |         |
                     PC0     Rogue-PC
                  (authorized)  (unauthorized device
                                  plugged into Fa0/2)
```

**Devices:**
| Device   | Interface | Role                                      |
|----------|-----------|--------------------------------------------|
| PC0      | Fa0/1     | Authorized device — MAC locked to this port |
| Rogue-PC | Fa0/2     | Simulated unauthorized device to trigger violation |
| Switch1  | VLAN 1 (mgmt) | Management IP for SSH access          |

## Configuration

### Port Security (Fa0/1 — lock to PC0's MAC address)

```
enable
configure terminal

interface FastEthernet0/1
 switchport mode access
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation shutdown
exit

end
write memory
```

*(`sticky` dynamically learns PC0's MAC address the first time it connects and locks it permanently into the running config. `violation shutdown` puts the port into err-disabled state if any other MAC address tries to connect — the strictest response, versus `restrict` which just drops traffic and logs it, or `protect` which drops silently.)*

### Device Hardening (SSH access instead of Telnet)

```
enable
configure terminal

hostname Switch1
ip domain-name supportlab.local

crypto key generate rsa
   ! (prompted for modulus size — use 1024 or higher)

username admin privilege 15 secret Cisco123!

line vty 0 15
 transport input ssh
 login local
exit

line console 0
 password Cisco123!
 login
exit

service password-encryption

end
write memory
```

*(`transport input ssh` disables Telnet on the VTY lines entirely — only SSH connections are accepted. `service password-encryption` ensures passwords aren't stored in plaintext in the config.)*

## Verification

```
Switch1# show port-security interface FastEthernet0/1

Port Security              : Enabled
Port Status                : Secure-up
Violation Mode              : Shutdown
Maximum MAC Addresses       : 1
Total MAC Addresses         : 1
Sticky MAC Addresses        : 1
Last Source Address:Vlan    : 0050.79AB.12CD:1
Security Violation Count    : 0
```

```
Switch1# show ip ssh

SSH Enabled - version 2.0
Authentication timeout: 120 secs; Authentication retries: 3
```

**Test — plug Rogue-PC into Fa0/1 (PC0's locked port) instead of Fa0/2:**
```
Switch1# show port-security interface FastEthernet0/1

Port Status                : Secure-shutdown
Security Violation Count    : 1
```
Port automatically transitions to `err-disabled`, confirming port security correctly blocked the unauthorized MAC address.

**SSH test:** connecting via PC0's terminal using `ssh -l admin 192.168.10.1` succeeds and prompts for the configured password; a Telnet attempt (`telnet 192.168.10.1`) is refused, confirming Telnet is disabled.

## Troubleshooting Scenario

**Symptom:** PC0 lost network connectivity after being unplugged and reconnected to a different port (Fa0/1 moved to Fa0/3 during a desk move) — the port showed `err-disabled`.

**Diagnostic steps:**
```
Switch1# show interfaces FastEthernet0/3 status
```
Showed `err-disabled` even though only the legitimate PC0 was connected.

```
Switch1# show port-security interface FastEthernet0/3
```
Confirmed a violation had occurred — but the "violating" MAC address was actually PC0's own address. Since port security with `sticky` had originally locked PC0's MAC to Fa0/1 specifically, plugging the same device into a different port (Fa0/3, which had its own separate sticky lock from a prior device) triggered a violation.

**Root cause:** Sticky MAC addresses are bound to the specific port they were learned on, not the switch as a whole — moving an authorized device to a different secured port is treated as a new, unrecognized device on that port.

**Fix:**
```
interface FastEthernet0/3
 shutdown
 no switchport port-security mac-address sticky 0050.79AB.99EF
 no shutdown
 switchport port-security mac-address sticky
```
Cleared the old sticky entry, re-enabled the port, and allowed it to relearn PC0's MAC address on its new port. Confirmed with `show port-security interface Fa0/3` that the port returned to `Secure-up`.

## What I Learned
Port security with sticky MACs is powerful but inflexible by design — it doesn't account for legitimate device moves, which is expected behavior, not a bug. In a real support environment, this means any planned device relocation needs a corresponding port security update, or it will generate a very predictable but easily misread "security incident" that's really just a desk move.

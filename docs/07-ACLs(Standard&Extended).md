# Lab 07: ACLs (Standard & Extended)

## Objective
Demonstrate how to control traffic flow between networks using Access Control Lists (ACLs) — both standard (source-IP-based) and extended (source/destination/protocol/port-based). ACLs are one of the most common tools used in support/security troubleshooting to permit, deny, and filter traffic.

## Topology

```
                        192.168.1.0/30
             R1 -------------------------------- R2
            Gi0/1         Gi0/0---Gi0/0        Gi0/1
             |                                   |
    LAN A: 192.168.10.0/24           LAN B: 192.168.20.0/24
             |                                   |
        PC0 (.10.10)                      PC1 (.20.10)  
                                   Server0 (.20.50) - Web/FTP
```

**Scenario:**
- **Standard ACL:** Block PC0 (192.168.10.10) from reaching anything on LAN B entirely
- **Extended ACL:** Instead of blocking all traffic, allow PC0 to reach Server0's web service (HTTP/port 80) but deny FTP (port 21)

## Configuration

### Standard ACL (block PC0 from LAN B entirely)

**R2 (applied closest to destination, per best practice for standard ACLs):**
```
enable
configure terminal

access-list 10 deny host 192.168.10.10
access-list 10 permit any

interface GigabitEthernet0/1
 ip access-group 10 out
exit

```

*(Standard ACLs only filter by source IP, so they should be placed as close to the destination as possible — otherwise they'd block PC0 from reaching networks it should still be allowed to access.)*

### Extended ACL (allow HTTP, deny FTP, from PC0 to Server0)

**R1 (applied closest to source, per best practice for extended ACLs):**
```
enable
configure terminal

access-list 110 deny tcp host 192.168.10.10 host 192.168.20.50 eq 21
access-list 110 permit tcp host 192.168.10.10 host 192.168.20.50 eq 80
access-list 110 permit ip any any

interface GigabitEthernet0/0
 ip access-group 110 in
exit

```

*(Extended ACLs can match source, destination, protocol, and port — so they should sit close to the source to avoid wasting bandwidth carrying traffic across the network only to drop it later.)*

## Verification

```
R2# show access-lists

Standard IP access list 10
    10 deny   host 192.168.10.10
    20 permit any
```

```
R1# show access-lists

Extended IP access list 110
    10 deny tcp host 192.168.10.10 host 192.168.20.50 eq 21
    20 permit tcp host 192.168.10.10 host 192.168.20.50 eq 80
    30 permit ip any any
```

**Test results:**
- PC0 → Server0 web browser (http ://192.168.20.50) — **succeeds** (port 80 explicitly permitted)
- PC0 → Server0 FTP client — **fails / times out** (port 21 explicitly denied)
- PC0 → PC1 ping (192.168.20.10) — **fails** (blocked by standard ACL 10 on R2, since PC1 isn't the web server and standard ACL blocks PC0 entirely from LAN B in this test variant)

## Troubleshooting Scenario

**Symptom:** After applying ACL 110 on R1, PC0 could no longer reach Server0 at all — even HTTP stopped working, despite the permit statement.

**Diagnostic steps:**
```
R1# show access-lists
```
Output showed:
```
Extended IP access list 110
    10 permit ip any any
    20 deny tcp host 192.168.10.10 host 192.168.20.50 eq 21
    30 permit tcp host 192.168.10.10 host 192.168.20.50 eq 80
```
The **implicit ordering was wrong** — a blanket `permit ip any any` had been entered first, so every packet matched that line immediately and the more specific deny/permit statements below it were never evaluated.

**Root cause:** ACL statements are processed top-down, and processing stops at the first match. The broad permit-any statement was accidentally configured before the specific FTP-deny and HTTP-permit rules, making those rules unreachable.

**Fix:** Removed and rebuilt the ACL in the correct order (most specific rules first, broad permit/deny last):
```
no access-list 110
access-list 110 deny tcp host 192.168.10.10 host 192.168.20.50 eq 21
access-list 110 permit tcp host 192.168.10.10 host 192.168.20.50 eq 80
access-list 110 permit ip any any
```
Reapplied to Gi0/0 and confirmed with `show access-lists` that order was correct; retested and both the FTP-block and HTTP-allow behaved as expected.

## What I Learned
ACL rule **order** matters as much as the rules themselves — since processing stops at the first match top-down, a broad statement placed too early silently makes every rule after it unreachable, with no warning from the router. Always verify the actual sequence with `show access-lists` after configuring, not just the intended logic.

# Lab 09: Basic WLAN Setup

## Objective
Demonstrate how to configure a wireless network using a Cisco Wireless Access Point (or wireless router) — creating an SSID, securing it with WPA2, and connecting wireless clients. Ties into CWNA study alongside the CCNA-focused labs in this series.

## Topology

```
                    +----------+
                    | Switch1  |
                    +----------+
                     Fa0/1   Fa0/2
                       |        |
                   Access     PC0
                   Point    (wired, 192.168.10.10)
              (192.168.10.2)
                       |
                (( WiFi - SSID: "SupportLab" ))
                       |
                  +---------+     +---------+
                  | Laptop0 |     | Phone0  |
                  +---------+     +---------+
                (wireless clients, DHCP)
```

**Devices:**
| Device       | Interface | IP Address       | Role                          |
|--------------|-----------|--------------------|--------------------------------|
| Access Point | Fa0/1     | 192.168.10.2/24    | Bridges wired LAN to wireless |
| PC0          | Fa0/2     | 192.168.10.10/24   | Wired client (reference/baseline) |
| Laptop0      | Wireless  | DHCP (auto)        | Wireless client                |
| Phone0       | Wireless  | DHCP (auto)        | Wireless client                |

**SSID:** `SupportLab`
**Security:** WPA2-PSK
**Passphrase:** (set during config — treat as a placeholder, never document real passphrases in a public repo)

## Configuration

### Access Point (via GUI in Packet Tracer — Config tab)

**Step 1 — Basic wireless settings (Config → Port 1 / Interface):**
```
SSID: SupportLab
Channel: 6 (or Auto)
```

**Step 2 — Security (Config → Port 1 → Security):**
```
Authentication: WPA2-PSK
Encryption: AES
Pass Phrase: [set a strong passphrase]
```

*(In real Cisco WLC/IOS environments, this maps to CLI commands like `dot11 ssid SupportLab`, `authentication open`, `wpa-psk ascii <passphrase>` — but in Packet Tracer's simplified AP model, this is done through the AP's GUI Config tab.)*

### Wireless Clients (Laptop0, Phone0)

- Open **Desktop → PC Wireless** (or the device's wireless config)
- Under **Connect**, select SSID `SupportLab`
- Enter the WPA2 passphrase
- Set **IP Configuration** to DHCP (assuming a DHCP pool exists upstream, per Lab 06) or static within 192.168.10.0/24 if no DHCP server is present

## Verification

**On Laptop0 (Desktop → Command Prompt):**
```
Laptop0> ipconfig

IP Address......................: 192.168.10.15
Subnet Mask......................: 255.255.255.0
Default Gateway..................: 192.168.10.1
```

**Wireless connection status icon** on Laptop0 and Phone0 shows a full signal bar (green), confirming successful association with the AP.

**Ping test:**
```
Laptop0> ping 192.168.10.10

Reply from 192.168.10.10: bytes=32 time=2ms TTL=128
Reply from 192.168.10.10: bytes=32 time=1ms TTL=128
```
Laptop0 (wireless) successfully reaches PC0 (wired) through the AP → Switch1 bridge, confirming the wireless and wired segments are on the same functioning LAN.

## Troubleshooting Scenario

**Symptom:** Phone0 could see the `SupportLab` SSID and entered the correct passphrase, but the connection kept failing/timing out instead of associating.

**Diagnostic steps:**
Checked the AP's **Config → Port 1 → Security** settings and compared against Phone0's wireless config:
- AP: `Authentication: WPA2-PSK`, `Encryption: AES`
- Phone0: `Authentication: WPA2-PSK`, `Encryption: TKIP`

The encryption type mismatch was the issue — Phone0 was still set to the older TKIP standard from a previous lab/default profile, while the AP was configured for AES only.

**Root cause:** WPA2 clients and access points must match not just the authentication method (PSK) but also the specific encryption cipher (AES vs TKIP). A mismatch causes the 4-way handshake to fail silently — the device usually just shows "unable to connect" with no specific error.

**Fix:** Updated Phone0's wireless profile to use **AES** encryption to match the AP, reconnected using the same SSID and passphrase, and confirmed successful association via the signal icon and a successful `ping` to PC0.

## What I Learned
Wireless troubleshooting often comes down to small mismatches between client and AP settings that don't produce a clear error message — encryption type (AES vs TKIP) being one of the most common. When a client can see the SSID but won't connect despite a correct passphrase, checking the encryption/cipher setting on both ends is one of the first things worth verifying before assuming a passphrase typo.

# Home Wi-Fi Security Lab — WPA2 & Open Network Traffic Analysis

## Overview

Practical security testing performed against my own home network and personal
devices, in an isolated home-lab environment. Goal: understand WPA2 handshake
capture/cracking, evaluate password strength via dictionary attack, and analyze
what network-level attackers can and cannot observe on an open (unencrypted)
Wi-Fi network in a world where most application traffic is TLS-encrypted.

**Scope:** own router, own phone. No third-party networks or devices were
targeted at any point.

## Environment

| Component | Details |
|---|---|
| OS | Kali Linux (VMware VM) |
| Wi-Fi adapter | Ralink RT5572 (USB, monitor mode + injection capable) |
| Driver | rt2800usb |
| Tools | aircrack-ng suite, wifite, hostapd, dnsmasq, iptables, Wireshark, tcpdump, nmap |

## Part 1 — WPA2 Handshake Capture & Dictionary Attack

### Objective
Assess whether the home network's WPA2-PSK password is resistant to an
offline dictionary attack.

### Methodology
1. **Reconnaissance** — identified target BSSID, channel, and connected
   client MAC via `airodump-ng`.
2. **Capture setup** — locked the adapter to the target channel and began
   writing all traffic to a `.cap` file:
   ```bash
   sudo airodump-ng -c <channel> --bssid <BSSID> -w capture wlan0mon
   ```
3. **Handshake forcing** — sent targeted deauthentication frames to the
   connected client to force a reassociation and capture the WPA 4-way
   handshake:
   ```bash
   sudo aireplay-ng -0 5 -a <BSSID> -c <client MAC> wlan0mon
   ```
4. **Verification** — confirmed a valid handshake was captured:
   ```bash
   aircrack-ng capture-01.cap
   # → 1 handshake
   ```
5. **Offline dictionary attack** — tested the handshake against the
   `rockyou.txt` wordlist (~14M entries):
   ```bash
   aircrack-ng capture-01.cap -w /usr/share/wordlists/rockyou.txt
   ```
6. **Automated cross-check** — repeated the full flow using `wifite` to
   validate the manual result against an automated tool.

### Findings
- WPA 4-way handshake successfully captured.
- Dictionary attack against `rockyou.txt` did **not** recover the password.
- **Conclusion:** the network's WPA2-PSK password is not present in common
  breach/wordlist compilations and resists a standard offline dictionary
  attack.

### Defensive takeaway
WPA2's security against offline attack depends entirely on password
strength, since the handshake itself (not the password) is what's
transmitted over the air. A password absent from common wordlists is the
single biggest defense against this attack class.

---

## Part 2 — Open Network Setup & Traffic Metadata Analysis

### Objective
Demonstrate what is (and isn't) visible to someone monitoring traffic on an
open, unencrypted Wi-Fi network, using my own phone as the test client.

### Methodology
1. **Access point setup** — configured an open (unauthenticated) test SSID
   using `hostapd`, with DHCP served via `dnsmasq` and NAT/forwarding via
   `iptables` to provide internet access to connected clients.
2. **Client connection** — connected my own phone to the test network.
3. **Traffic capture:**
   ```bash
   sudo tcpdump -i any -w phone_traffic.pcap
   ```
4. **Analysis in Wireshark:**
   - `dns` — enumerate all domains resolved by the device
   - `tls.handshake.type == 1` — extract SNI (destination domain) from every
     TLS Client Hello, without decrypting the session
   - `http` — check for any legacy unencrypted HTTP traffic
   - `Statistics → Protocol Hierarchy` / `Statistics → Conversations` —
     traffic volume/destination breakdown

### Findings
- **Visible without decryption:** DNS queries, TLS SNI (destination
  hostnames), traffic volume, and timing for every connection — i.e. *who
  the device talks to and when*, even though the network itself was
  unencrypted at the Wi-Fi layer.
- **Not visible:** any actual content — page contents, credentials, iCloud
  sync data, app data — because virtually all modern traffic (iCloud,
  App Store, browser HTTPS) is TLS-encrypted end-to-end, independent of
  whether the underlying Wi-Fi is open or WPA2.
- No plaintext HTTP traffic was observed during testing.

### Defensive takeaway
An open Wi-Fi network exposes **metadata** (destination domains, traffic
timing/volume) to anyone in range, but not **content**, as long as TLS is
in use — which is now the default for nearly all applications and
websites. This is why TLS/HTTPS adoption has made classic open-Wi-Fi
sniffing largely ineffective for credential theft, while metadata exposure
(browsing patterns, app usage) remains a real privacy consideration.
Practical mitigation for end users on public/open Wi-Fi: a VPN, which adds
an additional encrypted layer that hides even the metadata from the local
network.

---

## Part 3 — Cleartext HTTP Login Form Demonstration (badssl.com)

### Objective
Demonstrate, in a controlled way, how an HTTP (non-TLS) login form appears
to a passive observer on the same network.

### Methodology
Browsed to `http-login.badssl.com` — a public test domain maintained by the
badssl.com project specifically for demonstrating HTTP vs HTTPS behavior —
and submitted its demo login form while capturing traffic with `tcpdump`,
then inspected the session in Wireshark via:
```
ip.addr == <phone IP> && http
```
followed by right-click → **Follow → HTTP Stream** on the `POST /submit/`
request.

### Findings
- The entire HTTP request/response — including headers, form action, and
  page structure — was visible in cleartext, as expected for an
  unencrypted HTTP session.
- In this specific capture, the POST request body was empty
  (`Content-Length: 0`), so the literal form field values were not present
  in this particular request. The structural exposure (method, endpoint,
  content-type, and the fact that a login form exists and submits over
  plaintext HTTP) was fully visible regardless.
- Had the form transmitted actual field data (as it would on a real HTTP
  login page), those values would have been equally readable in plaintext
  to anyone monitoring the network — this is exactly the vulnerability
  class the badssl.com test domain exists to illustrate.

### Defensive takeaway
Any login form served over HTTP (no TLS) exposes its request structure,
and — when the form actually carries data — the submitted values
themselves, to passive observers on the same network segment. This is a
textbook example of why HTTPS is non-negotiable for anything handling
credentials, and why browsers now flag HTTP login forms as "Not Secure."

---

## Tools reference

| Tool | Purpose |
|---|---|
| `airodump-ng` | Wireless recon / packet capture |
| `aireplay-ng` | Deauthentication (handshake forcing) |
| `aircrack-ng` | Handshake verification / dictionary attack |
| `wifite` | Automated WPA2 attack workflow |
| `hostapd` | Software access point |
| `dnsmasq` | DHCP/DNS for the test AP |
| `iptables` | NAT/forwarding for the test AP |
| `Wireshark` / `tcpdump` | Packet capture and protocol analysis |

## Ethical note

All testing was performed exclusively against infrastructure and devices I
own, in a network isolated from third parties. No third-party networks,
devices, or data were accessed at any point.

# Home Wi-Fi Security Lab — WPA2, Network Traffic & DPI Evasion Analysis

## Overview
Practical security testing performed against my own home network and personal devices, in an isolated home-lab environment. Goal: understand WPA2 handshake capture/cracking, evaluate password strength via dictionary attack, analyze what network-level attackers can and cannot observe on an open (unencrypted) Wi-Fi network, and investigate DPI evasion techniques in VPN traffic.

> **Scope Note:** Own router, own phone. No third-party networks or devices were targeted at any point.

---

## Environment

| Component | Details |
|---|---|
| **OS** | Kali Linux (VMware VM) |
| **Wi-Fi adapter** | Ralink RT5572 (USB, monitor mode + injection capable) |
| **Driver** | rt2800usb |
| **Tools** | aircrack-ng suite, wifite, hostapd, dnsmasq, iptables, Wireshark, tcpdump, bettercap, nmap |

---

## Part 1 — WPA2 Handshake Capture & Dictionary Attack

### Objective
Assess whether the home network's WPA2-PSK password is resistant to an offline dictionary attack.

### Methodology
1. **Reconnaissance:** Identified target BSSID, channel, and connected client MAC via `airodump-ng`.
2. **Capture setup:** Locked the adapter to the target channel and began writing all traffic to a `.cap` file:
   ```bash
   sudo airodump-ng -c <channel> --bssid <BSSID> -w capture wlan0mon
   ```
3. **Handshake forcing:** Sent targeted deauthentication frames to the connected client to force a reassociation and capture the WPA 4-way handshake:
   ```bash
   sudo aireplay-ng -0 5 -a <BSSID> -c <client MAC> wlan0mon
   ```
4. **Verification:** Confirmed a valid handshake was captured:
   ```bash
   aircrack-ng capture-01.cap # → 1 handshake
   ```
5. **Offline dictionary attack:** Tested the handshake against the `rockyou.txt` wordlist (~14M entries):
   ```bash
   aircrack-ng capture-01.cap -w /usr/share/wordlists/rockyou.txt
   ```
6. **Automated cross-check:** Repeated the full flow using `wifite` to validate the manual result against an automated tool.

### Findings
* WPA 4-way handshake successfully captured.
* Dictionary attack against `rockyou.txt` did not recover the password.
* **Conclusion:** The network's WPA2-PSK password is not present in common breach/wordlist compilations and resists a standard offline dictionary attack.

### Defensive Takeaway
WPA2's security against offline attacks depends entirely on password strength, since the handshake itself (not the password) is what is transmitted over the air. A password absent from common wordlists is the single biggest defense against this attack class.

---

## Part 2 — Open Network Setup & Traffic Metadata Analysis

### Objective
Demonstrate what is (and isn't) visible to someone monitoring traffic on an open, unencrypted Wi-Fi network, using my own phone as the test client.

### Methodology
1. **Access point setup:** Configured an open (unauthenticated) test SSID using `hostapd`, with DHCP served via `dnsmasq` and NAT/forwarding via `iptables` to provide internet access to connected clients.
2. **Client connection:** Connected personal phone to the test network.
3. **Traffic capture:**
   ```bash
   sudo tcpdump -i any -w phone_traffic.pcap
   ```
4. **Analysis in Wireshark:**
   * `dns` — enumerate all domains resolved by the device.
   * `tls.handshake.type == 1` — extract SNI (destination domain) from every TLS Client Hello without decrypting the session.
   * `http` — check for any legacy unencrypted HTTP traffic.
   * `Statistics → Protocol Hierarchy / Conversations` — traffic volume/destination breakdown.

### Findings
* **Visible without decryption:** DNS queries, TLS SNI (destination hostnames), traffic volume, and timing for every connection — revealing communication patterns even though the Wi-Fi network was unencrypted.
* **Not visible:** Any actual application payload (page content, credentials, iCloud sync data, messages) due to end-to-end TLS encryption.
* No plaintext HTTP traffic was observed during standard operation.

### Defensive Takeaway
An open Wi-Fi network exposes metadata (destination domains, traffic timing/volume), but not payload content when TLS is in use. To protect metadata on public/untrusted networks, end users must encapsulate traffic within an encrypted VPN tunnel.

---

## Part 3 — Cleartext HTTP Login Form Demonstration (badssl.com)

### Objective
Demonstrate how an unencrypted HTTP (non-TLS) login form appears to a passive network observer.

### Methodology
1. Navigated to `http-login.badssl.com` — a public test domain maintained by the badssl.com project.
2. Submitted a demo login form while capturing traffic with `tcpdump`.
3. Inspected the captured stream in Wireshark via:
   ```text
   ip.addr == <phone IP> && http
   ```
4. Analyzed the payload using `Follow → HTTP Stream` on the `POST /submit/` request.

### Findings
* The entire HTTP request and response — including headers, endpoints, and form parameters — was visible in cleartext.
* Demonstrates why HTTPS and HSTS are mandatory for user authentication endpoints and why modern browsers explicitly flag plaintext HTTP forms as "Not Secure."

---

## Part 4 — SNI-Based DPI Evasion in VPN Traffic & Detection Analysis

### Objective
Investigate how VPN protocols behave when inspected at the network layer on a rogue AP, specifically observing TLS handshake metadata and analyzing techniques used to evade Deep Packet Inspection (DPI) censorship systems.

### Methodology
1. **Lab Setup:** Re-used the isolated rogue AP environment (`hostapd` + `dnsmasq` + `iptables` NAT/MASQUERADE on Kali Linux).
2. **Client Traffic:** Connected mobile client running a VPN tunnel designed to bypass state-level internet censorship.
3. **Packet Sniffing:** Captured TLS handshake negotiation using `bettercap` (`net.sniff` module filtered on HTTPS/TLS traffic).
4. **Observation:** Inspected outgoing `Client Hello` packets prior to session key establishment.

### Findings
* **Observed SNI Payload:** Captured repetitive TLS `Client Hello` frames containing a static, non-routable Server Name Indication:
  ```text
  sni 192.168.50.41 > https://fuck.rkn:23576
  ```
* **Mechanism:** Standard DPI firewalls inspect unencrypted plaintext SNI headers to block restricted domains without decrypting payloads. The observed VPN client spoofed a non-existent, politically-charged SNI string to prevent matching against ISP domain blacklists.
* **Limitation:** While effective against naive domain-name blocklists, using a static spoofed string creates a deterministic signature that DPI systems can easily add to customized detection rules.

### Defensive & Detection Takeaway (Blue Team Angle)
A SOC or network security analyst can detect such evasive traffic without payload decryption by:
1. **DNS Mismatch / Missing Queries:** Flagging TLS handshakes where no preceding DNS query occurred, or where the SNI domain does not resolve to the destination IP.
2. **JA3/JA3S Fingerprinting:** Identifying the client application via its TLS cipher suite order and extension parameters, which remain identifiable regardless of the spoofed SNI string.
3. **Endpoint Reputation:** Analyzing destination IP ASN/reputation feeds independently of SNI values.

---

## Tools Reference

| Tool | Purpose |
|---|---|
| **airodump-ng** | Wireless recon / packet capture |
| **aireplay-ng** | Deauthentication (handshake forcing) |
| **aircrack-ng** | Handshake verification / dictionary attack |
| **wifite** | Automated WPA2 attack workflow |
| **hostapd** | Software access point |
| **dnsmasq** | DHCP/DNS server for test AP |
| **iptables** | NAT/MASQUERADE packet routing & firewalling |
| **bettercap** | Network sniffing, HTTPS/SNI inspection & MITM testing |
| **Wireshark / tcpdump** | Deep packet capture and protocol analysis |

---

## Ethical Disclosure
All testing was performed exclusively against infrastructure and devices I personally own, in a private laboratory environment isolated from third parties. No unauthorized networks, devices, or data were targeted at any point.

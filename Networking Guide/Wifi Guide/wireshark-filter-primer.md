# Wireshark Filter Primer — Wi-Fi & Network Troubleshooting

**Audience:** Help desk, system administrators, security staff
**Purpose:** Read captures from the macOS Wi-Fi Troubleshooting Guide (§2.3 Sniffer, `tcpdump -I`) and the Network Simulation Lab Guide quickly, using display filters that isolate the fault.
**Scope:** Wireshark **display filters** (the top bar, green when valid). Capture filters (BPF) are noted separately where useful.

---

## 0. Orientation

- **Display filter vs. capture filter:** display filters (`ip.addr == ...`) hide/show already-captured packets and can be changed anytime. Capture filters (`host ...`, BPF syntax) decide what gets recorded and can't be changed after the fact. Prefer display filters unless the capture is huge.
- **Combine with:** `and` / `&&`, `or` / `||`, `not` / `!`. Parentheses group. Example: `wlan.fc.type_subtype == 0x0c and wlan.sa == aa:bb:cc:dd:ee:ff`.
- **Comparisons:** `==`, `!=`, `>`, `<`, `>=`, `<=`, `contains`, `matches` (regex).
- **802.11 requires monitor-mode captures** with a radiotap header (Wireless Diagnostics Sniffer, `tcpdump -I`, or a Linux monitor capture). A normal associated capture won't have management/control frames.
- **Right-click any field → Apply as Filter → Selected** builds the filter syntax for you — the fastest way to learn field names.

---

## 1. 802.11 / Wi-Fi (RF and management layer)

These need a monitor-mode capture. They read the *unencrypted* management and control frames — which is where most Wi-Fi faults live, regardless of WPA version.

### 1.1 Frame types

| Filter | Shows |
|---|---|
| `wlan.fc.type == 0` | All management frames (beacons, probes, auth, assoc, deauth) |
| `wlan.fc.type == 1` | All control frames (ACK, RTS/CTS, block-ack) |
| `wlan.fc.type == 2` | All data frames |
| `wlan.fc.type_subtype == 0x08` | Beacons |
| `wlan.fc.type_subtype == 0x04` | Probe requests (clients hunting for SSIDs) |
| `wlan.fc.type_subtype == 0x05` | Probe responses |
| `wlan.fc.type_subtype == 0x0b` | Authentication frames |
| `wlan.fc.type_subtype == 0x00` | Association requests |
| `wlan.fc.type_subtype == 0x01` | Association responses |

### 1.2 The high-value diagnostic filters

| Goal | Filter | What it tells you |
|---|---|---|
| **Deauth / disassoc** (attack or sticky client) | `wlan.fc.type_subtype == 0x0c or wlan.fc.type_subtype == 0x0a` | A flood of these = deauth attack or an AP kicking clients. Check the reason code (below). |
| **Deauth reason code** | `wlan.fixed.reason_code` (add as column) | Reason 7 = "class 3 frame from nonassociated"; 15 = 4-way handshake timeout; common attack floods show reason 1/7. |
| **Retries** (RF trouble) | `wlan.fc.retry == 1` | High retry ratio = interference, weak signal, or congestion. Compare count to total frames on that BSSID. |
| **One BSSID only** | `wlan.bssid == aa:bb:cc:dd:ee:ff` | Focus on one AP. |
| **One client** | `wlan.sa == <mac> or wlan.da == <mac>` | Follow a single device (beware MAC randomization). |
| **Power-save flapping** | `wlan.fc.pwrmgt == 1` | Client dropping into power save aggressively can look like "drops". |
| **RTS/CTS storms** | `wlan.fc.type_subtype == 0x1b or wlan.fc.type_subtype == 0x1c` | Excessive RTS/CTS = hidden-node or heavy contention. |

### 1.3 EAPOL / WPA handshake (auth failures)

| Filter | Shows |
|---|---|
| `eapol` | The WPA/WPA2/WPA3 4-way handshake key frames |
| `eap` | The EAP exchange (802.1X: identity, method, success/failure) |
| `eap.code == 4` | EAP **Failure** — the auth was rejected |
| `eap.code == 3` | EAP **Success** |
| `wlan.rsn.akms.type` (as column) | Which AKM the network advertises (PSK vs. 802.1X vs. SAE) |

**Reading a handshake:** a healthy WPA2 4-way handshake is four EAPOL messages (M1–M4) in quick succession. If you see M1/M2 repeating with no M3/M4, the PSK or PMK is wrong (client can't prove it has the key). For 802.1X, an `eap.code == 4` pinpoints rejection — correlate its timestamp with the RADIUS log (see Wi-Fi guide §4.6 and Lab 5.8).

### 1.4 Signal in the packet list

Add columns (Preferences → Columns, field type "Custom"): `radiotap.dbm_antsignal` (RSSI) and `radiotap.channel.freq`. Now you can sort/scan signal per frame and correlate weak RSSI with retries — the capture-side version of the History inspector.

### 1.5 Decrypting Wi-Fi data (when you legitimately can)

WPA2-PSK data payloads can be decrypted in Wireshark **only if** you have the passphrase **and** captured that client's 4-way handshake. Wireshark → Preferences → Protocols → IEEE 802.11 → enable decryption, add key as `wpa-pwd`. WPA3-SAE's forward secrecy defeats this. Management/control frames are never encrypted, so the §1.2 filters work regardless.

---

## 2. IP / transport layer (both Wi-Fi and simulation-lab captures)

These work on any normal capture (no monitor mode needed) — use them on the Mac's own traffic or a lab capture.

### 2.1 Addressing and scoping

| Filter | Shows |
|---|---|
| `ip.addr == 192.168.10.5` | All traffic to/from a host |
| `ip.src == X` / `ip.dst == X` | Direction-specific |
| `ip.addr == 192.168.10.0/24` | A subnet |
| `eth.addr == <mac>` | By MAC (L2) |
| `!(ip.addr == 192.168.10.5)` | Everything except a host (noise reduction) |
| `tcp.port == 443` / `udp.port == 53` | By service port |

### 2.2 DHCP (Lab 5.1, Wi-Fi §4.3)

| Filter | Shows |
|---|---|
| `dhcp` (or `bootp` on older builds) | All DHCP traffic |
| `dhcp.option.dhcp == 1` | Discover |
| `dhcp.option.dhcp == 2` | Offer |
| `dhcp.option.dhcp == 3` | Request |
| `dhcp.option.dhcp == 5` | ACK |
| `dhcp.option.dhcp == 6` | NAK |

**Reading it:** healthy = Discover → Offer → Request → ACK (DORA). Discover with **no** Offer = no reachable DHCP server / wrong VLAN (client ends up with 169.254.x.x). A NAK = server rejecting the requested address (scope/lease mismatch).

### 2.3 DNS (Lab 5.2, Wi-Fi §4.3)

| Filter | Shows |
|---|---|
| `dns` | All DNS |
| `dns.flags.response == 0` | Queries only |
| `dns.flags.response == 1` | Responses only |
| `dns.flags.rcode != 0` | Errors (rcode 2 = SERVFAIL, 3 = NXDOMAIN) |
| `dns.time > 0.5` | Slow responses (> 500 ms) — resolver latency |

**Reading it:** query with no matching response = resolver unreachable/wrong. Response with `NXDOMAIN` = name genuinely doesn't exist; `SERVFAIL` = resolver couldn't complete (often upstream/DNSSEC). Compare against a query to a known-good resolver.

### 2.4 TCP health (performance, resets, retransmits)

| Filter | Shows |
|---|---|
| `tcp.flags.reset == 1` | RSTs — connection refused/torn down (firewall, closed port, app crash) |
| `tcp.flags.syn == 1 and tcp.flags.ack == 0` | SYNs — connection attempts; many with no SYN/ACK = blocked/unreachable |
| `tcp.analysis.retransmission` | Retransmits — packet loss on the path (Lab 5.9) |
| `tcp.analysis.duplicate_ack` | Dup ACKs — receiver missing segments (loss) |
| `tcp.analysis.zero_window` | Receiver told sender to stop — endpoint overwhelmed, not the network |
| `tcp.analysis.flags` | All of Wireshark's flagged TCP problems at once |
| `tcp.stream eq 0` | Isolate one conversation (increment the number) |

**Reading it:** heavy `retransmission` + `duplicate_ack` = path loss (matches the netem/Link Conditioner loss profile). `zero_window` points at the host, not the link. A `reset` right after a SYN = actively refused (ACL/firewall, Lab 5.6).

### 2.5 ICMP (reachability, MTU) — Lab 5.10

| Filter | Shows |
|---|---|
| `icmp` | All ICMP |
| `icmp.type == 8` / `icmp.type == 0` | Echo request / reply (ping) |
| `icmp.type == 3` | Destination unreachable |
| `icmp.type == 3 and icmp.code == 4` | **Fragmentation needed but DF set** — the MTU/PMTUD smoking gun |
| `icmp.type == 11` | TTL exceeded (routing loop, or normal traceroute) |

### 2.6 ARP (duplicate IP, gateway issues) — Lab 5.3

| Filter | Shows |
|---|---|
| `arp` | All ARP |
| `arp.opcode == 1` / `== 2` | Request / reply |
| `arp.duplicate-address-detected` | Wireshark's own duplicate-IP flag |

**Reading it:** ARP request for the gateway with no reply = gateway down or wrong subnet. Two MACs answering for one IP = duplicate address / rogue.

### 2.7 802.1X on wired / VLAN tags — Lab 5.5, 5.8

| Filter | Shows |
|---|---|
| `eapol` | Wired 802.1X (same as Wi-Fi EAP over LAN) |
| `vlan` | 802.1Q-tagged frames |
| `vlan.id == 20` | A specific VLAN (confirm a frame is on the VLAN you expect) |

---

### 2.8 Real-time media — Zoom / VoIP / RTC (Video Conferencing playbook)

Real-time media rides **UDP**, so the `tcp.analysis.*` filters in §2.4 do **not** apply — a Zoom call generates no TCP retransmit events because there are no retransmissions. Analyze the UDP flows instead.

| Goal | Filter | Notes |
|---|---|---|
| Zoom media flows | `udp.port >= 8801 and udp.port <= 8810` | Zoom's primary media range; presence = healthy UDP path |
| Zoom NAT traversal | `udp.port == 3478 or udp.port == 3479` | STUN/TURN |
| STUN (any RTC) | `stun` | ICE/STUN negotiation for Zoom/Teams/Meet/WebEx |
| RTP media (Teams/Meet/SIP) | `rtp` | Zoom uses proprietary framing so it may not decode as RTP; Teams/Meet/WebEx often do. Try *Decode As → RTP* on a media UDP stream |
| **Detect TCP fallback** | `tcp.port == 443 and ip.addr == <zoom-ip>` **with no** `udp.port >= 8801` present | Media over TCP/443 only = UDP is blocked = silent quality degrade (playbook §5.3) |
| DSCP / QoS marking | `ip.dsfield.dscp != 0` | Are packets marked? EF (46) = voice, AF41 (34) = video. Missing/zeroed = markings stripped (playbook §5.7) |
| One media conversation | `ip.addr == <peer> and udp` | Isolate a single flow |

**Reading it:**

- **UDP present on 8801–8810** = Zoom is using its preferred media path. Good.
- **Only TCP/443 to Zoom, no UDP** = fallback; media is being retransmitted and head-of-line-blocked — the silent-degrade signature. Open outbound UDP 8801–8810 / 3478–3479.
- **`ip.dsfield.dscp == 0`** on media leaving the host, or arriving with marks cleared = QoS bleached somewhere in the path; the call gets no priority on a congested link.
- **UDP jitter/loss** isn't a single field — use **Statistics → I/O Graph** on the media UDP stream to see gaps/bursts, or better, read Zoom's in-app Statistics (playbook §2), which reports per-stream jitter and loss directly.

> For Zoom specifically, the app's own Statistics panel (playbook §2) beats packet analysis for jitter/loss numbers. Reach for Wireshark here mainly to prove **UDP-vs-TCP fallback** and **DSCP survival** — the two things Zoom Statistics can't show you.

## 3. Workflow recipes

**"Is this a deauth attack?"** (monitor capture)
```
wlan.fc.type_subtype == 0x0c
```
Add `wlan.sa`, `wlan.da`, `wlan.fixed.reason_code` as columns. A burst from one source to many clients (or broadcast) with a repeating reason code = attack. Isolated, occasional deauths are normal.

**"Why won't this client authenticate?"** (monitor capture)
```
eapol or eap
```
Follow the sequence. Handshake stalls after M2 = key mismatch (PSK) ; `eap.code == 4` = 802.1X reject → RADIUS log.

**"Connected but no internet"** (normal capture)
```
dhcp or dns or icmp
```
Confirm DORA completed, DNS resolves, and ICMP to the gateway/internet succeeds — walks the same layers as the triage tree §2.

**"The link is slow / lossy"** (normal capture)
```
tcp.analysis.flags
```
Ratio of retransmissions to total packets quantifies the loss; `zero_window` shifts blame to the endpoint.

**"Something is being blocked"** (normal capture)
```
tcp.flags.reset == 1 or icmp.type == 3
```
RSTs and unreachables show *what* is refusing and *where* — the ACL/firewall signature.

---

## 4. Practical tips

- **Coloring rules** (View → Coloring Rules): pre-color retransmissions red and DNS errors orange so problems jump out while scrolling.
- **Statistics → Conversations** ranks talkers by bytes/packets — find the top flow before filtering.
- **Statistics → I/O Graph:** plot `tcp.analysis.retransmission` over time to see *when* loss spiked.
- **Follow → TCP/HTTP Stream** reassembles a whole conversation into readable form.
- **tshark** (CLI, ships with Wireshark) applies the same display filters headlessly — handy for scripted checks on lab captures:
  ```
  tshark -r capture.pcapng -Y "wlan.fc.type_subtype == 0x0c" -T fields -e wlan.sa -e wlan.da
  ```
- **Time reference:** select a packet → Set/Unset Time Reference to measure deltas (e.g., time from Discover to ACK, or M1 to M4).

**Legal note:** capture and analyze only on networks and devices you own or are authorized to troubleshoot. Generating deauth/attack traffic to produce sample captures is lab-gear-only (see Wi-Fi guide §7.4).

---

## References

- Wireshark display-filter reference: https://www.wireshark.org/docs/dfref/
- 802.11 filter fields: https://www.wireshark.org/docs/dfref/w/wlan.html
- Companion docs (this folder): macOS Wi-Fi Troubleshooting Guide, Network Simulation Lab Guide, Network Triage Decision Tree, Video Conferencing / Real-Time Media Troubleshooting (§2.8 supports it)

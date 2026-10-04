# Wi-Fi Troubleshooting Guide — macOS Client

**Audience:** Help desk, system administrators, and security staff
**Tools covered:** macOS native tools, WiFi Explorer Pro 3, compatible Linux tools
**Applies to:** macOS 14 (Sonoma) and later. Version-specific notes are flagged inline.

---

## 1. Triage Workflow

Troubleshoot bottom-up. Each layer depends on the one below it — don't debug DNS if the client has no IP; don't debug DHCP if RSSI is -85 dBm.

| Layer | Symptom | First check |
|---|---|---|
| 1. RF / Physical | SSID not visible, weak signal | Signal/SNR, channel congestion (WiFi Explorer, `wdutil info`) |
| 2. Association/Auth | "Incorrect password", endless connect spinner | Security mode, saved network profile, RADIUS/802.1X logs |
| 3. IP / DHCP | Connected, self-assigned 169.254.x.x address | `ipconfig getsummary en0` |
| 4. DNS | "Server not found" but ping-by-IP works | `scutil --dns`, `dig` |
| 5. Performance | Slow, intermittent, high latency | `networkQuality`, channel utilization, roaming behavior |

Quick first move on any ticket:

```
sudo wdutil info
```

One screen gives you: SSID, BSSID, channel/width, RSSI, noise, SNR-relevant data, Tx rate, security type, IP config, and DNS — enough to place the fault in a layer.

---

## 2. macOS Native Tools

### 2.1 Wi-Fi menu (Option-click)

Hold **Option (⌥)** and click the Wi-Fi menu bar icon. Shows live: IP address, router, security, BSSID, channel, RSSI, noise, Tx rate, PHY mode (11ax/ac/n), MCS index. Zero-install, first stop at a user's desk.

### 2.2 Wireless Diagnostics.app

Option-click Wi-Fi menu → **Open Wireless Diagnostics**. Ignore the wizard; the value is in the **Window** menu:

| Window | Use |
|---|---|
| Info | Live connection detail, same data as wdutil |
| Logs | Enable verbose Wi-Fi logging for intermittent issues |
| Scan | Built-in scanner: SSIDs, BSSIDs, channels, RSSI, noise, security; recommends best channels |
| Performance | Live rolling graph of RSSI, noise, and Tx rate — walk the floor with this to map dead zones |
| Sniffer | Native monitor-mode capture to .pcap on a chosen channel/width (disconnects Wi-Fi while capturing) |

Sniffer output lands in `/var/tmp` — import the .pcap directly into WiFi Explorer Pro 3 (see §3.7) or Wireshark.

### 2.3 Command line

**Status and interface info**

```
sudo wdutil info                          # full Wi-Fi + network status dump
sudo wdutil dump                          # extended internal state
system_profiler SPAirPortDataType         # adapter caps, current network, and nearby networks
networksetup -listallhardwareports        # confirm Wi-Fi device name (usually en0)
ifconfig en0                              # link state, MAC address
```

> **SSID redaction (macOS 14+):** `wdutil` and `system_profiler` show `<redacted>` for SSID/BSSID unless the terminal app has Location Services permission (System Settings → Privacy & Security → Location Services). Grant it once per terminal app.

> **`airport` is gone:** the classic `/usr/libexec/airport` CLI was removed in macOS 14.4. `wdutil` and `system_profiler SPAirPortDataType` are the replacements; WiFi Explorer covers the scanning role.

**Interface control and saved networks**

```
networksetup -setairportpower en0 off && networksetup -setairportpower en0 on
networksetup -getairportnetwork en0                    # current SSID
networksetup -listpreferredwirelessnetworks en0        # saved network order
networksetup -removepreferredwirelessnetwork en0 "SSID"  # forget a network (fixes stale profiles)
```

**DHCP / IP layer**

```
ipconfig getsummary en0        # lease details, DHCP server, router, lease time
ipconfig getpacket en0         # raw DHCP ACK contents
ipconfig getifaddr en0         # just the IPv4 address
```

A `169.254.x.x` address = DHCP failed. Renew with the power-cycle above, then check VLAN/DHCP scope on the infrastructure side.

**DNS and reachability**

```
scutil --dns                   # resolver configuration actually in use
scutil --nwi                   # network state / primary interface
dig example.com                # test resolution against configured resolver
dig @1.1.1.1 example.com       # bypass configured resolver to isolate DNS vs. path
ping -c 5 <gateway-ip>         # local segment health
traceroute 8.8.8.8             # where the path breaks
```

**Performance**

```
networkQuality -v              # Apple's built-in throughput/responsiveness (RPM) test
networkQuality -v -I en0       # force test over Wi-Fi interface
```

**Packet capture (monitor mode)**

```
sudo tcpdump -I -i en0 -w ~/Desktop/wifi.pcap        # -I = monitor mode; disassociates the client
sudo tcpdump -i en0 -n port 53                        # normal mode: watch DNS while associated
```

See §2.5 for why monitor mode — not promiscuous mode — is what you want on Wi-Fi.

### 2.5 Promiscuous vs. monitor mode

A common point of confusion. On wired Ethernet, promiscuous mode captures everything. On Wi-Fi it does not, and the two modes are not interchangeable.

| | Promiscuous mode | Monitor mode (RFMON) |
|---|---|---|
| Association | Stays connected to the AP | Not associated — listens to raw air |
| Captures | Frames the AP forwards to this client: your unicast + broadcast/multicast | All 802.11 frames on the tuned channel, from every device |
| 802.11 mgmt/control frames | No | Yes (beacons, probes, deauth, EAPOL) |
| Other clients' unicast data | Encrypted and mostly not forwarded to you anyway | Seen, but payloads encrypted per-client |
| macOS support | Yes (tcpdump/Wireshark default) | Yes, native on built-in adapter |
| Stays connected? | Yes | No — disassociates for the capture |

**Promiscuous mode** is supported and is actually the default for tcpdump/Wireshark (`sudo tcpdump -i en0`; add `-p` to disable). But on a modern WLAN it's near-useless for RF analysis: the AP only forwards frames addressed to you plus broadcast/multicast, and under WPA2/WPA3 every other client's unicast is encrypted with per-client pairwise keys. You'll mostly see your own traffic, ARP, mDNS, and broadcast chatter.

**Monitor mode** is what WLAN troubleshooting needs, and macOS supports it natively:

```
sudo tcpdump -I -i en0 -w capture.pcap    # -I = monitor mode
```

Better still, use **Wireless Diagnostics → Window → Sniffer**, which lets you pick channel and width — plain `tcpdump -I` captures on whatever channel the radio was last on and can't retune it. Wireshark also exposes a monitor-mode checkbox for en0. This is the same mechanism WiFi Explorer's passive scan uses.

Caveats:

- Monitor mode drops your Wi-Fi connection for the duration of the capture.
- Data payloads stay encrypted. With WPA2-PSK you can decrypt in Wireshark *if* you have the passphrase **and** captured the target's 4-way handshake; WPA3-SAE forward secrecy makes even that impractical.
- macOS captures **one channel at a time**. Multi-channel/roaming analysis needs channel hopping or a multi-radio Linux sensor (§5.1, §5.3).
- The frames that matter most for troubleshooting — beacons, probes, deauth/disassoc, and the EAPOL handshake (§4.5, §4.6) — are unencrypted management/control frames, so monitor mode reads them fine regardless of WPA version.

**Logging and evidence collection**

```
sudo wdutil log                # toggle verbose Wi-Fi daemon logging
sudo wdutil diagnose           # collect a full Wi-Fi diagnostic archive (path printed on completion)
sudo sysdiagnose -f ~/Desktop  # full system diagnostic (large; for escalations to Apple/vendor)
log stream --predicate 'processImagePath contains "airportd"'   # live Wi-Fi daemon log
```

### 2.4 Known macOS gotchas

Client-side features that masquerade as network problems:

| Feature | Symptom | Check / Fix |
|---|---|---|
| **Private Wi-Fi Address** (per-network MAC randomization; macOS 15+ can also rotate) | Breaks MAC-based NAC allowlists, DHCP reservations, captive-portal MAC caching; device "disappears" from inventory | System Settings → Wi-Fi → Details (next to SSID) → Private Wi-Fi Address → Off/Fixed for managed networks, or set via MDM Wi-Fi payload |
| **iCloud Private Relay** | Internal split-horizon DNS fails in Safari, content filtering bypassed, geolocation wrong, "slow browsing" reports | System Settings → Apple ID → iCloud → Private Relay; disable per-network via Wi-Fi Details → Limit IP Address Tracking |
| **Encrypted DNS / VPN profiles** | Client ignores DHCP-provided DNS; `dig` results differ from expectations | `scutil --dns` shows the actual resolver order; check installed profiles (`sudo profiles show -type configuration`) and VPN state |
| **Captive Network Assistant sandbox** | Portal page won't load or login doesn't stick | Close the CNA popup, open a real browser to `http://captive.apple.com` |
| **AWDL (AirDrop/AirPlay/Handoff)** | Periodic latency spikes and jitter — the `awdl0` interface channel-hops off the serving channel | Confirm with `ping -i 0.2 <gateway>` showing rhythmic spikes; `sudo ifconfig awdl0 down` to test (re-enables itself; disable AirDrop to persist) |
| **Location Services redaction** | Scripts/agents get `<redacted>` SSIDs | See §2.3 note |
| **Security/filtering agents** (Zscaler, Netskope, Cisco Umbrella, CrowdStrike) | Connectivity fine but specific apps/sites blocked or slow; DNS answers look "wrong" | Check the agent's own status UI; `scutil --dns` for injected resolvers; test with the agent paused (if policy allows) |
| **Proxy / PAC file** | Some apps work, others don't; browser differs from `curl` | System Settings → Network → Details → Proxies; check for an auto-config (PAC) URL |
| **Local firewall / Little Snitch-type tools** | One app has no connectivity while others are fine | Check the tool's rules; macOS Application Firewall in System Settings → Network |

Client-side interceptors are the most-missed cause of "the network is broken" tickets on managed Macs: the packets reach the machine fine, but a security agent, proxy, or content filter drops or redirects them above the network layer. Rule them out before escalating to the network team — a fault that only affects certain apps or destinations, with a healthy `wdutil info`/`ipconfig`/gateway ping, points here rather than at Wi-Fi.

---

## 3. WiFi Explorer Pro 3

WiFi Explorer Pro 3 is the primary RF-layer analysis tool: scanning, signal history, channel utilization, client discovery, and spectrum analysis integration.

### 3.1 Pick the right scan mode

Scan Mode menu, in the toolbar:

| Mode | How it works | Use when |
|---|---|---|
| **Active** | Sends probe requests on all 2.4/5/6 GHz channels; works while connected | Default for general surveys |
| **Directed** | Active scan probing one specific SSID | Isolating one network's APs (finds hidden SSIDs too) |
| **Passive – All Channels** | Monitor mode, hops channels listening for beacons; disconnects Wi-Fi | Client discovery, seeing what active scans miss |
| **Passive – Single Channel** | Monitor mode parked on one channel, 100% dwell | Deep-dive on one AP/channel; best client detection |
| **Listener** | Receives encapsulated capture feeds (PEEKREMOTE UDP 5000, TZSP UDP 37008) from APs in sniffer mode | Remote troubleshooting via enterprise AP sniffer feature |
| **Sensor** | Remote passive scan via Linux/Android sensor over SSH | Surveying a remote site from your desk (see §5.1) |

Passive mode requires an Intel Mac or a Wi-Fi 6E-capable Apple silicon Mac, and cannot run while associated (the app disconnects/reconnects automatically).

### 3.2 Read the signal numbers

Select the user's network and open the **History inspector** — it logs signal (RSSI), noise, and SNR with avg/max/min, exportable to CSV.

| Metric | Good | Acceptable | Problem |
|---|---|---|---|
| RSSI | ≥ -60 dBm | -60 to -70 dBm | < -70 dBm (VoIP/video need ≥ -67) |
| SNR | ≥ 25 dB | 20–25 dB | < 20 dB (retries, rate drops) |
| Noise floor | ≤ -90 dBm | -85 to -90 dBm | > -85 dBm (suspect interference) |

Good RSSI with poor SNR = interference problem, not coverage problem → check Utilization (§3.4) and spectrum (§3.6).

### 3.3 Issues inspector — automated findings

The Issues inspector flags misconfigurations for the selected network. Findings it can raise, and what to do:

| Finding | Action |
|---|---|
| 40 MHz width in 2.4 GHz | Reconfigure AP to 20 MHz in 2.4 GHz |
| 80 MHz width in 5 GHz (dense env.) | Consider 40 MHz to reduce overlap |
| Overlapping 2.4 GHz channel (not 1/6/11) | Move AP to 1, 6, or 11 |
| Too Many Networks on channel | Re-plan channels; move to 5/6 GHz |
| Low Signal Strength | Coverage issue — AP placement/power |
| Legacy Equipment | AP lacks modern 802.11 support — replace |
| Weak Security | Open/WEP/WPA-TKIP — remediate immediately (security finding) |
| Hidden Network | Cosmetic at best; hiding ≠ security |
| Separate Network Names per band | Consolidate SSIDs; let band steering work |
| Large Group of Networks per AP | Every SSID adds beacon overhead — prune SSIDs |

### 3.4 Utilization inspector — congestion

Shows per-channel: beacon overhead %, network count, overlapping networks, and (passive mode) client count. Beacon overhead ratings: **Low** <10%, **Medium** 10–20%, **High** 20–50%, **Very High** ≥50%. High beacon overhead means airtime is being burned before any user data flows — reduce SSID count or AP density on that channel.

Select a channel row to filter the network table to that channel; check "Include overlapping networks" to see everything actually contending.

### 3.5 Clients inspector — who's on the AP

Passive/Listener/Sensor modes only (not Active). Shows client MAC, vendor, BSSID, channel, RSSI, first/last seen. For a reliable client inventory of one AP: **Passive – Single Channel** on that AP's channel, and let it run — detection improves over time.

Security use: spot unexpected client vendors on sensitive SSIDs, and confirm which BSSID a user's machine actually associated to.

### 3.6 Spectrum analysis (non-Wi-Fi interference)

With a supported USB analyzer (MetaGeek Wi-Spy 2.4x/DBx, Oscium WiPry 2500x/Clarity, Wi-Spy Lucid, NetAlly NXT-2000), the Spectrum Analysis menu overlays RF energy on the channel view: density view, waterfall view, live/average/max traces, and per-channel utilization trace. Use it when SNR is bad but the Wi-Fi environment looks clean — microwaves, Bluetooth, cameras, and cordless gear don't beacon. A Zigbee inspector (with a supported Zigbee adapter) identifies Zigbee networks in 2.4 GHz.

### 3.7 Import and export

- **Capture files:** opens `.pcap`, `.pcapng`, `.wcap`, `.pkt` with Radiotap/PPI/Prism headers — including captures made by Wireless Diagnostics Sniffer, `tcpdump -I`, or Linux tools (§5.3). Parses beacons/probe responses into scan results and rebuilds the client list from data/control frames.
- **CSV import:** scan results from AirPort Utility (iOS), Analiti (Android), Aruba Utilities, 7Signal Mobile Eye — paste or drag-and-drop.
- **Export:** scan results and History inspector data export to CSV for tickets and reports.

### 3.8 Filters, coloring, annotations

Filter by SSID/vendor/band/channel/security; organize by SSID, vendor, or physical AP. Coloring rules highlight conditions (e.g., color anything with RSSI > -65 dBm, or any open network red). Annotations let you label known APs/BSSIDs — annotate your own infrastructure once, and rogues stand out on every future scan.

---

## 4. Playbooks

### 4.1 SSID not visible

1. Confirm at RF layer: WiFi Explorer **Active** scan. Not present → try **Directed** scan with exact SSID (finds hidden networks and non-broadcast responses).
2. Present in scanner but not in macOS Wi-Fi menu → check band support (6 GHz SSID + non-6E Mac) and country-code/DFS channels.
3. Present but very weak (< -80 dBm) → coverage issue, not client issue.
4. Not present at all → AP down or wrong site; check adjacent channels and other bands before dispatching.

### 4.2 Won't connect / authentication failures

1. `networksetup -removepreferredwirelessnetwork en0 "SSID"` to clear a stale profile, then rejoin.
2. Check security type in WiFi Explorer (Security column) — mismatch between saved profile and current AP config (e.g., AP moved WPA2→WPA3) causes silent failures.
3. Enterprise (802.1X): check certificate trust prompts, then `sudo wdutil log` + `log stream --predicate 'processImagePath contains "airportd"'` while reproducing; correlate with RADIUS server logs. Full workflow in §4.6.
4. Verify the client isn't associating to a distant AP of the same SSID: option-click Wi-Fi menu → note BSSID → find it in WiFi Explorer to see its signal.

### 4.3 Connected, no internet

1. `ipconfig getsummary en0` — valid lease? 169.254.x.x → DHCP failure: check scope exhaustion, VLAN tagging on the SSID.
2. `ping <gateway>` — fails → local/ARP/client-isolation issue; succeeds → move up.
3. `dig example.com` vs `dig @1.1.1.1 example.com` — first fails, second works → DNS server problem. `scutil --dns` shows what the client is actually using.
4. Captive portal suspected: `curl -I http://captive.apple.com` — anything but `Success` HTML means a portal is intercepting.

### 4.4 Slow or intermittent

1. Baseline: `networkQuality -v -I en0`. Note both throughput and responsiveness (RPM) — low RPM with fine throughput = bufferbloat/latency, not coverage.
2. WiFi Explorer History inspector on the user's BSSID for several minutes: sawtooth or dropping RSSI → roaming/coverage; stable RSSI but low SNR → interference.
3. Utilization inspector: many networks or High+ beacon overhead on the channel → congestion; re-channel or steer to 5/6 GHz.
4. Sticky-client roaming: Wireless Diagnostics Performance window while walking with the user; watch whether Tx rate collapses before the client roams.
5. Clean Wi-Fi picture but still bad → spectrum analyzer pass (§3.6) for non-802.11 interference.
6. Test wired vs. wireless throughput to the same iperf3 server (§5.2) to prove/disprove the WLAN as bottleneck.
7. If the complaint is specifically **video calls** (Zoom/Teams/Meet), the fault is usually jitter, packet loss, or upload — not throughput — and a speed test will mislead you. Use the dedicated **Video Conferencing / Real-Time Media Troubleshooting** doc (start with the app's in-app Statistics).

### 4.5 Security checks (rogue AP / evil twin / weak config)

1. Scan with WiFi Explorer, organize by **SSID**: any BSSID broadcasting your SSID that isn't in your annotated inventory is a candidate rogue/evil twin. Check its Vendor column — an enterprise SSID from a random consumer-vendor OUI is a red flag.
2. Sort by Security: flag anything Open/WEP/WPA-TKIP (Issues inspector raises "Weak Security" automatically).
3. Locate a rogue: **Passive – Single Channel** on its channel, watch RSSI in History inspector while walking — signal rises as you approach.
4. Deauth-attack suspicion: capture on the affected channel (Wireless Diagnostics Sniffer or `tcpdump -I`), open the .pcap in Wireshark, filter `wlan.fc.type_subtype == 12`. A flood of deauth frames = active attack.
5. Verify what users actually connect to: Clients inspector shows client MAC ↔ BSSID associations.

### 4.6 802.1X / enterprise Wi-Fi deep dive

**The flow:** client associates → EAP exchange with the RADIUS server (via the AP/controller) → on success, EAPOL 4-way handshake derives encryption keys. Failures cluster in three places: certificate trust, credentials/policy, and profile configuration.

**EAP types on macOS**

| EAP type | Credential | Notes |
|---|---|---|
| EAP-TLS | Client certificate | Preferred. Cert usually delivered via MDM (SCEP/ACME); watch for expiry |
| PEAP (MSCHAPv2) | Username/password | Fails on expired/locked AD passwords; vulnerable to evil-twin credential capture without server cert validation |
| EAP-TTLS | Username/password | Same trust considerations as PEAP |

Without an MDM/configuration profile, macOS prompts the user to trust the RADIUS server certificate on first join — users clicking "Cancel" on that prompt is a classic "wrong password" ticket that isn't.

**Client-side diagnostics**

```
log stream --predicate 'process == "eapolclient"'      # live EAP exchange while reproducing
sudo profiles show -type configuration                  # installed Wi-Fi/cert payloads
security find-identity -v                               # client identities (EAP-TLS) — check expiry
```

In WiFi Explorer Pro 3, select the network and check the security details / RSN information element to confirm the AKM actually advertised (802.1X vs SAE vs PSK) matches what the profile expects — WPA2→WPA3-Enterprise transitions break stale profiles.

**Common failure modes**

| Failure | Evidence | Fix |
|---|---|---|
| RADIUS server cert expired / not trusted / name mismatch | eapolclient log shows TLS failure; user got trust prompt | Renew cert; add anchor + trusted server names to the MDM Wi-Fi payload |
| Client cert expired (EAP-TLS) | `security find-identity` shows expired identity | Re-enroll / renew via MDM |
| Missing intermediate CA on RADIUS server | Works on pre-provisioned devices only | Fix server cert chain |
| Password expired / account locked (MSCHAPv2) | RADIUS reject (e.g., NPS event 6273 reason 16/34) | Reset credential |
| Policy mismatch (wrong group/VLAN) | Auth succeeds then wrong network access | Correlate Calling-Station-Id and policy hit on RADIUS side |
| Private Wi-Fi Address breaks MAC-based rules | Random/rotating MAC in RADIUS logs | Disable Private Address for the SSID via MDM (§2.4) |
| TLS version mismatch (legacy NPS/FreeRADIUS) | TLS alert in eapolclient log | Enable TLS 1.2 on the server |

**Correlate both sides:** timestamp from `log stream` + client MAC (beware rotation) against RADIUS logs. For deep analysis, capture on the AP's channel (Sniffer or `tcpdump -I`) and filter `eapol || eap` in Wireshark — you can see exactly which EAP message the exchange dies on.

**Certificate & PKI lifecycle (the slow-motion outage).** Most 802.1X failures aren't misconfiguration — they're expiry. Certificates are time bombs that fail *en masse* on a predictable date, so treat the lifecycle as an operational task, not a one-time setup:

- **Client certs (EAP-TLS):** delivered via MDM (SCEP/ACME). Monitor expiry fleet-wide from the MDM; auto-renew before expiry. On a Mac, `security find-identity -v` shows the identity and you can inspect validity in Keychain Access. A batch of same-day auth failures across many users almost always means a cert or CA rolled over.
- **RADIUS server cert:** when it renews, clients that pin the old cert or lack the new chain break at once. Push the new anchor + trusted server names via the MDM Wi-Fi payload *before* the server cert changes.
- **CA rotation / intermediate changes:** the highest-blast-radius event. Stage the new CA to clients ahead of time; never cut over the server first.
- **What breaks at scale:** an expired intermediate that "works on already-provisioned devices" but fails every new/re-enrolled one; a renewed server cert with a new subject name that trusted-server-name rules don't match; clients whose system clock is wrong rejecting a valid cert.

Track cert and CA expiry dates in the baseline register (see the Network Ops Templates doc) so renewals are scheduled, not discovered by a wave of tickets.

### 4.7 AP placement — mini site survey

WiFi Explorer Pro 3 plus a MacBook is enough for a walk-test survey of small/medium spaces. (It doesn't render floor-plan heat maps — for large multi-floor deployments use dedicated survey software; this method still validates those designs.)

**1. Define targets first.** Placement is meaningless without requirements:

| Requirement | Target |
|---|---|
| Data-only coverage | RSSI ≥ -70 dBm everywhere |
| VoIP / video / roaming | RSSI ≥ -67 dBm, SNR ≥ 25 dB |
| Cell overlap for roaming | Neighbor AP at ≥ -70 dBm at cell edge (~15–20% overlap) |
| Channel reuse | Same-channel APs not audible above ~ -80 dBm from each other |

**2. AP-on-a-stick test.** Mount one AP (or use an existing one) at the candidate location at intended height/orientation. In WiFi Explorer use **Directed** scan on your SSID (filters out everything else), organize results **by physical AP**, and open the **History inspector** on the candidate BSSID.

**3. Walk the cell edge.** Walk outward, watching live RSSI — Wireless Diagnostics **Performance** window works well side-by-side. The contour where RSSI crosses -67 dBm is the cell boundary; mark it on a floor plan. Walk every room, closet, and stairwell inside the boundary — construction materials beat theory every time.

To log a timestamped walk you can annotate against positions later:

```
while true; do echo "$(date +%T) $(sudo wdutil info | grep -E 'RSSI|Noise|Tx Rate')"; sleep 2; done | tee walk.log
```

The History inspector's CSV export gives the same series with signal/noise/SNR per data point.

**4. Place the next AP** so its -67 dBm contour overlaps the first cell's edge (client roam trigger zone), then repeat. Prefer central, elevated, unobstructed mounts; avoid metal, elevator shafts, and placing APs in hallways when the users are in rooms (hallway placement over-covers corridors and under-covers rooms).

**5. Channel plan from real data.** Use the **Utilization inspector** at each AP location to see per-channel network count and beacon overhead, including neighbors' networks bleeding in. 2.4 GHz: 1/6/11 only, 20 MHz. 5/6 GHz: assign so adjacent cells don't share primary channels; verify same-channel APs hear each other below -80 dBm.

**6. Verify after install.** Re-walk with all APs live: confirm RSSI/SNR targets, run `networkQuality -v -I en0` and an iperf3 test (§5.2) at each cell edge and known trouble spots, and confirm clients actually roam (watch BSSID change via option-click Wi-Fi menu while walking).

**Locating an existing AP** (for relocation or rogue hunting): **Passive – Single Channel** on its channel, History inspector open, and walk — RSSI rises ~6 dB each time you halve the distance. See also §4.5.

### 4.8 IPv6 / dual-stack issues

macOS is aggressively dual-stack: when a network advertises IPv6, the Mac uses it *by default* and prefers it for destinations that publish both A and AAAA records. This is invisible when it works — and a genuine blind spot when it doesn't, because IPv4-only troubleshooting (`ipconfig getsummary`, `ping`, `dig`) all looks healthy while the actual traffic is failing over IPv6.

**How macOS picks a path:** it uses **Happy Eyeballs** — it races IPv6 and IPv4 connections and uses whichever answers first, falling back quickly. Well-behaved failure is invisible; *partial* IPv6 brokenness (address assigned, but no working IPv6 route) causes intermittent stalls and delays as connections try v6, time out, and fall back. Classic symptom: "some sites are slow to start loading, then fine," or one app struggling while another is instant.

**Check the IPv6 state:**

```
ifconfig en0 | grep inet6            # link-local (fe80::) always present; look for a global address
ping6 -c 5 2606:4700:4700::1111      # Cloudflare IPv6 — is IPv6 reachable at all?
dig AAAA example.com                 # does the name even have an IPv6 record?
netstat -rn -f inet6                 # IPv6 routing table / default route
```

**Common dual-stack failure modes:**

| Symptom | Cause |
|---|---|
| Global IPv6 address present, `ping6` to internet fails | Router advertises IPv6 (SLAAC/RA) but has no working upstream — the "broken IPv6" case; Happy Eyeballs mostly masks it but adds latency |
| Works on some networks, not others | One network is IPv6-broken; another is v4-only or v6-clean |
| VPN up, IPv6 leaks or breaks split-tunnel | VPN handles IPv4 only; IPv6 traffic bypasses the tunnel or blackholes |
| Fast on IPv4-only Wi-Fi, slow on home/ISP dual-stack | ISP IPv6 misconfig or CGNAT interaction |

**Isolate it fast:** temporarily disable IPv6 and see if the problem vanishes —

```
networksetup -setv6off Wi-Fi
# ... retest ...
networksetup -setv6automatic Wi-Fi   # restore
```

If disabling IPv6 fixes it, the fault is IPv6 path/config on that network (advise the user/ISP, or set the VPN/network to handle v6 properly). Don't leave IPv6 off as a "fix" beyond diagnosis — it's a workaround that masks the real issue and breaks IPv6-only services.

**Privacy addresses (macOS):** the Mac rotates temporary IPv6 source addresses by default, so the address in a firewall/ACL log changes over time — the IPv6 analogue of the Private Wi-Fi Address caveat (§2.4). Don't build reachability rules against a temporary address.

### 4.9 Captive portals, public & travel Wi-Fi

Hotel, airport, conference, and café networks are a WFH-adjacent reality with their own failure modes — and a security exposure worth briefing travelers on. The pattern: association succeeds, an IP is assigned, but nothing works until a **captive portal** login completes.

**How it should work:** on join, macOS's **Captive Network Assistant (CNA)** auto-probes a known URL (`captive.apple.com`) and, if it gets intercepted, pops a mini-browser for the portal login. When that works, you're done. When it doesn't, you get the classic "connected, no internet."

**Troubleshooting a stuck portal:**

```
curl -I http://captive.apple.com     # expect "Success" HTML if online; a redirect = portal intercepting
```

| Symptom | Fix |
|---|---|
| CNA popup never appears | Manually open `http://captive.apple.com` (plain HTTP — HTTPS won't redirect) in a real browser to force the portal |
| Portal loads but login won't stick | Close the CNA sandbox popup and use Safari/Chrome instead; the mini-browser rejects some portal scripts |
| Portal never loads, DNS fails | Portals often hijack DNS pre-login; don't fight it — the portal must complete first |
| "No internet" after login | Encrypted DNS (DoH/DoT), a custom DNS profile, or a VPN auto-connecting **before** portal completion blocks the probe — see below |

**The VPN / encrypted-DNS ordering trap:** an always-on VPN or an encrypted-DNS profile can try to connect *before* the captive portal is satisfied, and since the portal is blocking all traffic, the tunnel/probe fails and the portal never appears. Order matters: complete the portal login first (may require briefly allowing the VPN to stay disconnected), then bring up the VPN. This is a frequent "my VPN won't connect at the hotel" ticket.

**Security briefing for travelers (tie to the Wireless Security IR runbook):**

- Public SSIDs are trivially spoofed (evil twin, §4.5). "Airport_Free_WiFi" may be an attacker. Prefer a known/official SSID, and treat all public Wi-Fi as hostile.
- **Always use the corporate VPN** on public networks once the portal is up — it protects the traffic even on a malicious AP.
- Turn **off auto-join** for open networks so the Mac doesn't silently reconnect to a honeypot later (System Settings → Wi-Fi → the network → Auto-Join).
- Prefer a **personal hotspot / tethering** over untrusted public Wi-Fi for sensitive work — often the right advice for a traveling exec.
- Never dismiss a certificate-trust warning to "make the portal work" — that's the evil-twin credential-harvest vector.

---

## 5. Linux Tools (compatible and recommended)

### 5.1 Remote sensor for WiFi Explorer Pro 3 (direct integration)

Any Linux box with a compatible Wi-Fi adapter becomes a remote scanner for WiFi Explorer over SSH — scan a branch office from your desk. The [WLAN Pi](https://www.wlanpi.com) ships pre-configured as a sensor.

Required packages: `iproute2`, `iw`; plus `tcpdump` + `wpasupplicant` (passive mode) or [`scandump`](https://github.com/intuitibits/scandump) (active mode).

Grant passwordless sudo for exactly those binaries — create `/etc/sudoers.d/wlandump`:

```
# passive mode
%sudo ALL=(ALL) NOPASSWD: /sbin/iw, /bin/ip, /usr/bin/tcpdump, /usr/sbin/wpa_cli
```

```
sudo chmod -w /etc/sudoers.d/wlandump
```

Add it in WiFi Explorer Pro 3 → Settings → Sensors (+, hostname/IP, optional interface, SSH port), then pick it from the Scan Mode menu. Built-in sensor diagnostics: Settings → Sensors → select sensor → More → Run Diagnostics. Remote spectrum analysis also works through a sensor: `spectools` (Wi-Spy) or `wipry_raw` (Wi-Spy Lucid / WiPry Clarity) on the sensor.

### 5.2 iperf3 — throughput ground truth

Run the server on any wired Linux host; test from the Mac:

```
# Linux
sudo apt install iperf3 && iperf3 -s
# macOS (brew install iperf3)
iperf3 -c <server-ip>            # upload
iperf3 -c <server-ip> -R         # download
```

Wi-Fi throughput far below wired throughput to the same server isolates the fault to the WLAN.

### 5.3 Capture on Linux → analyze in WiFi Explorer Pro 3 / Wireshark

```
sudo ip link set wlan0 down
sudo iw wlan0 set monitor control
sudo ip link set wlan0 up
sudo iw wlan0 set channel 36               # park on the channel under test
sudo tcpdump -i wlan0 -w site-capture.pcap
```

The resulting radiotap pcap opens directly in WiFi Explorer Pro 3 (File → Open) — networks, signal history, and client lists are rebuilt from the capture. `airodump-ng` (aircrack-ng suite) captures are also importable.

### 5.4 Quick Linux-side diagnostics

| Tool | Command | Use |
|---|---|---|
| iw | `iw dev wlan0 link` / `iw dev wlan0 scan` / `iw dev wlan0 station dump` | Link state, scan, per-client stats on Linux APs |
| nmcli | `nmcli dev wifi list` | Fast scan table on NetworkManager systems |
| wavemon | `wavemon` | Live ncurses signal/quality monitor |
| Kismet | `kismet` | Long-running wireless IDS / rogue detection; exports pcapng importable into WiFi Explorer |
| mtr | `mtr 8.8.8.8` | Continuous path loss/latency (also on macOS via brew) |
| tshark | `tshark -r site-capture.pcap -Y "wlan.fc.type_subtype == 12"` | Scripted deauth/frame analysis on captures |

---

## 6. Remote / Work-From-Home Diagnosis

Remote workers change the problem in two ways: **you can't be there** (no walk-test, no AP-on-a-stick), and **you don't control the network** (their ISP, their router, their RF environment). WiFi Explorer's sensor mode doesn't help — it needs SSH to a sensor on the *same* LAN as the target AP, which you don't have. So remote diagnosis shifts from "analyze the RF myself" to "**have the endpoint report on itself**, then isolate which segment owns the fault."

### 6.1 The core question: whose problem is it?

A WFH connectivity ticket has four possible owners. Isolate in this order — each test rules out a layer:

| # | Segment | Test | If it fails |
|---|---|---|---|
| 1 | **Wi-Fi link** (client ↔ home AP) | RSSI/SNR + Tx rate at the desk; wired vs. wireless comparison | Home Wi-Fi problem — §6.3 |
| 2 | **Home LAN / router** | ping/`networkQuality` to the home gateway | Router, cabling, or overloaded home network |
| 3 | **ISP / internet path** | ping + `mtr` to a public target; DNS resolution | ISP outage/congestion — not yours to fix |
| 4 | **Corporate path / VPN** | reach internal resource with VPN up vs. reachability of public internet | VPN concentrator, split-tunnel config, or corp side |

The single most decisive test: **have the user plug into the router with Ethernet.** If the problem vanishes on wired, it's the Wi-Fi link or RF environment (§6.3). If it persists on wired, Wi-Fi is exonerated — stop looking at it and move to LAN/ISP/VPN.

### 6.2 Getting data off a machine you can't touch

You have no physical access and no local sensor. Options, roughly in order of preference:

- **MDM script push** (Jamf, Kandji, Intune, etc.): run a collection script and pull back the output. Best for fleets — no user skill required, and it works even while the user is offline (runs at next check-in). Push the §6.4 script as a policy.
- **Remote screen share / remote shell** (Zoom control, corporate RMM, SSH if enabled): run the §6.4 commands live while the user reproduces the issue. Best for interactive, intermittent problems.
- **User self-service:** send the user a copy-paste one-liner or a signed `.command` script (§6.4) and have them return the output/file. Lowest barrier when MDM/remote access isn't available.
- **Continuous telemetry:** if you run an endpoint monitoring agent, WFH link quality is often already being logged — check there before dispatching a ticket.

> **What does NOT translate to remote:** on-site walk-testing, AP-on-a-stick surveys (§4.7), WiFi Explorer sensor mode (needs a sensor on the home LAN), and scanning the *neighbors'* APs — you only see the RF the user's own machine can see. That's usually enough, because the question is whether *their* link is healthy, not what the whole building looks like.

### 6.3 Diagnosing the home Wi-Fi link specifically

Everything runs on the user's Mac; you read the output remotely. This is where the native tools (§2) carry the whole job — WiFi Explorer needs an operator at the keyboard, so it's only useful here if the *user* has it or you're screen-sharing.

Signal and link health (compare against §3.2 thresholds):

```
sudo wdutil info                 # RSSI, noise, Tx rate, channel/width, PHY mode, security
```

- RSSI < -70 dBm → coverage: user is too far from their router, or through too many walls. Fix is physical (move closer, reposition router, mesh node) — you can't fix it remotely, but you can *diagnose* it and advise.
- Good RSSI but low SNR / high retries → interference: neighbor APs (apartments/townhouses), microwave, USB3, baby monitors. 2.4 GHz is usually the culprit.
- Stuck on 2.4 GHz with a capable router → have them forget/rejoin or split SSIDs; confirm PHY mode shows 11ax/ac on 5 GHz.

Distinguish coverage vs. congestion remotely without WiFi Explorer:

```
# Wireless Diagnostics → Scan (user runs it) reports neighbor count, channels, and RSSI
# or capture a moment-in-time neighbor list:
system_profiler SPAirPortDataType     # "Other Local Wi-Fi Networks" = the user's RF neighborhood
```

A dense "Other Local Wi-Fi Networks" list all crowding channels 1/6/11 explains a congested home environment — common in apartments. Advice: move the router's channel or shift the client to 5/6 GHz.

Home-specific gotchas beyond the office set (§2.4):

- **ISP gateway + separate router (double NAT):** two DHCP servers, weird routing. `ipconfig getsummary en0` showing a 192.168.x.x behind another 192.168.x.x is the tell.
- **Powerline / MoCA / range extenders:** halve throughput and add latency; the user may not mention them.
- **Consumer mesh backhaul:** if the node the user is on backhauls over Wi-Fi, their good RSSI to the node hides a weak node-to-router link. Test throughput to the internet, not just RSSI to the nearest node.
- **Smart-home 2.4 GHz congestion:** dozens of IoT devices on the home SSID.

### 6.4 A self-contained WFH collection script

Push via MDM or hand to the user. Writes a single text file to the Desktop they can send back — it captures the whole stack (§6.1) in one shot, so you can place the fault without a back-and-forth:

```bash
#!/bin/bash
OUT=~/Desktop/wifi-report-$(date +%Y%m%d-%H%M%S).txt
{
  echo "===== $(date) $(scutil --get ComputerName) ====="
  echo "--- Wi-Fi link ---";        sudo wdutil info
  echo "--- RF neighborhood ---";   system_profiler SPAirPortDataType
  echo "--- IP / DHCP ---";         ipconfig getsummary en0
  echo "--- DNS ---";               scutil --dns | head -40
  echo "--- Gateway reachability ---"; ping -c 5 "$(ipconfig getoption en0 router)"
  echo "--- Internet path ---";     ping -c 5 1.1.1.1; traceroute -w1 -q1 1.1.1.1
  echo "--- DNS resolution ---";    dig +short apple.com
  echo "--- Throughput / RPM ---";  networkQuality -v
} > "$OUT" 2>&1
echo "Report saved to $OUT"
```

Reading it: healthy gateway ping but bad internet path = ISP; bad RSSI/Tx in the Wi-Fi link section = home Wi-Fi; everything clean but the *user* still reports failure only for corporate apps = VPN/corp path (§6.5).

For an intermittent problem, have them run this **at the moment it's bad** — a report taken while things are fine proves nothing.

> A hardened, self-classifying version of this collector lives in the `scripts/` folder (`Networking Guide/Wifi Guide/scripts/net-triage-snapshot.sh`) — it runs the same collection, adds a layer verdict and JSON output, and is MDM-pushable. See `scripts/README.md`.

### 6.5 The VPN wrinkle

Most WFH tickets arrive as "the VPN is slow/dropping," which conflates the tunnel with the link under it. Separate them:

- Test the **underlay** first (§6.4): if the home internet path is bad, the VPN can't be good — fix the underlay, the VPN follows.
- **Full-tunnel** VPNs route *everything* corporate-side, so a healthy home link can still feel slow if the VPN concentrator is congested or far away. Compare `networkQuality` with VPN up vs. down.
- **Split-tunnel** sends only corporate subnets through the tunnel — so "internet works but the app doesn't" points at the tunnel/corp side, not the home Wi-Fi.
- VPN clients often override DNS: `scutil --dns` shows whether the corporate resolver is active. Internal split-horizon names failing while public DNS works = VPN DNS scoping, not a home problem (and note iCloud Private Relay / encrypted DNS conflicts from §2.4).
- MTU: some home routers + VPN encapsulation cause fragmentation — symptom is small requests work, large transfers stall. Test with `ping -D -s 1472 <corp-host>` (don't-fragment); failures that succeed at smaller sizes indicate an MTU/MSS problem.
- **Video calls over VPN:** full-tunnel VPNs hairpin Zoom/Teams media through the concentrator, adding latency and often forcing TCP. Best practice is to split-tunnel conferencing traffic direct. See the **Video Conferencing / Real-Time Media Troubleshooting** doc §5.4.

### 6.6 Setting expectations

You can *diagnose* a home Wi-Fi problem remotely with confidence, but you generally can't *fix* the RF — router placement, neighbor interference, and ISP quality are the user's environment. The deliverable for a home-Wi-Fi-caused ticket is usually clear, specific advice (move the router, use Ethernet for calls, switch to 5 GHz, add a mesh node, contact the ISP) backed by the collected data — plus a note in the ticket that root cause is outside corporate control.

---

## 7. Practice & Readiness

The worst time to learn these tools is during a live outage with a user watching. The core skill isn't memorizing commands — it's **knowing what normal looks like** so abnormal jumps out. Build that in a lab first.

### 7.1 Establish baselines (do this before anything breaks)

You can't recognize a bad number if you've never seen a good one. On a known-healthy Mac and network, capture reference readings and keep them:

```
sudo wdutil info                 # note the healthy RSSI, noise, SNR, Tx rate, PHY mode
networkQuality -v -I en0         # note typical up/down throughput and RPM for this site
```

Do this per site/subnet and, for WFH staff, have each person run the §6.4 script once while everything works. A baseline library turns "is 300 Mbps down bad?" into "it's normally 900 here, so yes." Re-baseline after any infrastructure change.

### 7.2 Tool familiarization drills

Run every command in §2 and §5 on your own machine until the output is familiar — before you need it. For WiFi Explorer Pro 3, spend a session with a healthy network open and:

- Switch through every scan mode (Active, Directed, Passive All/Single Channel) and note what each reveals.
- Open each inspector (Issues, Utilization, Clients, History) on your own network and read the numbers against §3.2.
- Export a scan and a History series to CSV so you know the format before you attach one to a ticket.
- Annotate your own known APs/BSSIDs — then rogues stand out immediately on a real scan (§4.5).

### 7.3 Fault-injection lab

Deliberately break things on a **network and devices you own** and watch what each tool shows. This is the highest-value practice — you learn the *signature* of each fault so you recognize it live. All of these are safe and reversible:

| Induce | How | Observe | Maps to |
|---|---|---|---|
| Weak signal / coverage | Walk far from the AP, or into a shielded spot | RSSI falls, Tx rate drops, retries climb | §4.1, §6.3 |
| Co-channel congestion | Set your router to a busy 2.4 GHz channel | Utilization inspector: high beacon overhead, many networks | §3.4, §4.4 |
| Non-Wi-Fi interference | Run a microwave near the 2.4 GHz AP | SNR drops with RSSI steady; spectrum analyzer spike | §3.2, §3.6 |
| Auth failure | Enter a wrong PSK, or expire a test 802.1X cert | `eapolclient` / airportd logs show the failure point | §4.2, §4.6 |
| DHCP failure | Disable DHCP on the router briefly | Client gets 169.254.x.x; `ipconfig getsummary` confirms | §4.3 |
| DNS failure | Set a bogus static DNS server on the client | ping-by-IP works, `dig` fails; `scutil --dns` shows it | §4.3 |
| No internet, LAN fine | Unplug the router's WAN uplink | Gateway pings, internet path fails | §4.3, §6.1 |
| MTU / fragmentation | Lower router MTU or add a test VPN | Small pings fine, large transfers stall; `ping -D -s` test | §6.5 |

Reset after each. Keep notes on the exact tool output for each fault — that becomes your team's "what does X look like" reference.

### 7.4 Build a reference capture library

Capture (Wireless Diagnostics Sniffer or `tcpdump -I`, on your own network) and archive a labeled set of `.pcap`/`.pcapng` files: a clean association + 4-way handshake, a normal beacon stream, and — from the fault lab — a failed auth and a deauth event. Open them in WiFi Explorer Pro 3 and Wireshark so you've practiced reading them cold. Publicly available sample WLAN captures work too if you can't generate one. Note: generate deauth/attack traffic **only against your own lab gear** — injecting frames against networks you don't own is illegal in most jurisdictions.

### 7.5 Team drills

- **Tabletop:** read out a ticket ("WFH user, calls drop every afternoon, wired is fine") and have staff talk through the isolation path (§6.1) and which tool they'd reach for. No equipment needed; builds the mental flowchart.
- **Timed teardown/rebuild:** at a test AP, have a tech run the §4.7 mini-survey and produce RSSI numbers in under 15 minutes.
- **WFH dry run:** before any real ticket, have remote staff run the §6.4 script on their home setups and submit results. You get baselines *and* surface latent home problems (double-NAT, mesh backhaul) before they cause an outage — and staff learn the procedure while nothing's on fire.
- **New-hire ramp:** the fault lab (§7.3) doubles as an onboarding checklist — a new help desk hire should reproduce and identify each fault signature before taking live WLAN tickets.

### 7.6 Network simulators & impairment tools

Simulators let you reproduce faults on demand without walking into a shielded room. **Read this distinction first:** almost all of them impair the **IP / transport path** (latency, packet loss, jitter, bandwidth caps, reordering) — the symptoms of §4.4 and §6. They do **not** reproduce **RF-layer** faults (RSSI, SNR, retries, co-channel interference), because that happens below IP on the actual radio. For RF, you need physical methods or radio simulation (bottom of this section). Match the tool to the layer you're practicing.

**Path/transport impairment — closest to real WFH symptoms**

| Tool | Platform | Notes |
|---|---|---|
| **Network Link Conditioner** | macOS (native) | Apple's own — install via *Additional Tools for Xcode*. Presets (3G, Edge, Lossy, High Latency) plus custom up/down bandwidth, latency, and packet-loss %. System-wide toggle in System Settings → Developer. First choice for simulating a bad home link on the actual Mac under test. |
| **`dnctl` + `pfctl` (dummynet)** | macOS (native CLI) | What Link Conditioner drives underneath. Scriptable for repeatable drills — build pipes with set delay/bandwidth/loss and attach via pf rules. Good when you want a saved, versioned impairment profile. |
| **`tc` / netem** | Linux | The classic. `tc qdisc add dev eth0 root netem delay 200ms 50ms loss 5% reorder 25%` — precise, composable delay/jitter/loss/corruption/reordering. Run it on a Linux router/bridge the Mac sits behind to impair its traffic transparently. |
| **Toxiproxy** | Cross-platform | Shopify's TCP proxy; inject latency/timeouts/bandwidth limits per-connection via API. Best for app-level resilience testing rather than whole-link. |
| **comcast** | Cross-platform | Friendly Go wrapper over tc/netem/pfctl — one command for common impairment recipes. |
| **Pumba** | Containers | netem for Docker containers; useful if your test harness is containerized. |
| **WANem / Apposite Linktropy** | Appliance/VM | Dedicated WAN emulators (software VM or hardware) for lab-grade, reproducible path impairment across many clients. |

Typical drill: put the Mac behind a Linux bridge running netem, dial in 150 ms delay + 3% loss, and confirm `networkQuality` RPM tanks while throughput looks fine — the classic "calls stutter but speed test is OK" WFH ticket (§6.5).

**Full-topology / protocol simulators — for design and 802.1X practice**

- **GNS3** and **EVE-NG:** run real network OS images (routers, firewalls, controllers) in a virtual topology — practice RADIUS/802.1X flows (§4.6), DHCP scopes, VLANs, and double-NAT (§6.3) end to end.
- **Cisco Packet Tracer:** lighter, education-oriented; good for teaching the topology and DHCP/DNS/VLAN concepts without heavy VMs.
- **Mininet:** rapid virtual L2/L3 topologies with real Linux net stacks; scriptable for automated scenario replay.
- **ns-3:** research-grade discrete-event simulator with a detailed 802.11 model — overkill for help desk drills, but it *can* model RF/PHY behavior if you need protocol-level Wi-Fi simulation.

**Actual RF-layer simulation — the hard part**

No software running on the client can fake a real RSSI/SNR change end to end. Options:

- **Variable RF attenuator** (inline between AP and antenna) or simply **distance and obstructions** — the honest way to drive real signal degradation (ties back to the §7.3 fault lab).
- **Faraday bag / shielded pouch** around a phone or AP to force rapid signal collapse and observe client roaming/deauth behavior.
- **`mac80211_hwsim`** (Linux kernel module): creates virtual 802.11 radios that associate and pass frames with no hardware — you can stand up simulated APs/clients and even model signal parameters for automated Wi-Fi testing. Powerful but Linux-only and involved to set up; more for building an automated test rig than day-to-day drills.
- Injecting management-frame faults (deauth floods) for capture practice: lab gear you own only, per the §7.4 legal note.

For the WFH scenarios that dominate real tickets, **Network Link Conditioner (or netem behind a bridge)** covers the vast majority of what you'll want to rehearse — the RF tools are worth it only when you're specifically drilling coverage/interference diagnosis.

### 7.7 Keep skills fresh

RF environments and macOS releases drift. Re-baseline after major macOS updates (Apple changes Wi-Fi tooling between versions — e.g., the `airport` CLI removal in 14.4), re-run a fault-lab pass quarterly, and refresh annotated AP inventories after any infrastructure change so §4.5 rogue detection stays trustworthy.

---

## 8. Escalation Checklist

Attach to the ticket before escalating:

1. `sudo wdutil info` output (redact as policy requires)
2. `sudo wdutil diagnose` archive
3. WiFi Explorer scan export (CSV) + History inspector export for the affected BSSID
4. `networkQuality -v -I en0` results
5. If RF-related: .pcap from Wireless Diagnostics Sniffer or single-channel capture
6. Floor/location, time window, and affected user count
7. For remote/WFH tickets: the §6.4 collection report, taken while the issue was occurring, and wired-vs-wireless result

---

## References

- WiFi Explorer Pro 3 Help: https://www.intuitibits.com/help/wifiexplorerpro3/
- scandump (sensor active mode): https://github.com/intuitibits/scandump
- WLAN Pi: https://www.wlanpi.com
- spectools: https://www.kismetwireless.net/static/spectools/

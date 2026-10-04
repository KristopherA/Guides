# Network Ops Templates — Tickets, Escalation, Baselines

**Audience:** Help desk, system administrators, and security staff
**Purpose:** Turn individual troubleshooting skill into repeatable team process. Three copy-paste templates make ticket hand-offs consistent, route escalations to the right owner with the right evidence, and give the rest of the toolkit the per-site baselines and annotated AP inventory it assumes exist.

The companion macOS Wi-Fi Troubleshooting Guide teaches an operator how to isolate a fault. This document is the connective tissue that lets a *team* do it the same way every time: a ticket form so nothing gets lost between shifts, an escalation matrix so the fault reaches the right desk, and a baseline register that §7.1 (per-site baselines) and §4.5 (annotated AP inventory for rogue detection) of that guide both depend on but neither templates.

Layer names used throughout (RF / auth / DHCP / DNS / routing / path / VPN / performance) match the Network Triage Decision Tree, so a ticket, an escalation, and a triage session all speak the same vocabulary.

---

## 1. Ticket Documentation Template

**Why/when:** Fill this out on every network or Wi-Fi ticket, start to finish. The point is not paperwork — it is that the next person (a colleague picking up your shift, an escalation engineer, or you in three months) can reconstruct what was observed and what was ruled out without re-interviewing the user. A half-diagnosed ticket handed off without its command output forces the receiver to start over; a completed form makes the hand-off a read, not a redo. It also feeds the escalation matrix (§2) directly — every field an escalation needs is captured here first.

Capture the command outputs verbatim (attach the files; don't paraphrase). The "layer isolated to" field is the single most important line — it is the whole job of triage compressed to one word, and it decides which row of §2 you follow.

### Fill-in template

Copy this block into the ticket:

```
TICKET
  Ticket ID:            
  Date / time opened:   
  Reporter:             
  Location:             [ ] On-site  site/floor: ______   [ ] WFH
  Device + macOS ver:   e.g. MacBook Pro (M3), macOS 15.5
  Interface:            en0 (confirm via networksetup -listallhardwareports)

SYMPTOM
  Description:          
  When it started:      
  Frequency:            [ ] Constant  [ ] Intermittent  [ ] Time-of-day: ____
  Scope:                [ ] This user only  [ ] Multiple users  [ ] Site-wide
  Wired vs wireless:    [ ] Fails on both  [ ] Fixed on wired (→ Wi-Fi/RF)  [ ] Not tested

ISOLATION
  Layer isolated to:    [ ] RF/coverage  [ ] Auth/802.1X  [ ] DHCP  [ ] DNS
                        [ ] Routing/VLAN  [ ] Path/ISP  [ ] VPN  [ ] Performance
  Baseline compared:    [ ] Yes (site: ____)  [ ] No baseline on file

EVIDENCE COLLECTED (attach files; note filenames)
  sudo wdutil info:            
  ipconfig getsummary en0:     
  scutil --dns:                
  networkQuality -v -I en0:    
  Reachability (ping/traceroute/mtr): 
  WiFi Explorer scan/History (CSV):   
  .pcap (if RF/capture):       
  WFH collection report (§6.4):

RESOLUTION
  Root cause:           
  Root cause location:  [ ] In scope (corp infra)  [ ] User environment (home/ISP/RF)
  Resolution / advice given: 
  Escalated to:         [ ] Not escalated  [ ] Team: __________ (see §2)
  Closed by / date:     
```

### Worked example

```
TICKET
  Ticket ID:            NET-XXXX
  Date / time opened:   2026-07-02 14:10
  Reporter:             A. User, Finance
  Location:             [x] WFH
  Device + macOS ver:   MacBook Air (M2), macOS 15.5
  Interface:            en0

SYMPTOM
  Description:          Zoom calls stutter and drop audio every afternoon;
                        speed test "looks fine" to the user.
  When it started:      ~1 week ago, worse after 2pm
  Frequency:            [x] Time-of-day: after ~14:00
  Scope:                [x] This user only
  Wired vs wireless:    [x] Fixed on wired — clean on Ethernet to the router

ISOLATION
  Layer isolated to:    [x] Performance  (+ RF/coverage contributing)
  Baseline compared:    [x] Yes (site: WFH remote-user home baseline, 2026-05)

EVIDENCE COLLECTED
  sudo wdutil info:            RSSI -74 dBm, SNR 18 dB, PHY 11ac 5GHz, Tx 117 Mbps
                               (baseline was -58 dBm / SNR 34 at the desk)
  ipconfig getsummary en0:     valid lease, 192.168.1.42, router 192.168.1.1
  scutil --dns:                ISP resolver, resolves fine
  networkQuality -v -I en0:    down 210 Mbps, up 12 Mbps, RPM 190 (baseline RPM ~950)
  Reachability:                gateway ping clean; 1.1.1.1 clean; internet path fine
  WFH collection report:       wifi-report-20260702-1412.txt attached

RESOLUTION
  Root cause:           Weak Wi-Fi link + afternoon congestion; user moved to a
                        far room after lunch. Good throughput masks low RPM
                        (bufferbloat/latency) — the calls-stutter signature.
  Root cause location:  [x] User environment (home RF)
  Resolution / advice given: Use Ethernet for calls, or move router / add mesh
                        node; switch client to 5 GHz. Diagnosis + data left in
                        ticket; RF is outside corporate control.
  Escalated to:         [x] Not escalated
  Closed by / date:     K. Reyes, 2026-07-02
```

---

## 2. Escalation Matrix

**Why/when:** Use this the moment the ticket form (§1) lands on a layer you don't own. It maps *fault owner* to *responsible team*, tells you *what evidence to attach* so the receiving team can act without bouncing it back, and sets a *target response* so nothing stalls. The most common outcome — RF, home ISP, home Wi-Fi — is "advise the user," not "escalate": those are genuinely outside corporate control, and the deliverable is documented advice plus a ticket note that root cause is in the user environment (see §1 "root cause location").

Match the row to the "layer isolated to" field. If two rows could apply, escalate to the lower layer first (RF before performance, DHCP before DNS) — the triage tree's bottom-up rule holds here too.

| Fault owner | Responsible team | Evidence to attach | Target response |
|---|---|---|---|
| **RF / coverage** (weak RSSI, low SNR, dead zone) | Usually *advise user*; on-site → facilities/WLAN | `wdutil info` RSSI/SNR vs. site baseline; WiFi Explorer History export; walk notes | Same-day advice; on-site survey within SLA |
| **Home ISP / internet path** | *Advise user* — user's ISP | §6.4 report showing clean gateway ping but bad path; `mtr`/traceroute to public target | Advise user to contact ISP; no corp SLA |
| **Home Wi-Fi environment** (router placement, double-NAT, mesh backhaul, neighbor congestion) | *Advise user* | §6.4 report; RF-neighborhood list (`system_profiler SPAirPortDataType`); wired-vs-wireless result | Documented advice; note out-of-scope |
| **Corporate DHCP / DNS / routing / VLAN** | Network / infrastructure team | `ipconfig getsummary en0` (169.254 or wrong subnet); `scutil --dns`; VLAN/SSID mapping; affected user count + site | P2 by default; P1 if site-wide |
| **802.1X / RADIUS / identity** | Identity / NAC team | `log stream 'process == "eapolclient"'` excerpt; `security find-identity -v`; cert/profile state; timestamp + client MAC for RADIUS correlation | P2; escalate with RADIUS reject reason if known |
| **VPN concentrator / MTU** | VPN / remote-access team | `networkQuality` with VPN up vs. down; `scutil --dns` resolver order; `ping -D -s 1472 <corp-host>` MTU result; split- vs full-tunnel | P2; P1 if concentrator-wide |
| **Rogue AP / security incident** (evil twin, deauth flood, weak/open config) | Security IR team — **security incident runbook** | BSSID + vendor not in baseline register (§3); `.pcap` with deauth frames (`wlan.fc.type_subtype == 0x0c`); channel + approximate location | Immediate — treat as active incident, do not sit on ticket |
| **Hardware / AP failure** (AP down, port dead, adapter fault) | Network / field ops | AP absent from scan on expected channel/band; affected coverage area; switch-port/PoE status if known | P2; P1 if it removes site coverage |
| **Video conferencing** (Zoom/Teams choppy, frozen, robotic) | Triage first — often *advise user* (home upload) or network team (blocked UDP / VPN hairpin / QoS) — **video conferencing troubleshooting doc** | Zoom Statistics screenshot (Send/Receive latency/jitter/loss); `networkQuality` **upload**; UDP-vs-TCP-fallback evidence (`lsof`); VPN on/off comparison | Route by owner: upload→advise; blocked UDP/VPN→network/security P2 |

**A complete escalation packet** contains the Wi-Fi guide §8 items — do not escalate without them:

1. `sudo wdutil info` output (redact per policy).
2. `sudo wdutil diagnose` archive.
3. WiFi Explorer scan export (CSV) + History inspector export for the affected BSSID.
4. `networkQuality -v -I en0` results.
5. If RF-related: `.pcap` from Wireless Diagnostics Sniffer or single-channel capture.
6. Floor/location, time window, and affected user count.
7. For remote/WFH tickets: the §6.4 collection report **taken while the issue was occurring**, plus the wired-vs-wireless result.

Attach the completed §1 ticket form alongside the packet — it carries the isolation reasoning that tells the receiving team *why* it landed on their layer.

---

## 3. Baseline Register

**Why/when:** This is a living document, not a one-time capture. You cannot recognize a bad number if you have never recorded a good one — "is 210 Mbps down bad?" is unanswerable until the register says "normally ~900 at this site." The Wi-Fi guide §7.1 assumes per-site baselines exist; §4.5 rogue detection assumes an annotated known-AP inventory exists. Neither is templated there — this section is where both live. Keep it in a shared, versioned location (wiki, repo) so every tech reads from the same reference and updates it when the network changes.

Two parts: **(a)** per-site/subnet healthy baselines, and **(b)** the annotated known-AP inventory that lets a rogue stand out on any scan.

### 3a. Per-site / subnet baseline

One row per site or subnet. Capture on a known-healthy Mac at representative locations with `sudo wdutil info` and `networkQuality -v -I en0`.

| Site / subnet | Key location | Typical RSSI | Typical SNR | networkQuality (down / up / RPM) | DHCP scope | Gateway | DNS servers | Captured (date / macOS) |
|---|---|---|---|---|---|---|---|---|
| _____ | _____ | _____ dBm | _____ dB | ___ / ___ Mbps / ___ RPM | _____ | _____ | _____ | _____ |

**Worked example**

| Site / subnet | Key location | Typical RSSI | Typical SNR | networkQuality (down / up / RPM) | DHCP scope | Gateway | DNS servers | Captured (date / macOS) |
|---|---|---|---|---|---|---|---|---|
| HQ 3F / 10.20.30.0/24 | Open desks, NE corner | -58 dBm | 34 dB | 920 / 410 Mbps / 960 RPM | 10.20.30.50–.250 | 10.20.30.1 | 10.20.0.10, 10.20.0.11 | 2026-06-18 / 15.5 |
| HQ 3F / 10.20.30.0/24 | Conf room "Boardroom" | -66 dBm | 27 dB | 780 / 380 Mbps / 910 RPM | (same) | 10.20.30.1 | (same) | 2026-06-18 / 15.5 |
| Branch A / 10.40.5.0/24 | Reception | -61 dBm | 30 dB | 300 / 90 Mbps / 720 RPM | 10.40.5.20–.200 | 10.40.5.1 | 10.20.0.10, 8.8.8.8 | 2026-05-30 / 15.4 |
| WFH — remote user | Home office desk | -58 dBm | 34 dB | 240 / 22 Mbps / 950 RPM | 192.168.1.0/24 (ISP) | 192.168.1.1 | ISP-assigned | 2026-05-12 / 15.5 |

### 3b. Annotated known-AP inventory

One row per known BSSID (each radio/band of an AP is its own BSSID). This is the allowlist §4.5 checks against: any BSSID broadcasting a corporate SSID that is **not** in this table is a rogue/evil-twin candidate. Mirror it into WiFi Explorer Pro 3 annotations (§3.8) so the highlight is automatic on every scan.

| BSSID | SSID | Band / channel | Location | Vendor (OUI) | Notes |
|---|---|---|---|---|---|
| _____ | _____ | _____ | _____ | _____ | _____ |

**Worked example**

| BSSID | SSID | Band / channel | Location | Vendor (OUI) | Notes |
|---|---|---|---|---|---|
| aa:bb:cc:dd:ee:03 | ExampleCorp-WiFi | 5 GHz / ch 44 (80) | HQ 3F NE | Cisco Meraki | MR46, ceiling, sanctioned |
| aa:bb:cc:dd:ee:04 | ExampleCorp-WiFi | 2.4 GHz / ch 6 (20) | HQ 3F NE | Cisco Meraki | same AP, 2.4 radio |
| aa:bb:cc:dd:ee:05 | ExampleCorp-Guest | 5 GHz / ch 149 (80) | HQ 3F NE | Cisco Meraki | same AP, guest VLAN |
| aa:bb:cc:dd:ee:10 | ExampleCorp-WiFi | 5 GHz / ch 36 (80) | HQ 3F "Boardroom" | Cisco Meraki | MR46, conf room |
| aa:bb:cc:dd:ee:20 | ExampleCorp-WiFi | 5 GHz / ch 40 (80) | Branch A | Cisco Meraki | MR44, reception |

> Anything advertising `ExampleCorp-WiFi` from an OUI that is not Meraki, or on an unexpected channel/location, is a rogue candidate — escalate per §2 (rogue AP row) to the security IR runbook.

### Re-baseline cadence

Baselines drift; a stale register is worse than none because it invites false confidence. Re-capture when any of these happen:

- **After macOS updates** — Apple changes Wi-Fi tooling and driver behavior between releases (e.g., the `airport` CLI removal in 14.4). Re-run `wdutil info` and `networkQuality` on an updated reference Mac.
- **After infrastructure changes** — new/moved/replaced APs, channel-plan changes, DHCP scope or DNS server changes, VLAN re-mapping. Update both 3a and 3b, and re-sync WiFi Explorer annotations so §4.5 rogue detection stays trustworthy.
- **Quarterly** — a standing pass to catch slow drift (neighbor RF changes, ISP renegotiation on WFH links, firmware updates) even when nothing obvious changed.

Stamp every row with its capture date and macOS version so a reader can judge whether a baseline is still current before trusting it.

---

## References

- **macOS Wi-Fi Troubleshooting Guide** — §1 Triage Workflow (layer names); §2.3 command line (`wdutil info`, `ipconfig getsummary`, `scutil --dns`, `networkQuality`); §3.2 signal thresholds; §3.8 annotations; §4.5 rogue/evil-twin detection (depends on §3b here); §6.4 WFH collection script; §7.1 baselines (templated here in §3); §8 escalation checklist (reused in §2).
- **Network Triage Decision Tree** — the layer-isolation flow; the "layer isolated to" field in §1 and the fault-owner rows in §2 mirror its branches.
- **Wireshark Filter Primer** — evidence you attach to a rogue/security escalation (§2), e.g. deauth filter `wlan.fc.type_subtype == 0x0c` and the EAPOL/EAP handshake filters for auth escalations.
- **Network Simulation Lab Guide** — rehearse the fault signatures these templates document before capturing them live on a real ticket.
- **Security incident response runbook** — target of the rogue AP / security incident escalation row (§2).

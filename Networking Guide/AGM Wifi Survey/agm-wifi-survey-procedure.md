# AGM Wireless Survey Procedure

## Overview

Procedure for surveying venue-controlled Wi-Fi at the event venue during
the organization's AGM (~1200 attendees, phones + macOS laptops/iPads, steel/concrete
open floor). Goal is not to fix the network (the organization doesn't control it) but
to hand the venue's network admin fast, specific, timestamped evidence so
they can act during the event. Past complaints trace to three root causes:
DHCP exhaustion, per-AP client limits being hit, and macOS "sticky client"
roaming (staying associated to a distant/weak AP instead of a closer one).

## Quick Facts

| Field         | Value                                                   |
|---------------|----------------------------------------------------------|
| Owner         | IT lead (survey), venue network admin (remediation)      |
| Environment   | Venue-controlled network, one-time event                 |
| Location      | Event venue, single large open hall, ceiling + aisle APs  |
| Attendance    | ~1200, phones + macOS laptops/iPads                       |
| Known issues  | DHCP exhaustion, AP client-limit saturation, macOS roam stickiness |
| Last reviewed | 2026-08-23                                                |

## Software Needed

macOS-native tools cover most of this. A few gaps need extra software.

| Tool                        | Purpose                                      | Native? |
|-----------------------------|-----------------------------------------------|---------|
| WiFi Explorer 3             | AP inventory, RSSI, channel, band, noise, walk survey export | Have it |
| `wdutil info`               | Live association state, RSSI, negotiated PHY rate, roam events | macOS native (needs sudo) |
| `system_profiler SPAirPortDataType` | Point-in-time Wi-Fi/AP snapshot | macOS native |
| `log stream`                | Filter kernel/Wi-Fi roam & auth events in real time | macOS native |
| iPerf3                      | Throughput test against a known-good server | Install (`brew install iperf3`) |
| Wireshark / `tcpdump`       | Capture DHCP DISCOVER/OFFER/NAK sequences to prove exhaustion | Install (`brew install wireshark`) |
| `nmap`/`arp-scan`           | Count live hosts per subnet vs. venue's stated DHCP scope size | Install (`brew install nmap`) |
| Spreadsheet/CSV app         | Consolidate walk-survey + spot-check logs for the venue report | Have it |

One-line examples:

```
sudo wdutil info                          # current BSSID, RSSI, PHY mode, roam count
system_profiler SPAirPortDataType         # snapshot of visible APs/SSIDs
log stream --predicate 'subsystem == "com.apple.airport"' --info   # live roam/assoc events
iperf3 -c <venue-test-server-ip> -t 10    # 10s throughput test to a known host
sudo tcpdump -i en0 port 67 or port 68 -w dhcp_survey.pcap   # capture DHCP handshakes
sudo nmap -sn 10.0.0.0/24                 # count live hosts, compare to DHCP scope size
```

## WiFi Explorer 3 Pro — Step-by-Step

Pro adds signal logging/history and CSV export on top of the base scanner,
which is what the walk survey and spot-checks below depend on. Steps:

1. Launch WiFi Explorer 3, confirm the correct interface (en0) is selected
   in the toolbar — it should show live-updating networks within a few
   seconds.
2. Switch to the **Networks** table view and add columns for: SSID, BSSID,
   Channel, Band, RSSI, Noise, Security. (Column picker is the small icon
   at the top-right of the table header.)
3. Click a specific AP/BSSID row, then open the **Details** panel — this
   shows a live RSSI/noise strip chart for that one AP, useful for holding
   still at a sample point and watching signal stability over ~30-60s.
4. Turn on **logging/recording** (Pro feature — toolbar record button or
   File > Start Recording) before you start walking a grid point or a
   spot-check. This timestamps every scan sample so you can correlate it
   later with a tcpdump capture or a complaint report.
5. Walk to each survey point, pause 15-20s per point so the log captures
   a stable sample (not just a single instant), then move to the next point.
6. Stop recording (File > Stop Recording) at the end of the pass.
7. Export the recording: File > Export > CSV. Name it with location context
   and timestamp, e.g. `baseline_prehall_0800.csv` or
   `spotcheck_rearcorner_1130.csv`.
8. For the pre-event baseline pass, also check the **Channel** view (2.4GHz
   and 5GHz tabs) to visually spot co-channel/adjacent-channel overlap
   across APs before doors open.
9. Repeat steps 4-7 for each rolling spot-check during the event, keeping
   file names consistent so they sort chronologically for the post-event
   consolidation.

(Menu wording may differ slightly by exact Pro version — if a menu item
isn't where listed above, check Help > What's New or the app's own
tooltips; the recording/export workflow itself hasn't moved across recent
releases.)

## Procedure

### 1. Pre-event baseline (before doors open)

- Walk a grid pattern across the hall (aisles, back corners, stage-adjacent
  areas) stopping every ~10-15m. At each stop, log in WiFi Explorer 3:
  SSID, BSSID, channel, band (2.4/5/6GHz), RSSI, noise floor, security type.
  Export to CSV.
- From the CSV, flag: coverage gaps (no AP above -75dBm), co-channel or
  adjacent-channel overlap between APs, and 2.4GHz-only zones (phones will
  crowd these).
- Ask the venue admin for: DHCP scope size (e.g. /24 = 254 addresses),
  lease time, and per-AP client limit. Compare scope size to 1200+ expected
  devices (most attendees carry 2+ devices) — this alone often explains
  exhaustion before the event even starts.
- Run one iperf3 test to a known-good server (venue-provided or a laptop
  wired into their LAN) to establish a baseline throughput/latency number.

### 2. During-event spot checks (rolling)

- Pick 6-8 fixed sample points across the hall (entrance, center, two rear
  corners, stage-adjacent, one aisle). Re-survey each with WiFi Explorer 3
  every 30-60 minutes: same fields as baseline, plus current signal-to-noise.
- Run `sudo tcpdump -i en0 port 67 or port 68` at a sample point during a
  complaint spike. A DISCOVER with no OFFER, or repeated DISCOVERs from the
  same client, is direct evidence of DHCP exhaustion — save the pcap.
- Run `sudo wdutil info` at intervals (e.g. every 30s) while walking away
  from an AP toward a closer one. If RSSI drops below roughly -75dBm before
  the Mac roams to the nearer AP, that's the sticky-client behavior — note
  timestamp, BSSID before/after, and RSSI at each.
- Client-count-per-AP isn't visible from a client-side scan — WiFi Explorer
  only sees what the AP broadcasts, not its association table. This number
  has to come from the venue admin's controller/dashboard. Ask them to pull
  it live for any AP you flag as saturated (complaints clustering nearby).

### 3. Reporting to the venue admin

Keep it short and timestamped — they need to act mid-event, not read a
report afterward. For each issue, give:

- Location (sample point) + timestamp
- BSSID(s) involved, channel, RSSI
- What was observed (no DHCP OFFER / RSSI below threshold without roam /
  complaint volume) and the raw evidence (CSV row or pcap) backing it
- What you're asking them to check (scope size, per-AP client cap, channel
  plan) — don't diagnose their controller config for them, just point at
  the symptom and the location

### 4. Post-event

- Consolidate all CSV exports and pcaps into one folder, one row per
  sample point per timestamp.
- Summarize: coverage gap map, DHCP exhaustion evidence (if any), roam
  stickiness incidents (count + locations), throughput baseline vs.
  degraded readings during peak.
- Keep this doc's troubleshooting table updated with anything new observed.

## Troubleshooting (symptom-first)

### Symptom: Devices show "connected, no internet"
- Likely cause: DHCP scope exhausted
- Check: `sudo tcpdump -i en0 port 67 or port 68` — look for DISCOVER with no OFFER
- Evidence to hand off: pcap + timestamp + location

### Symptom: Slow/unstable connection despite good signal
- Likely cause: AP hit its client-association limit, device landed on a
  farther/less-loaded AP with weaker signal
- Check: WiFi Explorer 3 RSSI vs. distance from nearest AP; ask venue for
  live client count on the BSSID in use
- Evidence to hand off: BSSID + RSSI + venue's own client-count figure

### Symptom: macOS device stays slow while walking past better APs
- Likely cause: sticky roaming — driver hasn't triggered a roam yet
- Check: `sudo wdutil info` logged every 30s while walking; look for RSSI
  well below -75dBm with no BSSID change
- Evidence to hand off: BSSID-before/BSSID-after, RSSI at each, timestamp,
  walking path

### Symptom: Congestion only in one part of the hall
- Likely cause: channel overlap or 2.4GHz-only coverage in that zone
- Check: WiFi Explorer 3 channel view across nearby APs
- Evidence to hand off: channel map for that zone

---

## Hardware (requires separate purchase approval)

Everything above runs on existing MacBooks. These are optional upgrades
that get RF-level visibility a client-side Mac survey can't provide
(you can't see other clients' association/probe/deauth frames, or true
per-AP load, without monitor-mode capture):

| Item                                   | Why                                                                 |
|-----------------------------------------|----------------------------------------------------------------------|
| USB Wi-Fi adapter with monitor mode (e.g. Alfa AWUS036ACH) | macOS doesn't do monitor mode; pair with a Linux laptop/VM running `airodump-ng` to see real per-AP client counts and probe/deauth activity without needing the venue's controller access |
| Spare Linux laptop (or a VM you already have) | Host for `airodump-ng`/`wavemon` alongside the monitor-mode adapter |
| Dedicated iperf3 test host (small travel router or SBC) | A known-good, always-on LAN test point instead of relying on a borrowed venue server |

None of this is required to produce useful, actionable evidence for the
venue admin — the software-only procedure above already covers DHCP,
roaming, and coverage. The hardware just closes the "what's actually
associated to each AP right now" gap that only the venue's own controller
can otherwise answer.

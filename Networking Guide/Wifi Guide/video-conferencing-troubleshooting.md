# Video Conferencing / Real-Time Media Troubleshooting — Zoom (macOS)

**Audience:** Help desk, system administrators, security staff
**Focus:** Zoom, but the concepts apply to Teams, Google Meet, and WebEx (all real-time UDP media).
**Why this exists:** Standard connectivity troubleshooting (throughput, "is it up") misses the faults that break calls. Real-time media fails on **jitter, loss, and upload** long before a speed test looks bad. This is the missing lens.
**Port/bandwidth figures verified against Zoom support documentation (July 2026); Zoom updates IP ranges periodically — see References.**

---

## 1. Why real-time media is different

A file download is elastic: it uses whatever bandwidth is available and doesn't care if a packet arrives 200 ms late — TCP just retransmits. A Zoom call is the opposite. Media is **continuous, low-latency, and loss-sensitive**, sent over **UDP** with no retransmission (a late packet is useless, so it's dropped). That inverts what "good network" means:

- **Bandwidth is rarely the problem.** A call needs only a few Mbps (§3). Speed tests routinely pass while calls fail.
- **Latency, jitter, and loss are the problem.** These barely affect a download but directly determine call quality.
- **The link is bidirectional and often asymmetric.** Your *outbound* video can fail while incoming is perfect.

This is why "I ran a speed test and it's fine, but Zoom is choppy" is a legitimate, common, and *diagnosable* ticket — not user error.

### Real-time quality thresholds

| Metric | Good | Marginal | Broken | Effect when bad |
|---|---|---|---|---|
| **Latency (one-way)** | < 150 ms | 150–250 ms | > 250 ms | Talking over each other, awkward pauses |
| **Jitter** (delay variation) | < 30 ms | 30–50 ms | > 50 ms | Robotic/garbled audio, stutter — the #1 killer |
| **Packet loss** | < 1% | 1–3% | > 3% | Dropouts, frozen video, artifacts |

Note **jitter** as a first-class metric — it's absent from generic connectivity troubleshooting but is usually what's actually wrong. High jitter with low average latency and low loss is the classic "audio sounds like a robot" call.

---

## 2. Start here: Zoom's own in-app Statistics

Before any command line, the fastest and most accurate client-side diagnostic is **built into Zoom**. During or after a call:

**Zoom → Settings (gear) → Statistics → Audio / Video / Screen Sharing tabs.**

It shows, live and **per stream and per direction (Send / Receive)**:

- **Latency** (ms)
- **Jitter** (ms)
- **Packet Loss** — Send and Receive separately

Reading it is the whole diagnosis in one screen:

| What you see | What it means |
|---|---|
| **Send** loss/jitter high, Receive clean | *Your upload* is the problem — home uplink, Wi-Fi, or local congestion. "You're frozen to us, but I see you fine." |
| **Receive** loss/jitter high, Send clean | The other end's upload or the path *to* you — often not your machine at all |
| Both high | Shared path problem: Wi-Fi RF, home router, ISP, or VPN |
| Latency high, jitter/loss low | Distance/routing/VPN hairpin — feels laggy but not choppy |
| All clean but user reports bad video | Look local: CPU/thermal, camera/USB, or the app itself (§7) |

Have the user screenshot this during a bad moment — it turns a vague "Zoom is bad" ticket into a specific, escalatable fact.

---

## 3. Zoom's network profile (what to allow / expect)

Zoom media prefers **UDP** and falls back to **TCP/443** when UDP is blocked — and that fallback is the silent quality killer (§5.3).

**Ports & protocols (Zoom Meetings & Webinars):**

| Protocol | Ports | Purpose |
|---|---|---|
| UDP | **8801–8810** | Primary media (audio/video/screen share) — the preferred path |
| UDP | **3478, 3479** | STUN/TURN NAT traversal |
| TCP | **443, 8801, 8802** | Signaling, and media **fallback** when UDP is blocked |
| TCP/UDP | 80, 443 | General client, web, HTTP/3 |

**Rule of thumb for firewalls/VPNs:** allow **outbound UDP 8801–8810 and 3478/3479** to Zoom's ranges. If only TCP/443 is open, Zoom still connects — but every media packet rides TCP, which retransmits and head-of-line-blocks, producing exactly the lag/stutter users report. Confirm UDP is actually being used (§5.3, and the Wireshark primer's real-time section).

**Bandwidth (Zoom Meetings, recommended):**

| Scenario | Down / Up |
|---|---|
| Audio VoIP only | 60–80 kbps |
| 1:1 720p | 1.2 Mbps up/down |
| 1:1 1080p | 3.8 / 3.0 Mbps (down/up) |
| Group 720p | 2.6 / 1.8 Mbps (down/up) |
| Group 1080p | 3.8 / 3.0 Mbps (down/up) |
| Gallery view (receiving) | 2.0 Mbps (25 tiles), 4.0 Mbps (49 tiles) |
| Screen share | 50–150 kbps |

Two takeaways: the numbers are **small** (bandwidth is rarely the limit), and group/HD calls need **meaningful upload** (1.8–3.0 Mbps) — which is exactly what asymmetric home links skimp on (§5.2).

---

## 4. Quick triage — symptom to cause

| User says | Most likely | Go to |
|---|---|---|
| "I'm frozen/robotic to *them*, but I see everyone fine" | Upload starved or lossy | §5.2 |
| "Audio is robotic/garbled both ways" | Jitter / packet loss | §5.1 |
| "Fine at first, then degrades over minutes" | TCP fallback, or bufferbloat under load | §5.3, §5.5 |
| "Glitches every so often, then recovers" | Wi-Fi roaming or power-save mid-call | §5.6 |
| "Only bad on VPN" | VPN hairpin / full-tunnel media | §5.4 |
| "Bad only when I share my screen" | Upload saturation / bufferbloat | §5.5 |
| "Speed test is great though" | Not a bandwidth problem — jitter/loss/upload | §1, §5.1–5.2 |
| "Everyone in the house is affected" | Shared home congestion / router | §5.5 |

---

## 5. Root causes specific to real-time media

### 5.1 Jitter & packet loss (the core)

Zoom Statistics (§2) shows jitter/loss directly; confirm the path with:

```
ping -c 50 <gateway>          # watch the RTT spread (min vs max) = jitter proxy, and % loss
mtr -o "L S D A M" 1.1.1.1     # (brew install mtr) per-hop loss and latency, continuous
networkQuality -v -I en0      # RPM (responsiveness) low = high latency-under-load = bad for calls
```

Wide RTT spread in `ping` = jitter; steady loss in `mtr` at a specific hop localizes it. On Wi-Fi, correlate with RSSI/SNR (Wi-Fi guide §3.2): poor SNR causes retries that surface as jitter/loss to the app. If the RF is clean and the first hop is the gateway, the problem is the home LAN or beyond.

### 5.2 Upload asymmetry (the "you're frozen" ticket)

Home/broadband links are download-heavy; upload is a fraction. Your **outbound** video and screen share ride that thin upload, so *others* see you freeze while *you* see them fine. Zoom Statistics **Send** loss/jitter confirms it.

```
networkQuality -v             # note the UPLOAD figure specifically, not just download
```

Compare upload against §3: a group 720p call needs ~1.8 Mbps up. If upload is below that or is being shared (someone else uploading, a backup running), that's the cause. Fixes: close other uploaders, wire in, lower send resolution (Zoom → Video → disable HD), or upgrade the plan.

### 5.3 Blocked UDP → TCP/443 fallback (the silent degrade)

If a firewall, VPN, or captive network blocks UDP 8801–8810 / 3478, Zoom connects over TCP/443 and *works* — badly. This is common on locked-down corporate networks and some hotels.

Detect it (from the Mac during a call):

```
sudo lsof -nP -iUDP | grep -i zoom      # are there active UDP flows to Zoom? (good)
sudo lsof -nP -iTCP | grep -i zoom      # only TCP 443 to Zoom, no UDP = fallback (bad)
```

Or capture and check for UDP media vs. TCP-only (Wireshark primer §2.8, real-time filters). Fix: open outbound UDP 8801–8810 and 3478/3479 to Zoom's ranges; on VPN, split-tunnel Zoom out (§5.4).

### 5.4 VPN hairpinning & full-tunnel media

A full-tunnel VPN routes Zoom media to the corporate concentrator and back out — adding latency, a chokepoint, and sometimes forcing TCP. Signature: call quality is fine with VPN off, poor with it on; Statistics shows elevated latency across the board.

- **Best practice:** split-tunnel Zoom/Teams/Meet **out** of the VPN so media goes direct to Zoom. Most orgs do this deliberately; if yours doesn't, that's the escalation.
- Confirm the underlay is healthy first (Wi-Fi guide §6.5) — don't blame the VPN for a bad home link.
- `scutil --dns` and routing (`netstat -rn`) show whether Zoom's traffic is being pulled into the tunnel.

### 5.5 Bufferbloat & shared-link saturation

When the uplink saturates (a big upload, cloud backup, or your own screen share), a fat router buffer fills and *every* packet — including Zoom's — waits behind it. Latency spikes under load even though idle latency and speed tests look fine. This is **bufferbloat**, and `networkQuality`'s **RPM** metric is designed to catch it: high throughput but low RPM = bufferbloated.

```
networkQuality -v             # low RPM under load = bufferbloat
ping <gateway>                # run DURING a large upload; watch RTT climb
```

Fixes: enable SQM/QoS on the router if available, pause background uploads during calls, or wire in. On shared home networks, "everyone's affected" points here or at the router.

### 5.6 Wi-Fi-specific: roaming, power-save, band, airtime

Real-time media is uniquely sensitive to brief Wi-Fi interruptions that a download would shrug off:

- **Mid-call roaming:** moving between APs or mesh nodes drops 100–500 ms of packets = an audible glitch. A user walking around during a call, or a sticky client flapping between two weak APs, produces periodic dropouts. Diagnose per Wi-Fi guide §4.4; watch BSSID change (option-click Wi-Fi menu) during a glitch.
- **Power-save latency:** aggressive Wi-Fi power-save adds latency to real-time streams. Usually managed automatically, but a factor on some clients (Wireshark primer §1.2, `wlan.fc.pwrmgt`).
- **Band & airtime:** 2.4 GHz is congested and slower — steer calls to 5/6 GHz. High channel utilization (WiFi Explorer Utilization inspector) means the call competes for airtime; WMM should prioritize it, but only if markings survive (§5.7).
- **AWDL interference:** AirDrop/AirPlay/Handoff channel-hopping (`awdl0`) causes rhythmic latency spikes (Wi-Fi guide §2.4).

Advise: for important calls on Wi-Fi, stay put (don't roam), use 5 GHz, and prefer Ethernet.

### 5.7 QoS / DSCP / WMM (why priority silently disappears)

Real-time apps mark packets for priority — Zoom uses DSCP (e.g., EF for audio, AF41 for video). On Wi-Fi, **WMM (Wi-Fi Multimedia / 802.11e)** maps those marks to airtime access categories so voice/video jump the queue. The problem: **markings are routinely stripped** —

- Many consumer routers and switches clear DSCP.
- VPN encapsulation usually erases the inner marking.
- Some ISPs bleach DSCP at the edge.

So on a congested link the call may be competing on equal footing with bulk traffic despite being "marked." You can't reliably fix home QoS, but on managed networks: preserve DSCP end to end, ensure WMM is enabled on APs (it's mandatory for Wi-Fi 5/6 anyway), and trust/remark DSCP on switches. Verify markings survive with a capture (Wireshark primer §2.8, `ip.dsfield.dscp`).

---

## 6. Local (non-network) causes — rule these out

Zoom quality also drops for reasons that aren't the network at all. If Zoom Statistics is **clean** but video/audio is still bad, look here:

- **CPU / thermal throttling:** encoding video is CPU-heavy; a hot or loaded Mac drops frames. Check Activity Monitor (CPU tab, and the "Energy"/thermal state). Screen sharing on a dual-core is especially prone (Zoom recommends quad-core).
- **Camera / USB:** external webcams on a shared/underpowered USB hub, or another app holding the camera. Test with the built-in camera.
- **Other apps:** background sync, another conferencing app, or a browser with many media tabs contending for CPU and bandwidth.
- **Zoom itself:** an outdated client; try updating, or the Zoom web client to compare.

This matters because chasing a "network" ghost when the Mac is thermal-throttling wastes everyone's time.

---

## 7. Reproduce it in the lab (practice)

Use the Network Simulation Lab Guide's native impairment tools to build real-time faults on your own Mac, then watch Zoom Statistics react:

| Fault | Tool (Lab guide §4, §5.9) | Expect in Zoom Statistics |
|---|---|---|
| Jitter | Network Link Conditioner / netem `delay 100ms 50ms` | Jitter climbs, audio robotic |
| Loss | netem `loss 5%` or Link Conditioner Lossy | Send/Receive loss up, video artifacts/freeze |
| Upload cap | dummynet pipe `bw 1Mbit/s` on outbound | Send side degrades, "you're frozen" |
| Latency | Link Conditioner 300 ms | Latency high, jitter/loss OK — laggy not choppy |
| Bufferbloat | Saturate upload + `ping` gateway | RTT climbs under load, RPM drops |

Learning each signature in the lab is what lets you read a user's Statistics screenshot instantly.

---

## 8. Escalation specifics

Attach to a video-conferencing ticket (extends the Wi-Fi guide §8 packet and the Network Ops Templates escalation matrix):

1. **Zoom Statistics screenshot** taken during a bad moment (Send/Receive latency, jitter, loss per stream) — the single most useful artifact.
2. `networkQuality -v` (note **upload** and **RPM**, not just download).
3. `ping -c 50 <gateway>` and `mtr` output (jitter/loss localization).
4. Whether UDP media or TCP fallback was in use (`lsof`, §5.3).
5. VPN on/off comparison if VPN is involved.
6. Wired-vs-wireless result.

Route by owner: upload/home link → advise user or ISP; blocked UDP / VPN hairpin → network/security team (with the UDP-vs-TCP evidence); corporate QoS/DSCP → network team; clean-network-but-bad → endpoint/CPU (§6).

---

## References

- Zoom network firewall/proxy settings (ports & IP ranges): https://support.zoom.com/hc/en/article?id=zm_kb&sysparm_article=KB0060548
- Zoom system requirements & bandwidth: https://support.zoom.com/hc/en/article?id=zm_kb&sysparm_article=KB0060748
- Companion docs (this folder): macOS Wi-Fi Troubleshooting Guide (§3.2 thresholds, §4.4 slow/intermittent, §6.5 VPN), Network Simulation Lab Guide (§5.9 impairment), Wireshark Filter Primer (real-time/UDP filters), Network Triage Decision Tree, Network Ops Templates (escalation matrix)

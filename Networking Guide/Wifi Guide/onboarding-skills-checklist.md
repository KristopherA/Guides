# Onboarding Skills Checklist — Network & Wi-Fi Help Desk

**Audience:** New help desk hires (system administration / security track), macOS environment
**Purpose:** A progressive, verifiable competency checklist that takes a new hire from tool install to first live ticket. Each item is an *observable* skill — reproduce a fault, name its signature, fix it — not a topic to "understand." Work the levels in order; the goal of Levels 3–4 is that the hire has personally reproduced and can identify every fault signature *before* it appears on a live ticket.

The companion **macOS Wi-Fi Troubleshooting Guide** §7.5 states the bar directly: "a new help desk hire should reproduce and identify each fault signature before taking live WLAN tickets." This document is that checklist. Sign-off (§10) is the gate to live queue work.

How to use: tick each `- [ ]` only when the hire has *demonstrated* the skill to an assessor or produced the artifact (a note, a CSV, a saved capture). Every reproduce-the-fault item cites its source scenario — do that scenario, don't improvise. Keep the notes: the fault-signature notes a hire takes become part of the team's "what does X look like" reference (Lab Guide §6, Wi-Fi guide §7.3).

---

## Level 0 — Environment setup

Get the toolkit installed and verified before touching any scenario. Nothing below works without these.

- [ ] Install Cisco Packet Tracer (native, Intel + Apple silicon) via a free netacad.com account and sign in successfully — Lab Guide §2
- [ ] Confirm macOS native tools respond: run `sudo wdutil info` and get a full status dump (not `<redacted>`) — Wi-Fi guide §2.3
- [ ] Grant the terminal app Location Services permission so `wdutil`/`system_profiler` show real SSID/BSSID instead of `<redacted>` — Wi-Fi guide §2.3
- [ ] Verify `ipconfig getsummary en0`, `scutil --dns`, `networkQuality -v -I en0`, `dig`, `ping`, `traceroute` all run — Wi-Fi guide §2.3
- [ ] Install Network Link Conditioner (via *Additional Tools for Xcode*) and confirm the toggle appears in System Settings → Developer — Lab Guide §4
- [ ] Install Wireshark and confirm the en0 monitor-mode checkbox is present — Wi-Fi guide §2.5, Wireshark Filter Primer §0
- [ ] Obtain WiFi Explorer Pro 3 access on a Mac and launch it against a live network — Wi-Fi guide §3
- [ ] (Optional) Stand up GNS3: read the Apple-silicon caveat first — on M-series, plan to run the GNS3 *server* remotely on an x86 Linux/cloud host and use the Mac as GUI client only; local x86 emulation is not recommended — Lab Guide §3
- [ ] Confirm which interface is Wi-Fi: `networksetup -listallhardwareports` (usually en0) — Wi-Fi guide §2.3

---

## Level 1 — Native tool fluency on a healthy network

You cannot recognize a bad number if you have never recorded a good one. Run every §2 command on a *known-healthy* network until the output is familiar, and capture a personal baseline you can measure future faults against.

- [ ] Read a full `sudo wdutil info` and correctly point out SSID, BSSID, channel/width, RSSI, noise, Tx rate, PHY mode, security, IP, and DNS — Wi-Fi guide §1, §2.3
- [ ] Option-click the Wi-Fi menu and read live BSSID, channel, RSSI, noise, Tx rate, PHY mode — Wi-Fi guide §2.1
- [ ] Open Wireless Diagnostics and use the Window menu (Info, Logs, Scan, Performance, Sniffer) — not the wizard — Wi-Fi guide §2.2
- [ ] Read `ipconfig getsummary en0` and identify the lease, DHCP server, router, and lease time on a healthy client — Wi-Fi guide §2.3
- [ ] Run `scutil --dns` and state which resolver the client is actually using — Wi-Fi guide §2.3
- [ ] Run `dig example.com` vs `dig @1.1.1.1 example.com` and explain what a difference between them would indicate — Wi-Fi guide §2.3, §4.3
- [ ] Run `networkQuality -v -I en0` and read both throughput *and* responsiveness (RPM); explain why low RPM with fine throughput matters — Wi-Fi guide §2.3, §4.4
- [ ] Explain promiscuous vs. monitor mode on Wi-Fi and state why monitor mode is what WLAN analysis needs — Wi-Fi guide §2.5
- [ ] Identify at least three macOS "gotcha" features that masquerade as network faults (e.g., Private Wi-Fi Address, iCloud Private Relay, AWDL) — Wi-Fi guide §2.4
- [ ] **Establish a personal baseline:** capture `sudo wdutil info` and `networkQuality -v -I en0` on a healthy Mac/network, record RSSI, noise, SNR, Tx rate, PHY, and typical down/up/RPM, and file it in the Baseline Register — Wi-Fi guide §7.1, Network Ops Templates §3a

---

## Level 2 — WiFi Explorer Pro 3 fluency

Spend a session with a healthy network open and exercise every scan mode and inspector, reading the numbers against the §3.2 thresholds. The payoff is that abnormal jumps out later.

- [ ] Switch through and describe what each scan mode reveals: Active, Directed, Passive – All Channels, Passive – Single Channel (note Listener and Sensor exist for remote feeds) — Wi-Fi guide §3.1
- [ ] Open the History inspector on your own BSSID and read RSSI/noise/SNR against the §3.2 thresholds (RSSI ≥ -60 good / < -70 problem; SNR ≥ 25 good / < 20 problem; noise ≤ -90 good / > -85 suspect) — Wi-Fi guide §3.2
- [ ] Correctly diagnose the "good RSSI + poor SNR = interference, not coverage" case from the numbers — Wi-Fi guide §3.2
- [ ] Open the Issues inspector and interpret at least three findings and their actions (e.g., overlapping 2.4 GHz channel, Weak Security, Too Many Networks) — Wi-Fi guide §3.3
- [ ] Open the Utilization inspector and read beacon-overhead ratings (Low <10% / Medium 10–20% / High 20–50% / Very High ≥50%) and channel congestion — Wi-Fi guide §3.4
- [ ] Open the Clients inspector (Passive/Listener/Sensor mode) and list clients on one AP with a Passive – Single Channel scan — Wi-Fi guide §3.5
- [ ] Export a scan and a History series to CSV so the ticket-attachment format is familiar — Wi-Fi guide §3.7, §7.2
- [ ] **Annotate your own known APs/BSSIDs** so rogues stand out on future scans, and mirror the annotations against the Baseline Register's known-AP inventory — Wi-Fi guide §3.8, §7.2, Network Ops Templates §3b

---

## Level 3 — Reproduce & identify each fault in the simulation lab

For each scenario: build the fault, watch the diagnostic output, state the failure signature in your own words, then fix it. Follow the Lab Guide scenario exactly — do not improvise faults not in the source. Keep a note of the exact command output for each; that is the real deliverable (Lab Guide §6).

- [ ] Reproduce a DHCP failure in Packet Tracer and identify the `169.254.x.x` (APIPA) signature — empty `show ip dhcp binding`, no Offer returned — then fix the pool/interface — Lab 5.1
- [ ] Reproduce a DNS misconfiguration and identify the signature (ping by IP works, `nslookup`/`dig` by name fails), then point the client at the correct resolver — Lab 5.2
- [ ] Reproduce a wrong subnet mask / missing gateway fault: local pings work, remote/gateway pings fail, `tracert` dies at hop 1 — then correct mask/gateway — Lab 5.3
- [ ] Reproduce a missing route: router-to-router link pings but far LAN fails, subnet absent from `show ip route`, often asymmetric — then add the route on both sides — Lab 5.4
- [ ] Reproduce a VLAN mismatch / trunk misconfiguration: same-VLAN pings work, cross-VLAN fails; `show vlan brief` / `show interfaces trunk` reveal the gap — then correct VLAN/trunk — Lab 5.5
- [ ] Reproduce an ACL blocking legitimate traffic: one service/subnet fails, `show access-lists` hit counts climb on the deny line, drop is at the router — then reorder/narrow the ACL — Lab 5.6
- [ ] Reproduce a NAT/PAT (and double-NAT) fault: no entries in `show ip nat translations`, private-behind-private signature — then set inside/outside + ACL + overload — Lab 5.7
- [ ] Reproduce an 802.1X / RADIUS auth failure in GNS3: session stuck `UNAUTHORIZED`, RADIUS Access-Reject, and identify whether it died on shared secret, credential/method, or cert trust — then fix — Lab 5.8 *(if GNS3 is unavailable, defer this one item and cover the signature via Level 7 EAPOL analysis)*
- [ ] Reproduce latency/loss/bandwidth starvation on the real link (Network Link Conditioner or `dnctl`/`pfctl`): high latency = low RPM with fine throughput; loss = TCP collapse beyond the loss %; cap = throughput pins at the cap — then remove the profile — Lab 5.9
- [ ] Reproduce an MTU/fragmentation fault: `ping -D -s 1372` succeeds but `ping -D -s 1472` fails — then correct MTU / clamp MSS — Lab 5.10
- [ ] After the impairment scenarios, confirm the teardown (`sudo pfctl -d ; sudo dnctl -q flush`; Network Link Conditioner off) so the lab Mac isn't left throttled — Lab Guide §7

---

## Level 4 — Reproduce & identify each fault in the §7.3 fault-injection lab (real gear)

These are the RF-layer and endpoint faults the IP simulators *cannot* reproduce (Wi-Fi guide §7.6). Break real gear you own, observe the signature in the native tools and WiFi Explorer, and reset after each. This is the highest-value practice for live Wi-Fi tickets.

- [ ] Induce weak signal / coverage (walk far from the AP) and observe RSSI falling, Tx rate dropping, retries climbing — Wi-Fi guide §7.3 (→ §4.1, §6.3)
- [ ] Induce co-channel congestion (set the router to a busy 2.4 GHz channel) and read high beacon overhead / many networks in the Utilization inspector — Wi-Fi guide §7.3 (→ §3.4, §4.4)
- [ ] Induce non-Wi-Fi interference (microwave near a 2.4 GHz AP) and observe SNR dropping while RSSI holds steady — Wi-Fi guide §7.3 (→ §3.2, §3.6)
- [ ] Induce an auth failure (wrong PSK, or expire a test 802.1X cert) and find the failure point in `eapolclient`/airportd logs — Wi-Fi guide §7.3 (→ §4.2, §4.6)
- [ ] Induce a DHCP failure (disable DHCP briefly) and confirm the `169.254.x.x` signature via `ipconfig getsummary en0` on a real Mac — Wi-Fi guide §7.3 (→ §4.3)
- [ ] Induce a DNS failure (bogus static DNS on the client): ping-by-IP works, `dig` fails, `scutil --dns` shows the bad resolver — Wi-Fi guide §7.3 (→ §4.3)
- [ ] Induce "no internet, LAN fine" (unplug the router's WAN uplink): gateway pings, internet path fails — Wi-Fi guide §7.3 (→ §4.3, §6.1)
- [ ] Induce an MTU/fragmentation fault on real gear (lower router MTU or add a test VPN): small pings fine, large transfers stall, `ping -D -s` confirms — Wi-Fi guide §7.3 (→ §6.5)
- [ ] Build a small reference capture library from the fault lab: a clean association + 4-way handshake, a failed auth, and a deauth event (own gear only) — Wi-Fi guide §7.4

---

## Level 5 — Run the triage workflow end-to-end

Tie the tools together into the bottom-up isolation the whole team uses. Run the **Network Triage Decision Tree** on a real or simulated ticket and produce a completed ticket form.

- [ ] Explain the bottom-up rule: don't debug DNS with no IP, don't debug DHCP at RSSI -85 — Wi-Fi guide §1
- [ ] Walk the Network Triage Decision Tree end-to-end on a simulated ticket and land on the correct "layer isolated to" — Network Triage Decision Tree; Wi-Fi guide §1
- [ ] Apply the "connected, no internet" playbook in order (lease → gateway → DNS → captive portal) — Wi-Fi guide §4.3
- [ ] Apply the "slow or intermittent" playbook, distinguishing coverage (sawtooth RSSI) from interference (low SNR) from congestion (beacon overhead) — Wi-Fi guide §4.4
- [ ] Fill out the Ticket Documentation Template completely for one worked case, including the single most important line — "layer isolated to" — and attaching evidence files — Network Ops Templates §1
- [ ] Given the isolated layer, name the correct escalation target and required evidence packet from the escalation matrix (and recognize when the answer is "advise the user," not escalate) — Network Ops Templates §2

---

## Level 6 — WFH / remote diagnosis

Remote changes the problem: you can't be there and you don't own the network. The skill is having the endpoint report on itself, then isolating which of four segments owns the fault.

- [ ] State the four possible owners of a WFH ticket and the order to rule them out: Wi-Fi link → home LAN/router → ISP/path → corporate/VPN — Wi-Fi guide §6.1
- [ ] Name the single most decisive test (plug into the router with Ethernet) and what a pass vs. fail proves — Wi-Fi guide §6.1
- [ ] **Run the §6.4 collection script** on a machine and read its output top to bottom, placing the fault from the report alone — Wi-Fi guide §6.4
- [ ] Given a sample §6.4 report, correctly isolate the fault to the right owner (e.g., clean gateway ping + bad internet path = ISP; bad RSSI/Tx = home Wi-Fi; only corporate apps fail = VPN/corp path) — Wi-Fi guide §6.1, §6.4, §6.5
- [ ] Recognize home-specific gotchas: double NAT (192.168 behind 192.168), mesh Wi-Fi backhaul, powerline/extenders, smart-home 2.4 GHz congestion — Wi-Fi guide §6.3
- [ ] Separate a VPN complaint from the underlay: test the underlay first, and use `scutil --dns` + full- vs split-tunnel reasoning + the `ping -D -s 1472 <corp-host>` MTU test — Wi-Fi guide §6.5
- [ ] **Diagnose a video-call complaint:** open Zoom → Settings → Statistics and read per-stream Send/Receive latency, jitter, and loss; state the real-time thresholds (latency <150 ms, jitter <30 ms, loss <1%) and why a passing speed test doesn't clear a call — Video Conferencing Troubleshooting doc §1, §2
- [ ] Explain UDP media (8801–8810) vs. TCP/443 fallback and check which is in use (`lsof`), and why upload asymmetry produces "you're frozen to us" — Video Conferencing doc §3, §5.2, §5.3

---

## Level 7 — Packet analysis in Wireshark

Open a capture and apply the primer's high-value display filters to read the fault directly off the wire/air. Use captures from your own fault lab (§7.4) or provided samples.

- [ ] Open a monitor-mode capture in Wireshark and confirm it has a radiotap header (management/control frames present) — Wireshark Filter Primer §0, §1
- [ ] Read a healthy DHCP DORA with `dhcp` filters (Discover → Offer → Request → ACK) and recognize "Discover with no Offer" as the 169.254 signature — Wireshark Filter Primer §2.2 (Lab 5.1)
- [ ] Read a WPA2 4-way handshake with `eapol`, and recognize M1/M2 repeating with no M3/M4 as a wrong PSK/PMK — Wireshark Filter Primer §1.3
- [ ] Read a failed 802.1X EAP exchange with `eap` and pinpoint the rejection with `eap.code == 4` (correlate to the RADIUS log) — Wireshark Filter Primer §1.3 (Lab 5.8, Wi-Fi guide §4.6)
- [ ] Quantify path loss with `tcp.analysis.retransmission` / `tcp.analysis.duplicate_ack`, and distinguish `tcp.analysis.zero_window` (endpoint, not link) — Wireshark Filter Primer §2.4 (Lab 5.9)
- [ ] Add `radiotap.dbm_antsignal` and `wlan.fc.retry` columns and correlate weak RSSI with retries — Wireshark Filter Primer §1.2, §1.4
- [ ] Spot the MTU smoking gun `icmp.type == 3 and icmp.code == 4` (fragmentation needed, DF set) — Wireshark Filter Primer §2.5 (Lab 5.10)

---

## Level 8 — Security awareness (recognize & escalate — not full IR)

The bar here is *recognition and escalation*, not incident response. Know the signatures §4.5 raises and know exactly when to hand off to the security team.

- [ ] Recognize a rogue AP / evil twin: organize a WiFi Explorer scan by SSID and flag any BSSID broadcasting a corporate SSID that isn't in the annotated known-AP inventory (check the Vendor/OUI column) — Wi-Fi guide §4.5, Network Ops Templates §3b
- [ ] Recognize weak-security signatures: Open / WEP / WPA-TKIP flagged by the Issues inspector — Wi-Fi guide §4.5, §3.3
- [ ] Recognize a deauth-flood signature: capture on the affected channel and filter `wlan.fc.type_subtype == 0x0c` — a burst from one source to many/broadcast with a repeating reason code = attack; isolated deauths are normal — Wi-Fi guide §4.5, Wireshark Filter Primer §1.2, §3
- [ ] Locate a suspected rogue: Passive – Single Channel + History inspector, walk toward it and watch RSSI rise — Wi-Fi guide §4.5, §4.7
- [ ] State when to invoke the **Wireless Security Incident Response Runbook** and via which escalation-matrix row — and that the response is "escalate immediately, treat as active incident, do not sit on the ticket," *not* to run containment yourself — Network Ops Templates §2 (rogue AP row), Wireless Security Incident Response Runbook §1

---

## 9. (Reserved)

*Levels are numbered 0–8; §10 and §11 follow.*

---

## 10. Sign-off

### 10.1 Practical assessment

Before the hire takes live tickets, they pass a short practical under an assessor:

- [ ] **Unknown injected fault:** given a fault the assessor injects (from the Lab §5 or fault-lab §7.3 set) without telling the hire which, isolate it to the correct layer in **under 15 minutes** and state the signature — Wi-Fi guide §7.5 (timed drill), §7.3
- [ ] **Document it:** produce a completed Ticket Documentation Template for that fault, with the correct "layer isolated to" and attached evidence — Network Ops Templates §1
- [ ] **Escalate it correctly:** name the right escalation target (or "advise user") and the evidence packet the receiving team needs — Network Ops Templates §2
- [ ] **WFH dry run:** run the §6.4 collection script on their own home setup and submit a readable report — Wi-Fi guide §6.4, §7.5
- [ ] **Security recognition:** correctly call a rogue/evil-twin or deauth signature from a sample scan/capture and state the escalation path — Wi-Fi guide §4.5

### 10.2 Supervisor sign-off table

| Competency area | Demonstrated (date) | Assessor |
|---|---|---|
| Level 0 — Environment setup | | |
| Level 1 — Native tool fluency + personal baseline | | |
| Level 2 — WiFi Explorer fluency | | |
| Level 3 — Simulation-lab faults (5.1–5.10) | | |
| Level 4 — Fault-injection lab on real gear (§7.3) | | |
| Level 5 — Triage tree + ticket template end-to-end | | |
| Level 6 — WFH/remote diagnosis | | |
| Level 7 — Packet analysis (Wireshark) | | |
| Level 8 — Security recognition & escalation | | |
| Practical assessment (§10.1) — cleared for live queue | | |

---

## 11. Keep skills fresh

Competency decays as RF environments and macOS releases drift — a stale skill set is like a stale baseline: it invites false confidence. Tie the refresh cadence to the toolkit's own maintenance schedule (Wi-Fi guide §7.6–§7.7, Network Ops Templates §3):

- [ ] **After a major macOS update:** re-baseline (`wdutil info`, `networkQuality`) on a reference Mac — Apple changes Wi-Fi tooling between versions (e.g., the `airport` CLI removal in 14.4) — Wi-Fi guide §7.7, Network Ops Templates §3
- [ ] **Quarterly:** re-run a fault-lab pass (a subset of Levels 3–4) to keep the fault signatures sharp — Wi-Fi guide §7.7
- [ ] **After infrastructure changes:** refresh the annotated AP inventory / WiFi Explorer annotations so §4.5 rogue detection stays trustworthy — Wi-Fi guide §7.7, Network Ops Templates §3b
- [ ] Note: software impairment simulators cover the IP-path/WFH symptom set only; RF-layer signatures still require the physical fault-lab methods (§7.3, §7.6)

---

## References

- **macOS Wi-Fi Troubleshooting Guide** — §1 triage workflow; §2 native tools (`wdutil`, Wireless Diagnostics, `ipconfig getsummary`, `scutil --dns`, `networkQuality`, monitor vs. promiscuous, §2.4 gotchas); §3 WiFi Explorer Pro 3 (scan modes, inspectors, §3.2 thresholds, §3.8 annotations); §4 playbooks (§4.1–§4.7); §6 remote/WFH diagnosis (§6.1 four owners, §6.4 collection script, §6.5 VPN); §7.1 baselines, §7.3 fault-injection lab, §7.4 capture library, §7.5 new-hire ramp / timed drill, §7.6 simulators, §7.7 keeping fresh.
- **Network Simulation Lab Guide** — §2 Packet Tracer setup, §3 GNS3 (Apple-silicon caveat), §4 native impairment; Lab scenarios 5.1 DHCP, 5.2 DNS, 5.3 subnet/gateway, 5.4 routing, 5.5 VLAN, 5.6 ACL, 5.7 NAT, 5.8 802.1X/RADIUS, 5.9 latency/loss, 5.10 MTU; §7 reset/hygiene.
- **Network Triage Decision Tree** — the bottom-up layer-isolation flow run in Level 5.
- **Wireshark Filter Primer** — §1 802.11 filters (deauth `wlan.fc.type_subtype == 0x0c`, `wlan.fc.retry`, EAPOL/EAP handshake), §2 IP/transport filters (DHCP, DNS, TCP health, ICMP/MTU), §1.4 signal columns; used in Level 7 and Level 8.
- **Network Ops Templates** — §1 Ticket Documentation Template, §2 Escalation Matrix, §3 Baseline Register (§3a per-site baselines, §3b annotated AP inventory).
- **Wireless Security Incident Response Runbook** — the escalation target for confirmed wireless incidents; Level 8 covers recognition and hand-off only, not the runbook's response steps.

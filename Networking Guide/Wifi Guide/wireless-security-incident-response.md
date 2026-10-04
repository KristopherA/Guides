# Wireless Security Incident Response Runbook

**Audience:** Help desk, system administrators, and security staff
**Purpose:** What to do *after* a wireless threat is confirmed. The macOS Wi-Fi Troubleshooting Guide §4.5 tells you how to *detect* rogue APs, evil twins, and deauth attacks; this runbook is the response half — containment, evidence, eradication, and recovery once you have a confirmed incident.
**Applies to:** Corporate WLAN infrastructure with macOS clients. Detection procedures are cross-referenced rather than repeated.

---

## 0. How to use this document

Each incident type (§2–§7) follows the same shape: **how it was confirmed** (brief, pointing back to detection), **immediate containment**, **evidence collection**, **eradication**, and **recovery**. Read §1 first on every incident — it decides whether this is even an incident and who to wake up. Read §8 (evidence) and §11 (legal) before you touch anything, because how you act in the first ten minutes determines whether the evidence is admissible and whether you tip off the attacker.

This runbook assumes detection has already happened. If you are still confirming, go to the Wi-Fi guide §4.5 and the Network Triage Decision Tree first.

---

## 1. First 15 minutes (triage)

Before containment, answer three questions in order. Do not skip to eradication — pulling a cable or blocking a BSSID before you know the blast radius destroys evidence and can tip off the attacker.

**1. Is it active right now?**

| Signal | Active | Historical |
|---|---|---|
| Deauth/disassoc frames | Ongoing flood in a live capture (`wlan.fc.type_subtype == 0x0c`, Wireshark primer §1.2) | A burst in an old log, none now |
| Rogue/evil-twin BSSID | Beaconing now (visible in a live WiFi Explorer scan) | Seen once, gone on rescan |
| Credential harvest | Clients currently associated to the impostor | No current associations |

An active incident means someone (or something) is present and possibly watching. A historical one is a hunt, not a fire.

**2. What is the blast radius?** Scope it before you act:

- **Which SSID/BSSID/channel** is affected, and is it your corporate SSID or a nearby network?
- **How many clients** are affected or exposed? (WiFi Explorer Clients inspector shows client↔BSSID associations, §4.5 step 5.)
- **Physical area** — one floor, one building, the parking lot? RSSI gives a rough range even before you walk it (§9).
- **Data at risk** — is this a DoS (availability only) or a credential/data-harvesting attack (confidentiality)?

**3. Who needs to know?** Notify per §10. As a floor: any *active* attack against corporate infrastructure gets the security team immediately; a credential-harvesting evil twin also gets an urgent heads-up to affected users ("do not reconnect until told").

**Golden rules for the first 15 minutes:**

- **Preserve first, act second.** Start a capture (§8) *before* you unplug, block, or reconfigure anything. Frames you don't record now are gone.
- **Don't tip off the attacker.** No broadcast emails naming the attack, no visible changes to the rogue's environment, no probing the attacker's device, until you've decided containment strategy. If the attacker is on-site and mobile, they will leave the moment they sense detection.
- **Don't attack back.** No deauthing their AP, no jamming, no connecting to their honeypot to "investigate." See §11.

---

## 2. Rogue AP on the corporate wired LAN

An unauthorized AP someone plugged into a wired network port — a "convenience" AP, a personal travel router, or a deliberate backdoor. The danger is that it bridges the trusted wired LAN to an attacker-controllable wireless segment, bypassing your perimeter.

**How it was confirmed (§4.5).** A BSSID appears that isn't in your annotated inventory, and — critically — its wired side is reachable on your LAN. Confirm the wired attachment by finding the AP's MAC (or its clients' MACs) in a switch's MAC address table / bridge table, or by seeing corporate-subnet DHCP leases handed to devices behind it. A rogue that only exists in the air (no wired attachment) is an evil twin (§3), not a wired rogue.

**Immediate containment.** The goal is to sever the bridge without losing the trail.

| Step | Action |
|---|---|
| Locate the switchport | Trace the rogue's MAC through the switch MAC-address table to a specific port. Note the port, VLAN, and any CDP/LLDP neighbor. |
| Decide: disable vs. monitor | If active data exfiltration is suspected, disable the port (`shutdown`) to stop the bleed. If you want to observe first, place the port in a monitored/quarantine VLAN instead. |
| Preserve the switch state | Before shutting the port, capture the MAC table entry, port config, and (if available) a SPAN/mirror of that port's traffic (§8). |

**Evidence collection.** Photograph the physical device *in place* before removing it (location, cabling, serial, MAC label). Record the switchport, VLAN, uplink, and timestamps. If you mirrored the port, preserve the .pcap. Capture the rogue's beacon/probe traffic over the air on its channel (Sniffer or `tcpdump -I`, §8) so you have both the wired and wireless sides. Do not power-cycle the device — volatile config and logs may be lost.

**Eradication.** Physically remove the device (chain-of-custody, §8). Keep the switchport disabled until you understand *why* it was there — a malicious rogue is a different incident from an employee's unsanctioned range extender, and both need root-cause follow-up. Sweep for others: the same person or method may have planted more than one.

**Recovery.** Re-enable the port only after applying switchport controls (§10: 802.1X/MAB port auth, port security MAC limits, disabling unused ports). Re-baseline your annotated AP inventory so this device can never again masquerade as "known." If the rogue reached sensitive VLANs, treat any credentials or data on those segments as potentially exposed.

---

## 3. Evil twin / SSID impersonation

An attacker AP broadcasting your corporate SSID (or a look-alike) to lure clients into associating, usually to harvest credentials — especially PEAP/EAP-TTLS username/password or captive-portal logins. Unlike a wired rogue (§2), the evil twin is *not* on your wired LAN; it lives entirely in the air.

**How it was confirmed (§4.5).** Organizing a WiFi Explorer scan by SSID surfaces a BSSID broadcasting your SSID that is not in your annotated inventory, often from a consumer-vendor OUI, sometimes with a different security posture (e.g., your enterprise SSID suddenly offered as Open or PEAP-without-server-validation). Clients may report unexpected certificate-trust prompts (Wi-Fi guide §4.6) — a classic evil-twin tell against PEAP/EAP-TTLS.

**Immediate containment.** You usually cannot lawfully disable the attacker's radio (see §11 — no deauthing, no jamming). Containment is client-side and infrastructure-side:

| Vector | Action |
|---|---|
| Stop new victims | Warn users on the affected floor/SSID: do not connect, do not accept new certificate-trust prompts. Push the message through a channel that doesn't depend on the wireless being attacked. |
| Break the credential-harvest premise | Confirm managed clients have server-certificate validation enforced via MDM (trusted RADIUS server names + CA anchor, §4.6). A correctly configured EAP-TLS or validated-PEAP client will *refuse* the evil twin, because it can't present the real RADIUS cert. |
| Protect the SSID | If your WLAN supports it, use WIPS containment against the confirmed rogue on infrastructure you own; do not free-form transmit against arbitrary devices (§11). |

**Evidence collection.** Capture the impostor's beacons and any probe/association traffic on its channel (§8): its BSSID, advertised RSN/AKM (WiFi Explorer security details or `wlan.rsn.akms.type`, primer §1.3), channel, and RSSI over time. If clients associated to it, record which client MACs (beware MAC randomization) and when. Preserve any credential-harvest artifacts (fake captive-portal page, cert prompt screenshots from users). Timestamp everything.

**Eradication.** The attacker's device is not yours to seize — locate it physically (§9) and, if it's inside your premises, involve physical security / facilities to remove the person and hardware, or hand off to law enforcement per §10/§11. If it's off-premises (parking lot, neighboring unit), you cannot remove it; harden clients instead so the twin is inert.

**Recovery.** Force credential rotation for any user who may have submitted credentials to the impostor (assume the worst if PEAP/TTLS-without-validation was in play). Verify server-cert validation is enforced fleet-wide (§10), which is the durable fix — a validated client is immune to this class of attack regardless of how convincing the twin is. Re-baseline the AP inventory.

---

## 4. Deauthentication / disassociation flood (DoS)

A flood of spoofed 802.11 deauth or disassoc management frames that kicks clients off the network. On networks without management-frame protection, these frames are unauthenticated, so anyone can forge them. This is a denial-of-service attack (availability), and it is often a *precursor* to an evil-twin attack (§3) — knock clients off the real AP so they reconnect to the impostor.

**How it was confirmed (§4.5, primer §3).** A live monitor capture on the affected channel shows a burst of subtype 0x0c (deauth) or 0x0a (disassoc) frames — `wlan.fc.type_subtype == 0x0c or wlan.fc.type_subtype == 0x0a` — from one source to many clients or broadcast, with a repeating reason code (commonly 1 or 7). Add `wlan.sa`, `wlan.da`, and `wlan.fixed.reason_code` as columns (primer §1.2). Isolated, occasional deauths are normal; a sustained flood is the attack.

**Immediate containment.** You cannot filter forged management frames at the client, and you must not jam or counter-transmit (§11). Realistic short-term moves:

| Lever | Effect |
|---|---|
| Enable 802.11w / PMF | The durable fix (see recovery). If your infrastructure supports it and it isn't on, this is the single highest-value change — PMF cryptographically protects deauth/disassoc so forged frames are ignored. Rolling it out mid-incident may be feasible on managed WLANs. |
| Channel change | Moving the affected SSID to another channel can dodge a channel-locked attacker briefly; a determined attacker follows. Buys time, not a fix. |
| Locate and remove the source | Physically find the transmitter (§9) and, if on-premises, have physical security remove it. Often the fastest real end to the DoS. |

**Evidence collection.** Preserve the monitor capture with radiotap (§8): source MAC(s), target MAC(s), reason codes, frame rate over time, channel, and RSSI (which supports the §9 walk-down). Note start time, duration, and which clients/services were disrupted (impact matters for §10 legal/HR and for any prosecution).

**Eradication.** Remove the transmitter physically (§9) — a deauth flood requires a nearby radio actively transmitting, so it ends when the device is gone or powered off. There is nothing to "clean up" on your infrastructure; the frames leave no persistent artifact.

**Recovery.** Enable **802.11w / PMF (Protected Management Frames)** everywhere it's supported — this is real and accurate: PMF protects the management frames that deauth attacks abuse, so a compliant network simply ignores forged deauth/disassoc. WPA3 mandates PMF; on WPA2 enable it as required or optional per client support. Where legacy clients can't do PMF, plan their upgrade. Add deauth-flood alerting to your WIPS/monitoring (§10) so the next one is caught in seconds.

---

## 5. Karma / probe-response attacks and open-network honeypots

A client-side attack. The attacker's device answers *any* probe request — when a laptop or phone probes for a remembered network ("HomeWiFi", "Airport_Free"), the karma AP replies "yes, that's me," and the client auto-associates to an attacker-controlled open network. Open-network honeypots are the passive version: an inviting open SSID ("Free Airport WiFi") that harvests whatever the client sends. The primary risk is to **travelers and remote workers** off the corporate LAN, where you can't see the RF and can't run a walk-test (Wi-Fi guide §6, §4.8 remote limits).

**How it was confirmed (§4.5, primer §1.1).** On-site: a WiFi Explorer scan shows an SSID that mirrors names your clients probe for, or a monitor capture shows one BSSID answering many different probe-request SSIDs (`wlan.fc.type_subtype == 0x04` requests met by 0x05 responses from a single source, primer §1.1). Off-site, confirmation is usually indirect — a user reports auto-joining an unexpected open network, or endpoint telemetry shows an association to an unknown open SSID.

**Immediate containment.** This is mostly client hygiene, and for travelers it's preventive guidance more than live containment:

| Action | Why |
|---|---|
| Have the affected client forget the joined network and disconnect | Stops the current association; macOS: Wi-Fi settings → the network → Forget, or `networksetup -removepreferredwirelessnetwork en0 <SSID>`. |
| Assume traffic on the honeypot was observed | Anything sent in clear over an attacker's open AP is compromised; treat sessions/credentials used during that window as exposed. |
| Advise VPN-always on untrusted Wi-Fi | A corporate always-on VPN makes honeypot capture near-useless — the payload is encrypted end to end. |

**Evidence collection.** On-site, capture the karma AP's probe-response behavior (§8) — the "answers everything" pattern is the signature. Off-site, collect what the endpoint can report: the SSID/BSSID it joined, timestamps, and any endpoint-security logs. You will rarely get RF evidence for a remote incident; document the client-side facts instead.

**Eradication.** On-site, locate and remove the transmitter (§9) as with any rogue radio. Off-site, there is nothing for you to eradicate — the AP isn't on your network and isn't yours; the fix is entirely on the client (below).

**Recovery.** Prune saved open networks from client profiles (a device that never auto-joins open networks can't be karma'd into one). Disable auto-join for open networks via MDM where supported. Enforce always-on VPN for untrusted networks. This scenario is why the travel playbook matters — see the captive-portal / travel guidance in the Wi-Fi guide (§6) for the day-to-day version of these controls.

---

## 6. WPA handshake capture / offline PSK cracking

Against a PSK (pre-shared-key) network, an attacker passively captures the WPA/WPA2 4-way handshake — or forces a reconnection to capture it — then takes it away to brute-force the passphrase offline. Nothing is "broken" on your network at capture time; the compromise happens later, on the attacker's hardware, and is invisible to you. A weak PSK falls in minutes to hours.

**How it was confirmed (§4.5, primer §1.3).** Direct confirmation is hard because passive capture is silent. The tells: an unexpected deauth burst against a PSK SSID (§4) — often used to *force* a reconnection and capture a fresh handshake — followed by a client re-associating. In a monitor capture you'd see the deauth, then the client's EAPOL M1–M4 exchange (`eapol`, primer §1.3). Treat any deauth activity against a PSK network as a probable handshake-capture attempt.

**Immediate containment.** You cannot recall a handshake once captured, and offline cracking happens elsewhere. Containment is about the passphrase's survival time:

| Action | Effect |
|---|---|
| Assume the handshake is captured | If a deauth-then-reassoc pattern was seen on a PSK SSID, treat the PSK as being actively cracked. |
| Plan a PSK rotation | A captured handshake only matters if the passphrase is guessable *and still valid*. Rotating the PSK invalidates the captured material. |
| Enable PMF (§4) | Won't stop passive capture, but blocks the forced-reconnection trick that speeds handshake collection. |

**Evidence collection.** Preserve the monitor capture (§8) showing the deauth source and the subsequent EAPOL exchange, with radiotap timestamps. Record which SSID/BSSID and the AKM (confirm it really is PSK, not 802.1X — `wlan.rsn.akms.type`, primer §1.3). Note that you generally cannot prove a *passive* capture occurred; document the observable precursors.

**Eradication.** Rotate the PSK to a long, high-entropy passphrase (a captured handshake against a strong passphrase is not practically crackable). Push the new PSK to legitimate clients via MDM so you don't create a support flood. There is no attacker device to seize unless they were also transmitting (deauth) on-site (§9).

**Recovery.** The strategic fix is to **stop using PSK for anything that matters.** Migrate to 802.1X/EAP (Wi-Fi guide §4.6) so each user has individual credentials and there is no shared secret to capture and crack — this is the durable answer and the recommendation to carry into the postmortem (§10). Where PSK must remain (IoT, guest), use a strong unique passphrase, enable PMF, and consider WPA3-SAE, whose handshake is resistant to this offline-dictionary attack.

---

## 7. Compromised or misconfigured legitimate AP

Not an attacker's device — one of *your* APs found running a weak or wrong security configuration: Open or WEP where it should be WPA2/3, WPA-TKIP instead of AES/CCMP, default/known admin credentials, PMF disabled, an old firmware with known CVEs, or an SSID accidentally bridged to the wrong VLAN. It's your hardware, so containment is straightforward — but the exposure window may already have been abused.

**How it was confirmed (§4.5).** Sorting a WiFi Explorer scan by Security flags anything Open/WEP/WPA-TKIP; the Issues inspector raises "Weak Security" automatically. An AP in your annotated inventory showing a security posture that doesn't match policy is this incident. Also check for unexpected management-plane exposure (admin interface reachable, default creds).

**Immediate containment.** Because it's your device you have full control:

| Step | Action |
|---|---|
| Assess exposure | How long has it been misconfigured, and what could reach it? An Open corporate SSID or wrong-VLAN bridge is urgent; a single AP on TKIP is lower. |
| Correct or isolate | Fix the config immediately if safe (enforce WPA2/3-AES + PMF, remove Open/WEP/TKIP), or take the AP offline / into a quarantine VLAN if you suspect it was actively abused and want to preserve state. |
| Change credentials | Rotate any admin/management credentials, especially if defaults were in use. |

**Evidence collection.** Before changing anything, capture the current running config, firmware version, admin-access logs, and association logs from the AP/controller. If the AP was compromised (not just misconfigured), preserve its state for forensics before reflashing. Note the exposure window (when the misconfig was introduced vs. detected) — it bounds what could have been captured or accessed.

**Eradication.** Apply the correct hardened config from a known-good template: WPA2/WPA3 with AES/CCMP, PMF enabled, no legacy ciphers, strong unique management credentials, current firmware. If compromise is suspected, reset to factory and reprovision from your config-management source rather than trusting the on-box state.

**Recovery.** Re-scan to confirm the AP now advertises the correct security (§4.5). Audit *all* APs against the hardened baseline — a misconfig is rarely unique. Add a recurring config-drift check (§10) so an AP that falls off policy is caught before a scan does. If the exposure window overlapped sensitive traffic on a weak cipher, treat that data as potentially exposed and rotate affected credentials.

---

## 8. Evidence preservation

Wireless evidence is volatile — RF frames exist only while transmitted, and an attacker's on-site device leaves when detected. How you capture in the first minutes decides whether the evidence is usable.

**Capture correctly.**

- Use a **monitor-mode capture with a radiotap header** (Wireless Diagnostics Sniffer, or `tcpdump -I` on the affected channel). Radiotap carries RSSI (`radiotap.dbm_antsignal`) and channel/frequency, which you need both for analysis and for the §9 walk-down. A normal associated capture has no management/control frames and is useless for this (Wi-Fi guide §2.3, primer §0).
- Capture on the **exact affected channel** (Passive – Single Channel, §9), not hopping — you want 100% dwell on the incident, not a sampled survey.
- Save the raw **.pcap/.pcapng unedited.** Analyze on a copy; never annotate the original. Note that the Sniffer disconnects Wi-Fi while capturing (Wi-Fi guide §2.2).

**Timestamps and integrity.**

- Ensure the capturing Mac's clock is synced (NTP) so timestamps correlate with switch/RADIUS/controller logs.
- Record a **cryptographic hash** of each capture file at collection time (`shasum -a 256 capture.pcapng`) and log it. A later matching hash proves the file is unaltered.
- Keep a written timeline: what was seen, when, on which channel/BSSID, by whom, with what tool version.

**Chain of custody (basics).** For anything that might lead to HR action or prosecution, log every hand-off: who collected each item (capture files, photos, physical devices), when, from where, and who held it since. Store originals read-only. Physical devices go into a labeled evidence bag with the collector, date/time, and location. When in doubt, involve legal early (§10) — evidence handling requirements are theirs to set.

**Don't tip off the attacker.** All collection so far is *passive listening*, which is safe and silent. Do **not** connect to the rogue/honeypot, probe or port-scan the attacker's device, or make visible changes to its RF neighborhood — an on-site attacker who senses detection leaves (taking the evidence and the attribution with them). Coordinate the visible containment step (unplug, remove, announce) as a deliberate decision, not a reflex.

---

## 9. Locating the source physically

A transmitting attacker device — a wired rogue, an evil twin, a deauth flooder, a karma AP — is within RF range, which means you can walk to it. This reuses the walk-down technique from the Wi-Fi guide (§4.5 step 3, §4.7 "Locating an existing AP").

**Method.** In WiFi Explorer Pro 3, set **Passive – Single Channel** on the target's channel and open the **History inspector** to watch its RSSI live. Walk the floor: **RSSI rises as you approach** — roughly +6 dB each time you halve the distance. Move toward rising signal, away from falling. Sweep systematically (grid the floor rather than wandering) and the source converges on a room, then a corner, then a device. A directional antenna sharpens the bearing but is optional; the RSSI gradient alone is enough for most spaces.

**Notes and caveats.**

- Passive mode requires an Intel Mac or a Wi-Fi 6E-capable Apple silicon Mac, and disconnects Wi-Fi while running (Wi-Fi guide §3, §4.5). Plan for the tech to be offline during the hunt.
- RF reflects off walls and metal; expect false peaks. Trust the *trend* over several readings, not a single spike.
- For a wired rogue (§2), the switch MAC-address table often localizes it faster than a walk — trace the port first, then walk only if the port serves a large area.
- **When you find it, do not touch the attacker's device beyond what §11 and §10 allow.** For your own hardware, proceed. For an attacker's device or an unknown one, hand off to physical security / legal — locating is intelligence-gathering, seizure is a decision for others.

---

## 10. Roles and notification

Escalate by *type and severity*, not reflexively. The table is a floor, not a ceiling — when unsure, over-notify security and under-act on your own.

| Team | Involve when | They own |
|---|---|---|
| **Security team** | Any confirmed active attack (rogue §2, evil twin §3, deauth §4, karma §5, handshake capture §6). Immediately for anything active. | Incident command, containment strategy, WIPS action, threat attribution, the postmortem. |
| **Network team** | Wired rogue (§2, switchport action), deauth mitigation/PMF rollout (§4), misconfigured AP (§7), any infrastructure change. | Switch/WLAN controller changes, VLAN/port security, config remediation, monitoring rules. |
| **Physical security / facilities** | An attacker device is on-premises and must be located/removed (§9), or a person is involved. | Escorting/removing persons, seizing on-site hardware, camera footage, access-log correlation. |
| **Legal / HR** | Any employee involvement (insider rogue), potential prosecution, credential/data exposure with disclosure duties, or any evidence that may leave the security team. | Chain-of-custody requirements, disclosure obligations, disciplinary process, law-enforcement liaison. Involve **before** collecting evidence you might act on. |
| **Affected users** | Evil twin (§3) or honeypot (§5) exposing credentials — urgent "do not reconnect / do not enter credentials." | Nothing operationally; they need clear, non-technical, out-of-band instructions. |

Communicate through a channel that doesn't depend on the network under attack, and — until containment strategy is set — keep messaging vague enough not to tip off an on-site attacker (§1, §8).

---

## 11. Legal and authorization caution

Read this before acting. Defensive posture is monitoring, capturing, and remediating *your own* infrastructure and clients. It is **not** attacking back.

- **Jamming is illegal.** RF jammers and intentional interference are prohibited in most jurisdictions (e.g., the FCC bans their use, marketing, and sale outright). Do not use one to stop a deauth attack or evil twin — you'd be committing a more serious offense than the attacker.
- **Don't counter-transmit indiscriminately.** Broadcasting your own deauth frames to knock an attacker's device off "the air" is the same illegal act you're responding to and can hit innocent bystanders' networks. **WIPS containment** against a rogue is a vendor feature you may operate on infrastructure and airspace you control, within policy — but it is not a license to transmit against arbitrary third-party devices. When in doubt, don't transmit; locate physically (§9) and remove.
- **Don't attack back or "investigate" the attacker's systems.** Connecting to their honeypot, port-scanning their device, or accessing their AP's admin interface exceeds authorization and can itself be a computer-misuse offense — even against an attacker.
- **Capture only what you're authorized to.** Passive monitor-mode capture of management/control frames in your own environment is defensive and appropriate (primer §0, Wi-Fi guide §2.3). Decrypting others' data traffic is not (primer §1.5).
- **Seizure and persons are not yours to handle.** Locating a device (§9) is fine; seizing hardware or confronting a person is for physical security and, where relevant, law enforcement (§10).

The single-sentence rule: **listen and remediate on your own gear; escalate everything that touches someone else's device or person.**

---

## 12. Post-incident

Close every incident with a short postmortem and at least one preventive control, or the same attack returns.

**Postmortem (blameless).** Record the timeline (detection → triage → containment → eradication → recovery), how it was detected and how *fast*, what worked, what was slow, and the root cause (not "a rogue AP" but "no switchport authentication let anyone bridge the LAN"). Assign owners and dates to the preventive actions below. Keep it blameless — an insider-rogue postmortem should improve controls, and HR handles the person separately (§10).

**Preventive controls** — most incidents in this runbook map to one or more of these:

| Control | Mitigates | Notes |
|---|---|---|
| **WIPS** (wireless intrusion prevention) | Rogue §2, evil twin §3, deauth §4, karma §5 | Continuous monitoring and alerting; the durable version of the §4.5 manual scan. Configure containment within legal bounds (§11). |
| **802.1X everywhere** | Evil twin §3, PSK handshake capture §6, rogue-client access | Per-user credentials, no shared PSK to capture/crack, server-cert validation defeats credential-harvesting twins. Wi-Fi guide §4.6. |
| **PMF / 802.11w** | Deauth/disassoc flood §4, forced-reconnection handshake capture §6 | Cryptographically protects management frames so forged deauth/disassoc are ignored. Mandatory in WPA3; enable on WPA2 where clients support it. **This is the specific mitigation for §4.** |
| **Rogue-AP detection** | Rogue §2, evil twin §3 | Automated comparison against an annotated known-AP inventory (§4.5); wired-side detection via switch MAC-table correlation. Keep the inventory current (Wi-Fi guide §7.7). |
| **Switchport security** | Wired rogue §2 | 802.1X/MAB port authentication, port-security MAC limits, disable unused ports, DHCP snooping. Stops the "plug an AP into any jack" attack at the port. |
| **Config-drift monitoring** | Misconfigured AP §7 | Recurring check of every AP against a hardened baseline (WPA2/3-AES, PMF on, no WEP/TKIP, no defaults) so drift is caught before an attacker or a scan finds it. |
| **Client hardening / always-on VPN** | Evil twin §3, karma/honeypot §5 | MDM-enforced server-cert validation, no auto-join to open networks, VPN on untrusted Wi-Fi. Protects travelers where you can't see the RF (Wi-Fi guide §6). |

Feed the lessons back into detection: if triage was slow because you couldn't tell active from historical (§1), add the alerting that would have answered it. Re-run the detection drills in the Network Simulation Lab Guide and refresh the annotated AP inventory (Wi-Fi guide §7.7) so §4.5 stays trustworthy.

---

## References

- macOS Wi-Fi Troubleshooting Guide (this folder) — §4.5 (rogue/evil-twin/deauth detection), §4.6 (802.1X), §4.7 (walk-test / locating an AP), §6 (remote/WFH), §7.7 (keeping inventories fresh)
- Wireshark Filter Primer (this folder) — §1.1–§1.3 (802.11 frames, deauth, EAPOL), §3 (attack-detection recipes)
- Network Triage Decision Tree (this folder) — first-response branching
- Network Simulation Lab Guide (this folder) — rehearsing detection and 802.1X/RADIUS flows
- FCC on jammer prohibition: https://www.fcc.gov/general/jammer-enforcement
- IEEE 802.11w-2009 (Protected Management Frames): https://standards.ieee.org/ieee/802.11w/3748/
- Wi-Fi Alliance WPA3 (mandates PMF): https://www.wi-fi.org/discover-wi-fi/security

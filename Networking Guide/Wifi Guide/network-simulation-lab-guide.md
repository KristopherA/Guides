# Network Simulation Lab Guide — GNS3, Cisco Packet Tracer & macOS Native Tools

**Audience:** Help desk, system administrators, and security staff
**Purpose:** Build and practice diagnosing common network faults in a safe, repeatable lab before they happen in production.
**Host OS:** macOS (Apple silicon notes included throughout)

---

## 1. Which Tool for What

| Tool | Best for | Runs on macOS | Realism |
|---|---|---|---|
| **Cisco Packet Tracer** | Learning L2/L3 concepts: DHCP, DNS, VLANs, routing, ACLs, NAT. Fast, native, self-contained. | Native (Intel + Apple silicon) | Simulated — simplified device behavior, not real IOS |
| **GNS3** | Realistic labs running actual network OS images: production-accurate IOS/IOS-XE, RADIUS/802.1X, complex topologies. | GUI native; emulation back-end has caveats (see §3) | High — runs real firmware |
| **macOS native tools** | Impairing a real link (latency/loss/bandwidth), and verifying/diagnosing from the endpoint under test. | Native | Real — operates on your actual Mac's traffic |

Rule of thumb: **learn the concept in Packet Tracer, prove it against real firmware in GNS3, and impair/verify real traffic with the native tools.** For pure help desk skill-building, Packet Tracer plus the native tools cover 80% of value with the least setup.

---

## 2. Cisco Packet Tracer Setup

Packet Tracer is free but requires a (free) Cisco Networking Academy account.

1. Create an account and enroll in the free "Introduction to Packet Tracer" course at netacad.com — this unlocks the download.
2. Download the macOS build from the course resources / netacad download page.
3. Install the `.dmg` — it runs natively on both Intel and Apple silicon; no VM required.
4. Launch and sign in with the same netacad account (it validates on first run; a guest login option exists but nags).

**Interface orientation (5 minutes before your first lab):**

- **Bottom-left device bar:** categories (Routers, Switches, End Devices, Connections). Drag onto the canvas.
- **Connections (lightning-bolt icon):** "Copper Straight-Through" for router↔switch/PC↔switch, "Copper Cross-Over" for like-to-like, "Automatically choose" if unsure.
- **Click a device → CLI tab** for router/switch config; **Desktop tab** on a PC for IP Configuration, Command Prompt, and a Web Browser.
- **Simulation mode** (bottom-right toggle): steps packets hop-by-hop and shows PDU contents — invaluable for *seeing* where a packet dies.
- **Realtime mode:** normal live behavior; use `ping`/`ipconfig` from a PC's Command Prompt.

---

## 3. GNS3 Setup (and the Apple silicon reality)

GNS3 has two parts: the **GUI/client** (runs on the Mac) and a **server/back-end** that runs the emulators (Dynamips for classic IOS, QEMU for IOSv/IOU/ASAv, Docker for appliances).

**The critical caveat:** the recommended back-end, the **GNS3 VM, is an x86-64 virtual machine**, and almost all Cisco images are x86. Apple silicon (M-series) Macs cannot run x86 VMs at native speed. Your realistic options:

| Your Mac | Recommended approach |
|---|---|
| **Intel Mac** | Install GNS3 GUI + GNS3 VM in VMware Fusion (free for personal use) or VirtualBox. Works as documented. |
| **Apple silicon Mac** | Best: run the **GNS3 server remotely** on an x86 Linux box, an old Intel PC, or a cloud instance, and use the Mac only as the GUI client (GNS3 supports remote servers natively). Alternative: run an x86 Linux VM under UTM/QEMU and host the server there — functional but slow. Local x86 emulation on M-series is not recommended for daily use. |

**Install (GUI client):**

1. Create a free account at gns3.com and download the macOS package.
2. The macOS GUI build is **unsigned** — after dragging to Applications, right-click → Open on first launch to bypass Gatekeeper, or approve it in System Settings → Privacy & Security.
3. Install into `/Applications` (paths with non-ASCII characters can crash it).
4. On first launch, approve the `uBridge` root helper (needed to bridge to real interfaces).

**Setup wizard — choose your server:**

- **Remote server** (recommended on Apple silicon): point the GUI at the IP/port of your Linux/cloud GNS3 server.
- **Local GNS3 VM** (Intel): let the wizard import and boot the VM in Fusion/VirtualBox.
- **Local server only:** limited to Dynamips classic-IOS images; no IOSvL2/IOU/ASAv.

**Adding images (appliances):**

- Use the **GNS3 Marketplace** appliance templates (File → Import Appliance, `.gns3a`). You supply the actual firmware image — Cisco images require entitlement/licensing you must obtain legally. Open-source alternatives (**FRRouting**, **VyOS**, **Open vSwitch**, **Alpine/Linux** Docker nodes) need no Cisco license and run well, including on remote Linux servers.
- The built-in **VPCS** (virtual PC) and a Linux Docker guest are enough to be endpoints for most lab scenarios below.

> If GNS3 setup is blocking you, do every §5 scenario labeled "PT" in Packet Tracer first — none require GNS3. Return to GNS3 for the 802.1X/RADIUS and real-firmware scenarios.

---

## 4. macOS Native Tools for the Lab

These operate on your real Mac and its traffic — use them to *impair* a link and to *diagnose* from the endpoint (mirroring what a user's machine reports).

**Path impairment:**

- **Network Link Conditioner** — free via *Additional Tools for Xcode* (developer.apple.com). GUI presets plus custom bandwidth/latency/loss; system-wide toggle in System Settings → Developer. First choice.
- **`dnctl` + `pfctl` (dummynet)** — built in; scriptable, repeatable impairment. Example (create a 5 Mbit pipe with 200 ms delay and 5% loss on traffic to a host):

```
sudo dnctl pipe 1 config bw 5Mbit/s delay 200ms plr 0.05
echo 'dummynet out proto tcp from any to 93.184.216.34 pipe 1' | sudo pfctl -f - -e
# revert:
sudo pfctl -d ; sudo dnctl -q flush
```

> Note: `pfctl -f -` replaces the active pf ruleset for the session. On a lab Mac that's fine; on a machine with existing pf/firewall rules, load your rule inside a dedicated anchor instead, or just use Network Link Conditioner (which manages this for you).

**Endpoint diagnosis (the same commands a real ticket uses):**

```
ipconfig getsummary en0        # DHCP lease, router, DNS
scutil --dns                   # resolver actually in use
ping -c 5 <gateway>            # local reachability
traceroute <dest>              # where the path breaks
dig <name>  /  dig @<server>   # DNS resolution, specific server
arp -a                         # ARP table (duplicate IP / gateway MAC)
networkQuality -v              # throughput + responsiveness (RPM)
```

For Wi-Fi RF-layer faults (RSSI/SNR/interference), see the companion **macOS Wi-Fi Troubleshooting Guide** — simulators here cannot reproduce those; they impair the IP path only.

---

## 5. Lab Scenarios

Each scenario follows the same shape: **Problem → Tool & setup → Reproduce → Troubleshoot → Failure looks like → Success looks like → Fix.** Build the fault deliberately, watch the diagnostic signature, then repair it — that signature is what you'll recognize on a live ticket.

Baseline topology used by several PT scenarios: **PC0 — Switch0 — Router0 — Server0** (PC and Server in different subnets across the router). Build it once and save it as `baseline.pkt`.

---

### 5.1 DHCP failure — client gets no address (PT)

**Problem:** User's machine has a 169.254.x.x (APIPA) self-assigned address; no connectivity.

**Setup (Packet Tracer):**
1. On Router0's LAN interface, configure a DHCP pool, then *break it* one of two ways: (a) shut the DHCP-facing interface, or (b) point the PC at a VLAN/subnet the pool doesn't serve.
2. Set PC0's IP Configuration to **DHCP**.

Working pool config for reference (so you know what "fixed" looks like):

```
Router0(config)# ip dhcp pool LAN
Router0(dhcp-config)# network 192.168.10.0 255.255.255.0
Router0(dhcp-config)# default-router 192.168.10.1
Router0(dhcp-config)# dns-server 192.168.10.1
Router0(config)# interface gig0/0
Router0(config-if)# ip address 192.168.10.1 255.255.255.0
Router0(config-if)# no shutdown
```

**Reproduce:** with the interface shut (`shutdown`) or pool absent, set PC0 to DHCP.

**Troubleshoot (from PC0 → Desktop → Command Prompt):**
```
ipconfig /all        # look at the assigned address
ipconfig /release ; ipconfig /renew
```
On the router: `show ip dhcp binding`, `show ip dhcp pool`, `show ip interface brief`.

**Failure looks like:** PC0 shows `169.254.x.x`, subnet `255.255.0.0`, no default gateway. `show ip dhcp binding` is empty. In Simulation mode, the DHCP Discover leaves the PC and gets no Offer back.

**Success looks like:** PC0 gets `192.168.10.x`, gateway `192.168.10.1`. `show ip dhcp binding` lists the lease. Ping to the gateway succeeds.

**Fix:** `no shutdown` the interface / create the pool / correct the subnet so the client and pool match.

**macOS parallel:** `ipconfig getsummary en0` showing a 169.254 address is the exact same signature on a real Mac (guide §4.3 of the Wi-Fi guide).

---

### 5.2 DNS misconfiguration — "server not found" but IPs work (PT)

**Problem:** Web/app by name fails; connectivity by IP is fine.

**Setup (PT):**
1. On Server0 (Desktop → Services → DNS): turn DNS **On**, add an A record e.g. `www.lab.local → 192.168.20.10`.
2. On PC0, set DNS server to a **wrong** address (e.g. an IP with no DNS service) to induce the fault.

**Reproduce:** From PC0's Web Browser, load `www.lab.local`.

**Troubleshoot (PC0 Command Prompt):**
```
ping 192.168.20.10          # by IP — should work
nslookup www.lab.local      # by name — the real test
ipconfig /all               # confirm which DNS server is set
```

**Failure looks like:** `ping` by IP succeeds; `nslookup` returns "request timed out" / can't resolve; browser says host not found. In Simulation mode the DNS query goes to the wrong server and no response returns.

**Success looks like:** `nslookup` returns `192.168.20.10`; browser loads the page. DNS query/response pair completes in Simulation mode.

**Fix:** point PC0 (or the DHCP pool's `dns-server`) at the correct DNS server address.

**macOS parallel:** `dig www.example.com` fails while `dig @1.1.1.1 www.example.com` works ⇒ configured resolver is the problem; `scutil --dns` shows what's actually set.

---

### 5.3 Wrong subnet mask / default gateway — local works, remote doesn't (PT)

**Problem:** User reaches devices on their own subnet but nothing beyond the router.

**Setup (PT):** On PC0, statically set an IP with a **wrong mask** (e.g. `/24` host on a `/25` network) or a **missing/incorrect default gateway**.

**Reproduce:** Ping a same-subnet host (works) then a remote host/Server0 (fails).

**Troubleshoot (PC0):**
```
ipconfig /all               # verify IP, mask, gateway
ping <same-subnet-host>     # succeeds
ping <default-gateway>      # may fail if mask is wrong
ping <remote-host>          # fails
tracert <remote-host>       # dies at the first hop
```

**Failure looks like:** local pings succeed, gateway or remote pings fail; `tracert` shows no progress past the local segment. Wrong mask makes the host think a remote IP is local and it never sends to the gateway (ARP for an unreachable address).

**Success looks like:** gateway and remote pings succeed; `tracert` shows the gateway as hop 1 then continues.

**Fix:** correct the mask and/or default gateway so off-subnet traffic routes to the gateway.

**macOS parallel:** `ipconfig getsummary en0` + `netstat -rn` (routing table) + `arp -a`; ping gateway vs. remote to bisect exactly as above.

---

### 5.4 Missing route — one segment unreachable (PT / GNS3)

**Problem:** Two subnets can't talk; everything else is fine.

**Setup (PT):** Two routers connected, each with a LAN. Omit a static route (or don't enable a routing protocol) so R1 doesn't know how to reach R2's LAN.

Reference fix config:
```
R1(config)# ip route 192.168.20.0 255.255.255.0 10.0.0.2
R2(config)# ip route 192.168.10.0 255.255.255.0 10.0.0.1
```

**Troubleshoot:** `show ip route` on each router; `traceroute` from an endpoint to see where it dies; ping the far router's near interface vs. its LAN.

**Failure looks like:** ping to the router-to-router link works, ping into the far LAN fails; `show ip route` lacks the far subnet; `traceroute` stops at the router missing the route. Often one direction works and the return path is missing (asymmetric) — a classic "it half-works" clue.

**Success looks like:** both LANs' subnets appear in each `show ip route`; end-to-end ping and traceroute complete.

**Fix:** add the missing static route on both routers, or enable/verify the routing protocol (e.g., OSPF) so routes exchange.

---

### 5.5 VLAN mismatch & trunk misconfiguration (PT)

**Problem:** Devices on the same switch can't reach each other, or a device is on the wrong network entirely.

**Setup (PT):**
1. Create VLAN 10 and VLAN 20 on Switch0. Put PC0's access port in VLAN 10, PC1's in VLAN 20.
2. Induce faults: (a) put PC1's port in the *wrong* VLAN; (b) between two switches, set one trunk end to only allow VLAN 10 and forget VLAN 20; or (c) leave the link as access instead of trunk.

Reference config:
```
Switch0(config)# vlan 10
Switch0(config)# vlan 20
Switch0(config)# interface fa0/1
Switch0(config-if)# switchport mode access
Switch0(config-if)# switchport access vlan 10
Switch0(config)# interface gig0/1
Switch0(config-if)# switchport mode trunk
Switch0(config-if)# switchport trunk allowed vlan 10,20
```

**Troubleshoot:** `show vlan brief` (which port is in which VLAN), `show interfaces trunk` (what's actually trunking and which VLANs are allowed), ping between same- and cross-VLAN hosts.

**Failure looks like:** same-VLAN hosts ping; cross-VLAN or across-a-broken-trunk hosts don't. `show vlan brief` shows a port in an unexpected VLAN; `show interfaces trunk` shows a VLAN missing from "allowed" or an interface not trunking. Inter-VLAN traffic also fails if there's no router/SVI ("router-on-a-stick") — a separate common gap.

**Success looks like:** ports appear in the intended VLANs; trunk allows both VLANs on both ends; intra-VLAN pings work, and inter-VLAN works once a router/SVI is present.

**Fix:** correct the access-port VLAN assignment; add the missing VLAN to the trunk's allowed list on both ends; ensure the link is `mode trunk` where needed.

---

### 5.6 ACL blocking legitimate traffic (PT / GNS3)

**Problem:** A specific service or subnet is unreachable; everything else works — the hallmark of an access list.

**Setup (PT):** Apply an ACL that's too broad or on the wrong interface/direction. Example that accidentally blocks all of a subnet's return traffic:

```
R1(config)# access-list 101 deny ip 192.168.20.0 0.0.0.255 any
R1(config)# access-list 101 permit ip any any
R1(config)# interface gig0/0
R1(config-if)# ip access-group 101 in
```

**Troubleshoot:** `show access-lists` (hit counts climb on the deny line when traffic is dropped), `show ip interface gig0/0` (which ACL is applied, which direction), test connectivity from inside vs. outside the blocked scope. In Simulation mode, watch the PDU be dropped at the router with an ACL note.

**Failure looks like:** targeted subnet/service fails while others succeed; `show access-lists` shows increasing matches on a `deny` entry; the drop is at the router, not the endpoint. Implicit `deny any` at the end of every ACL catches traffic you forgot to permit — a frequent surprise.

**Success looks like:** intended traffic passes, `permit` counters increment, `deny` counters only rise for traffic you actually meant to block.

**Fix:** reorder/narrow the ACL entries, correct the interface or direction (`in` vs `out`), and remember the implicit deny — add explicit permits for required return traffic.

**Security note:** this scenario doubles as firewall-rule practice — the same "which rule matched, in which direction" logic applies to any stateful firewall.

---

### 5.7 NAT / PAT misconfiguration & double NAT (PT / GNS3)

**Problem:** Internal hosts can't reach the internet, or only one can at a time; or a home-style double-NAT causes odd reachability.

**Setup (PT):** Configure PAT (overload) but omit a piece to induce failure — e.g., forget to mark the inside/outside interfaces, or leave out the access list.

Reference working config:
```
R1(config)# interface gig0/0
R1(config-if)# ip nat inside
R1(config)# interface gig0/1
R1(config-if)# ip nat outside
R1(config)# access-list 1 permit 192.168.10.0 0.0.0.255
R1(config)# ip nat inside source list 1 interface gig0/1 overload
```

**Troubleshoot:** `show ip nat translations` (are entries being created?), `show ip nat statistics`, `debug ip nat` in a lab, ping/traceroute from inside to an outside address.

**Failure looks like:** no entries in `show ip nat translations`; inside hosts can't reach outside; `show ip nat statistics` shows misses. If `ip nat inside`/`outside` are missing or swapped, translation never happens.

**Success looks like:** translations appear (inside local ↔ inside global), outside pings/HTTP work from multiple inside hosts simultaneously (PAT distinguishes them by port).

**Double-NAT lab:** chain two NAT routers (R1 behind R2) to reproduce the home ISP-gateway-plus-own-router situation. Signature: a private address behind another private address — on a Mac, `ipconfig getsummary en0` shows a 192.168.x.x whose gateway is itself behind another NAT (traceroute shows two private hops). Ties to the WFH guide §6.3.

**Fix:** set inside/outside correctly, ensure the ACL matches the inside subnet, confirm `overload` for PAT; for double NAT, bridge the ISP gateway or disable NAT on one device.

---

### 5.8 802.1X / RADIUS authentication failure (GNS3)

**Problem:** Enterprise client fails to authenticate to the network — the realistic version of the WFH/enterprise Wi-Fi auth ticket. This one is worth doing in **GNS3** because it needs real RADIUS behavior.

**Setup (GNS3):**
1. Topology: a switch (IOSvL2 or Open vSwitch) as the authenticator, a Linux Docker/VM node running **FreeRADIUS** as the auth server, and a client node (Linux with `wpa_supplicant`).
2. Configure the switch for 802.1X (`dot1x`) on the access port, pointing `radius server` at the FreeRADIUS node.
3. On FreeRADIUS, define the client (switch) shared secret and a test user.

**Reproduce faults, one at a time:**
- Wrong shared secret between switch and RADIUS.
- Expired/absent user credential or wrong EAP method.
- RADIUS server certificate not trusted by the client (EAP-TLS/PEAP).

**Troubleshoot:**
- Switch: `show authentication sessions interface <if>`, `show dot1x all`, `debug radius`, `debug dot1x`.
- RADIUS: `radiusd -X` (FreeRADIUS debug mode) shows the Access-Request arriving and whether it's Accept/Reject and why.
- Client: `wpa_supplicant -d` logs the EAP exchange step by step.

**Failure looks like:** switch session stuck in `UNAUTHORIZED` / `AUTH FAIL`; RADIUS debug shows Access-Reject (bad secret ⇒ no matching client / "Shared secret is incorrect"; bad creds ⇒ reject with reason); client EAP log shows where the handshake dies (cert trust vs. credential vs. method). Maps directly to the failure-mode table in the Wi-Fi guide §4.6.

**Success looks like:** `show authentication sessions` shows the port **AUTHORIZED** with the user; RADIUS logs Access-Accept; the client gets network access.

**Fix:** align the shared secret on both sides, correct the credential/EAP method, and establish server-cert trust — the same three buckets you'd chase on a real 802.1X ticket.

---

### 5.9 Latency, loss & bandwidth starvation (macOS native)

**Problem:** "Everything is slow" / "calls stutter but the speed test is fine" — the dominant WFH complaint. No simulator device needed; impair the real link.

**Setup (macOS):** Use Network Link Conditioner or the `dnctl`/`pfctl` recipe from §4. Start with three separate profiles:
- High latency: 300 ms delay, 0% loss.
- Lossy: 20 ms delay, 5% loss.
- Starved: 2 Mbit/s bandwidth cap.

**Reproduce:** enable one profile, then run a call/stream and the tests below.

**Troubleshoot:**
```
ping -c 20 1.1.1.1          # watch RTT and % packet loss
mtr 1.1.1.1                 # (brew install mtr) continuous loss/latency per hop
networkQuality -v           # throughput AND responsiveness (RPM)
```

**Failure looks like:**
- High latency: throughput can look fine, but `networkQuality` RPM is low and ping RTT is high — interactive apps (calls, RDP, SSH) feel laggy though downloads are OK.
- Loss: ping shows dropped packets; TCP throughput collapses far more than 5% due to retransmits/backoff.
- Bandwidth cap: throughput pins at the cap; large transfers crawl, small requests are fine.

**Success looks like (profile off):** RTT low and stable, 0% loss, RPM high, throughput back to the §7.1-style baseline.

**Teaching point:** this proves *why* a speed test can pass while calls fail — latency and loss, not raw bandwidth, drive real-time quality. That's the insight to bring to WFH tickets.

---

### 5.10 MTU / fragmentation (macOS native + GNS3)

**Problem:** Small requests work, large transfers or specific apps (often over VPN) stall — a classic MTU/MSS mismatch.

**Setup:** Lower the MTU on a GNS3 router interface in the path (e.g., `mtu 1400`) or add a VPN/tunnel; alternatively test against any host across a tunnel.

**Reproduce & troubleshoot (macOS):**
```
# Don't-fragment ping, sweep sizes:
ping -D -s 1472 <host>      # 1472 + 28 = 1500; fails if path MTU < 1500
ping -D -s 1372 <host>      # succeeds if path MTU is ~1400
```

**Failure looks like:** small pings and `ping -D -s 1372` succeed, but `ping -D -s 1472` fails ("message too long" or no reply) — the path can't carry full-size frames and something isn't signaling PMTUD. Symptom for users: web pages start loading then hang, SSH connects then freezes on output.

**Success looks like:** full-size `ping -D -s 1472` succeeds end to end, or the app works once MSS clamping / correct MTU is set.

**Fix:** correct the interface MTU, enable MSS clamping on the tunnel (`ip tcp adjust-mss 1360`), or lower client MTU as a stopgap.

---

## 6. Suggested Practice Progression

1. **Packet Tracer, baseline topology:** run 5.1–5.3 (DHCP, DNS, gateway/mask) — the bread-and-butter of help desk tickets.
2. **Packet Tracer, add a second router/switch:** 5.4–5.7 (routing, VLANs, ACLs, NAT) — infrastructure-side faults.
3. **Native tools on your own Mac:** 5.9–5.10 (impairment, MTU) — the WFH symptom set, and the fastest to set up.
4. **GNS3 when ready:** 5.8 (802.1X/RADIUS) and re-run any L3 scenario against real firmware to confirm the simplified PT behavior matches production.

For each, keep notes on the exact **failure signature** — the specific command output that revealed the fault. That library of "what X looks like" is the real deliverable; it's what lets you recognize the problem in seconds on a live ticket.

---

## 7. Reset & Hygiene

- **Packet Tracer:** save a clean `baseline.pkt`; use *File → Save As* per scenario so you never corrupt the baseline.
- **GNS3:** snapshot a topology before inducing faults (right-click project → snapshot) so you can revert instantly.
- **macOS native:** always tear down impairment when done —
  ```
  sudo pfctl -d ; sudo dnctl -q flush        # dummynet
  ```
  and toggle Network Link Conditioner **off** in System Settings. Leaving dummynet pipes active will silently throttle your real machine.

---

## References

- GNS3 documentation: https://docs.gns3.com/docs
- GNS3 Marketplace appliances: https://gns3.com/marketplace/appliances
- Cisco Packet Tracer (via Networking Academy): https://www.netacad.com
- FreeRADIUS: https://www.freeradius.org
- Companion doc: macOS Wi-Fi Troubleshooting Guide (this folder)

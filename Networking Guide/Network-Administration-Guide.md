# Network Administration & Troubleshooting Guide
### pfSense Router + Ubiquiti Switches — Segmented Network with IKEv2 Split-Tunnel VPN

This guide covers a segmented network built on a main pfSense router (terminating an IKEv2 VPN with split tunneling) and Ubiquiti switches/gateways downstream. It's been cross-checked against official Netgate (pfSense) and Ubiquiti documentation — see **Sources** at the end, and the **Verification Notes** callouts marking anything that couldn't be independently confirmed this pass.

---

## 1. Network Overview

- **pfSense** — main router/firewall. Handles routing between VLANs/segments, NAT, firewall rules, DHCP/DNS for some segments, and terminates the IKEv2 VPN (split tunnel).
- **Ubiquiti switches/gateways** — handle L2 switching, VLAN trunking to access ports, and (on UniFi Gateways/EdgeRouter) additional routing.
- **Segmentation** — traffic is split across VLANs (staff, guest, servers, etc.). Inter-VLAN routing and firewall policy live on pfSense; VLAN tagging/trunking lives on the Ubiquiti switches.
- **IKEv2 VPN (split tunnel)** — remote clients route only internal-subnet traffic through the tunnel; everything else goes out the client's own connection. "No internet at all" and "can't reach an internal resource" are different problems with different causes — see Section 2. Section 3 is a dedicated deep dive on the two highest-payoff troubleshooting areas for this setup: physical layer and VPN MTU/fragmentation.

---

## 2. Determining Where in the Network an Issue Exists

Work outward from the client in layers. Stop at the first layer that fails — that's where the fault is.

**Layer 0 — Physical**
- Check link lights, cable seating, and (for PoE devices) PoE budget on the switch. A device that "disappeared" is often a bad cable, a failed SFP, or a switch port that hit its PoE ceiling — check this before touching any configuration. Full walkthrough in Section 3.1.

**Layer 1 — Client / Endpoint**
- Confirm the device has a valid IP, subnet mask, and gateway for its VLAN: `ipconfig /all` (Windows) or `ip addr` (Linux/Ubiquiti).
- No IP, or an APIPA/link-local address (169.254.x.x) → DHCP or switch-port problem, not routing.

**Layer 2 — Switch Port / VLAN (Ubiquiti)**
- Confirm the port is up, in the correct VLAN/Virtual Network, and not error-disabled. Ping the switch/gateway management IP.
- Check port status in the UniFi Network app (Devices → switch → Ports) or `ip addr` / `ip -s link` on an EdgeRouter or UniFi console shell.
- Client can't reach its own default gateway but link light is up → suspect VLAN (Virtual Network) misconfiguration or a bad trunk between switch and router.

**Layer 3 — Router / Firewall (pfSense)**
- From the client, ping the pfSense interface for that VLAN.
- Succeeds, but nothing beyond it does → check the pfSense routing table (`netstat -rn`) and firewall rules for that interface (Firewall → Rules, or `pfctl -sr`).
- Remember rule evaluation order: Ethernet rules → Outbound NAT → Inbound NAT → automatic/internal rules → user rules (Floating, then interface-group, then interface tabs, top-down/first-match) → automatic VPN rules. A rule higher up can silently shadow one you just added.
- Common cause: a firewall rule blocking the VLAN, or the interface/VLAN not correctly assigned under Interfaces → Assignments.

**Layer 4 — WAN / ISP**
- From pfSense itself: Diagnostics → Ping / Traceroute (GUI) or `ping 8.8.8.8` / `traceroute 8.8.8.8` (CLI).
- pfSense can't reach the internet but LAN clients reach pfSense fine → WAN link, ISP, or WAN firewall rule issue, not internal network.
- Check WAN interface status: Status → Interfaces, or `ifconfig`.

**Layer 5 — DNS**
- Test IP connectivity first (`ping 8.8.8.8`), then name resolution (`drill google.com` / `nslookup google.com`).
- IPs work, names don't → DNS Resolver/Forwarder issue on pfSense (Services → DNS Resolver), not a routing problem.

**Layer 6 — VPN Tunnel (IKEv2, split tunnel)**
- "No internet at all while on VPN": on a correctly configured split tunnel this shouldn't happen. On pfSense, split tunneling is controlled by the **Network List** checkbox under VPN → IPsec → Mobile Clients — if checked, only the mobile Phase 2 "Local Network" subnets are pushed to the client; if unchecked, the client routes *everything* through the tunnel. If a client loses general internet, check whether that box is unchecked, or whether a Phase 2 entry's Local Network is accidentally `0.0.0.0/0`.
- "Can't reach internal resources over VPN": check tunnel status (Status → IPsec, SAD/SPD tabs, or `swanctl --list-conns` / `swanctl --list-sas` — see the CLI note in Section 5.2).
- Tunnel up (SA established) but traffic doesn't pass → check Phase 2 selectors (local/remote networks) match the actual subnets, and check firewall rules on the IPsec interface allowing traffic to the target VLAN.
- Tunnel won't establish → confirm WAN firewall allows UDP 500, UDP 4500, and ESP; confirm Phase 1 parameters (encryption/hash/DH group, PSK or cert) match on both ends.
- **Tunnel establishes, small stuff works, large transfers or specific apps (RDP/SMB) hang** → classic MTU/fragmentation symptom. Full walkthrough in **Section 3.2**.

**Quick decision table**

| Symptom | Likely layer |
|---|---|
| No IP / APIPA address | Client or DHCP (Layer 1/2) |
| Can't ping own gateway | Switch port / VLAN (Layer 2) |
| Reach gateway, nothing else on LAN | pfSense routing/firewall (Layer 3) |
| Reach pfSense LAN, not internet | WAN/ISP (Layer 4) |
| IPs work, names don't resolve | DNS (Layer 5) |
| VPN connects but no internet at all | Split-tunnel misconfiguration (Layer 6) |
| VPN up, internal resources unreachable | Phase 2 selectors / firewall on IPsec interface (Layer 6) |
| VPN works for small traffic, hangs on large/specific protocols | MTU / MSS clamping (Layer 6, Section 3.2) |
| Everything down network-wide | Check power/physical layer and core switch first (Layer 0, Section 3.1) |

---

## 3. Deep Dive: Physical-Layer Checks & VPN MTU/Fragmentation Troubleshooting

These two get their own section because they're the most common source of "intermittent" or "makes no sense" problems on a segmented network with a VPN, and because it's easy to jump straight to firewall rules or DNS and skip past them.

### 3.1 Physical-Layer Checks

Rule this out before touching any configuration — a cabling or hardware problem will make a rule change look like it "didn't work," every time, because the real fault is downstream of the config.

**Link and port status**
- Confirm link lights and negotiated speed/duplex on both ends of a cable. On pfSense: Status → Interfaces (shows negotiated speed/duplex), or `ifconfig` at the CLI.
- On a Ubiquiti switch: per-port link state, speed, and PoE draw are visible in the UniFi Network app (Devices → switch → Ports). Many UniFi switch models also expose port error counters and a cable diagnostics/test option in the port detail panel — check what your specific model supports before assuming a cable is fine just because the link light is on.

**Duplex mismatch** (confirmed against Netgate's troubleshooting docs) — most common on circuits at 100Mbit/s or below, typically because an ISP's media converter or CPE is hard-coded to 100Mbit/s full-duplex while the firewall side is left on Autoselect. The documented symptom: the interface shows **100Mbit/s half-duplex** on Status → Interfaces, along with interface errors, collisions, and low throughput. The actual failure mode is a *mismatch* between forced and auto settings, not "auto" itself — fix by matching speed/duplex explicitly on both ends, or setting both to Autoselect.

**PoE budget** — a switch that has hit its power budget will drop or reboot lower-priority powered devices (APs, phones, cameras) without necessarily showing an obvious link failure anywhere else. Check total PoE draw against budget in the UniFi Network app before treating a dropped device as a config or hardware fault.

**Checksum/offload bugs that masquerade as a physical fault** — Netgate's documentation calls out a specific, easy-to-misdiagnose failure mode: hardware checksum offloading issues (historically tied to Realtek NICs, some Via Rhine and Intel `fxp` chips, and virtualized/VirtIO NICs in some hypervisors) can cause packets to be silently rejected somewhere in the path — OS, NIC, switch, or peer — in a way that looks exactly like a cabling or hardware problem. It can also present as a PPPoE connection that authenticates and brings the interface up but never actually passes traffic. Test: System → Advanced → Networking tab → enable **Disable hardware checksum offload** → try to reproduce the issue. A reboot afterward isn't strictly required but is recommended.

**Stale ARP after a hardware swap** — if a device (modem, firewall, switch) was recently replaced and things are flaky afterward, check `arp -a` before assuming a new physical fault; it may just be a stale ARP entry on a peer device. pfSense supports spoofing the old MAC address onto a new interface as a stopgap during a hardware transition (Interfaces → [interface] → MAC address), but Netgate's own docs describe this as "a long-term solution to a temporary problem" — the real fix is letting ARP caches age out (or clearing them), not leaving a spoofed MAC in place indefinitely.

### 3.2 VPN MTU / Fragmentation — Full Troubleshooting Walkthrough

This is the highest-payoff troubleshooting item for this specific deployment, given the IKEv2 split-tunnel setup. It expands on the Section 2 summary with the complete picture pulled from Netgate's IPsec and throughput troubleshooting documentation.

**Recognizing it:** the tunnel shows established, small traffic (ping, DNS, basic web browsing) works fine, but RDP sessions, SMB file transfers, or other bulk-data protocols hang, stall, or time out partway through.

**Why it happens:** ESP encapsulation adds overhead to every packet passing through the tunnel. If the resulting packet exceeds the path MTU, it needs to fragment, or trigger Path MTU Discovery (an ICMP "fragmentation needed" message sent back to the sender so it knows to shrink packets). Quoting Netgate's documentation directly: **"IPsec does not gracefully handle fragmented packets."** If a device along the path drops that ICMP message instead of relaying it — common with misconfigured firewalls or ISP equipment — the sender never learns to send smaller packets, and larger transfers hang instead of failing cleanly with an error.

**Fix, in order of what to try:**
1. Enable MSS clamping for VPN traffic: System → Advanced → **Firewall & NAT** tab → **VPN Packet Processing** section → enable, set Maximum MSS.
2. Start at **1400** (Netgate's documented default). If problems persist, work downward — **1350, 1300, 1250**, etc. — until it resolves, then stop at the first value that works rather than clamping further than necessary.
3. If using routed IPsec (VTI) instead of policy-based tunnels, set MSS on the assigned VTI interface directly — the global VPN Packet Processing option doesn't apply to VTI interfaces.
4. If the problem shows up on WAN throughput generally, not just over the VPN, the equivalent fix is WAN-side MSS clamping or adjusting the WAN interface MTU (Interfaces → WAN → MTU/MSS fields; default Ethernet MTU is 1500).

**Diagnosing before/after the fix:**
- `tcpdump -ni enc0` — captures traffic for all IPsec tunnels regardless of the underlying physical interface; watch for retransmits or one-sided traffic during a transfer that's stalling.
- `tcpdump -ni <wan_if> host <peer_ip>` — watch the raw handshake if the problem looks like it's happening during negotiation rather than after the tunnel is already up.
- Status → IPsec → SAD/SPD tabs to confirm the tunnel and policies actually match what's expected (relevant to the "disappearing traffic" case below).
- Check CPU load during the stall (Diagnostics → System Activity, or `top -aSH`) — if a core is pegged by an interrupt process for a NIC, or by PF/encryption load, the bottleneck may be hardware rather than MTU. A faster cipher (e.g., AES-GCM with AES-NI) or more capable hardware may be needed in that case, not MSS clamping.

**Related failure modes that look like MTU issues but aren't:**
- **Tunnel establishes but zero traffic passes at all**: check firewall rules on the IPsec interface first (Firewall → Rules → IPsec tab). Netgate's docs flag a specific common mistake — scoping that rule to TCP only, which silently breaks ICMP and DNS across the tunnel. Also check Status → System Logs → Firewall for blocked ESP or UDP 4500 traffic on WAN, and confirm the remote/local subnet definitions match exactly (a documented real-world case: one side defined as `192.0.2.1/24`, the other as `192.0.2.0/24` — the tunnel came up, but traffic never passed until the subnet was corrected).
- **Some hosts work, others don't, over the same tunnel**: usually one of — a missing/incorrect default gateway on the client (device isn't pointed at pfSense, or ignores its configured gateway entirely; some embedded devices like IP cameras and printers are known to do this), an incorrect subnet mask that makes a host think the remote VPN subnet is local (so it tries ARP instead of routing), a host-level firewall (Windows Firewall, iptables) blocking the traffic, or a firewall rule on pfSense itself.
- **VPN endpoint isn't the client's default gateway**: if traffic reaches the far side of the tunnel but replies never come back, confirm the responding host's default gateway is actually pfSense. If it's a different router, replies get routed the wrong way (or straight out to the internet) and never re-enter the tunnel — this is a routing design issue, not something MSS clamping fixes. Running `traceroute` from both sides is the fastest way to confirm; expect to see intermediate hops "disappear" across the tunnel — that's normal.
- **Traffic vanishes and never appears on `enc0` at all**: check for an overlapping route or subnet — e.g., another VPN (WireGuard/OpenVPN) using the same network as an old or still-defined IPsec Phase 2 entry. Per Netgate's docs, the kernel will attempt to route matching traffic into IPsec if the traffic selectors match exactly, even if that IPsec tunnel is currently down. Remove/disable the stale IPsec config and restart the daemon; in rare cases a full reboot is needed to clear the old policy.

---

## 4. pfSense Administration

### 4.1 Access

- GUI: `https://<firewall-ip>` (default `https://192.168.1.1`)
- SSH: `ssh admin@192.168.1.1` (enable under System → Advanced → Admin Access)
- Console: VGA/HDMI, serial, or hypervisor console
- Default login is `admin` / `pfsense` — change immediately. **On pfSense Plus 24.03 and later, the system forces a password change on first login**; this is not enforced on CE.

### 4.2 System Status

| Task | GUI | CLI |
|---|---|---|
| Dashboard/system info | Status → Dashboard | `uname -a`, `uptime`, `top`, `df -h` |

### 4.3 Interfaces & VLANs

- Assign/configure: Interfaces → Assignments; Interfaces → LAN/WAN/OPT
- CLI: `ifconfig` (list interfaces)
- Restart an interface: `ifconfig em0 down` then `ifconfig em0 up` — **never do this remotely on the interface you're connected through**
- Create VLAN (GUI): Interfaces → Assignments → VLAN tab → Add. Set Parent Interface, VLAN Tag Type (C-Tag/0x8100 is standard/default; S-Tag/0x88a8 for QinQ), VLAN Tag (1–4094), optional 802.1p priority → Save → return to the Interface Assignments tab and add the new VLAN as an interface.
- Create VLAN (CLI): `ifconfig vlan10 create` then `ifconfig vlan10 vlan 10 vlandev em1`
- QinQ (double-tagged trunks) is configured separately under Interfaces → Assignments → QinQ tab, not the plain VLAN tab.
- Per-interface MTU/MSS and Speed/Duplex fields live on each interface's own configuration page (Interfaces → [interface]) — see Section 3.1/3.2 for when to touch these.

### 4.4 DHCP

pfSense currently ships **two DHCP backends**: Kea DHCP and ISC DHCP. Per the official pfSense Documentation: "After Kea integration is complete it will become the default DHCP server on a future release of pfSense software and eventually the deprecated ISC DHCP server will be removed. **The exact timing of these changes has not been finalized.**" So there's no fixed version cutover to plan around yet — just confirm which backend this deployment is actually running before assuming either command set applies. The backend is selected under System → Advanced → Networking.

- GUI: Services → DHCP Server; static mappings under Services → DHCP Server → Static Mappings (or the Kea Settings tab if Kea is active)
- View ISC leases (CLI): `cat /var/dhcpd/var/db/dhcpd.leases`
- Restart (ISC backend): `service isc-dhcpd restart` — existing leases remain active, minimal disruption
- Restart (Kea backend): manage via Status → Services in the GUI, or the equivalent `service kea-dhcp4` command — confirm the exact service name against your installed version before scripting it.

### 4.5 DNS

- GUI: Services → DNS Resolver (Unbound, recursive/forwarding, DNSSEC/DoT capable, on by default) / Services → DNS Forwarder (dnsmasq, forward-only, off by default)
- CLI lookups: `drill google.com`, `nslookup google.com`
- Restart: `service unbound restart`

### 4.6 Firewall Rules

- GUI: Firewall → Rules (tabs: Ethernet, WAN, LAN, VLANs, Floating)
- Floating rules apply across multiple interfaces and can filter any direction, but run **after NAT** — on WAN they'll see post-NAT addresses. Most setups never need them outside of what the Traffic Shaper wizard adds automatically.
- CLI: view active ruleset with `pfctl -sr`
- Reload after manual rule changes: `pfctl -f /tmp/rules.debug`
- Use **aliases** (Firewall → Aliases — Host, Network, Port, URL/URL Table types) instead of one-off IP rules for blocklists, country ranges, management hosts, VPN clients. URL Table aliases auto-refresh daily.

### 4.7 NAT & Port Forwarding

- GUI: Firewall → NAT (Port Forward, 1:1 NAT, Outbound NAT, NPt)
- Example port forward: WAN TCP 443 → 192.168.1.10:443
- Precedence: inbound Port Forward beats 1:1; outbound 1:1 beats Outbound NAT. Outbound NAT modes: Automatic / Hybrid / Manual / Disabled.
- Reload: `pfctl -f /tmp/rules.debug`

### 4.8 Security Hardening

- **Remote/GUI access**: Netgate's own guidance is not to expose the GUI to the internet even on a non-default port ("security by obscurity" — no real protection). Prefer VPN (IPsec/OpenVPN/SSH tunnel) for remote admin; if WAN GUI access is unavoidable, restrict it to specific IPs/aliases.
- **Brute-force protection**: `sshguard` runs by default against both GUI and SSH login attempts (default: block after 30 attempts, 120s block with 1.5x escalation, 1800s detection window). If you get locked out yourself: Diagnostics → Tables → sshguard, or `pfctl -T flush -t sshguard` at the shell.
- **MFA / stronger auth**: no native TOTP toggle for the WebGUI in current docs — MFA support comes via an external RADIUS or LDAP authentication server (System → User Manager → Authentication Servers).
- **IDS/IPS**: install Suricata or Snort via System → Package Manager.
- **GeoIP / blocklists**: pfBlockerNG-devel (Firewall → pfBlockerNG → IP) for hostile-country blocking, TOR, bogons.
- **Session timeout**: User Manager settings — default 240 minutes; 0/never is explicitly called out as insecure.
- **Password hashing**: bcrypt is the recommended option; SHA-512 is kept only for compatibility.
- **Logging**: Status → System Logs; configure remote syslog under Status → System Logs → Settings.

### 4.9 Package Management

- GUI: System → Package Manager
- Common packages: pfBlockerNG, Suricata, Snort, OpenVPN Client Export, WireGuard

---

## 5. pfSense VPN Management (IKEv2, Split Tunnel)

### 5.1 Setup / Verification Checklist

1. **Enable IPsec**: VPN → IPsec → Tunnels → check "Enable IPsec"
2. **Phase 1** (VPN → IPsec → Tunnels → Add P1):
   - Key Exchange: **IKEv2**
   - Authentication: **EAP-MSCHAPv2** (simplest, local users), **EAP-RADIUS** (preferred when a RADIUS server exists), or **EAP-TLS** (per-user certificates)
   - My Identifier: Fully Qualified Domain Name (must match the Common Name on the IPsec server certificate); Peer Identifier: Any
   - Encryption Algorithm: Netgate's own worked example ("IPsec Remote Access VPN Example Using IKEv2 with EAP-MSCHAPv2") recommends adding **multiple** algorithm/hash/DH combinations, most-preferred first, to cover a range of clients — its documented "good starting set" is: AES256-GCM/SHA256/DH Group 16, then AES256-GCM/SHA256/DH Group 2, then AES256/SHA256/DH Group 14, then AES256/SHA1/DH Group 14. Note the DH Group 2 entry: it's a compatibility fallback for older clients, and conflicts with the general pfSense guidance elsewhere in the docs to avoid DH groups **1, 2, 5, 22, 23, and 24** as insufficiently secure — only include it if a client actually requires it.
   - DH Group (general rule, outside that specific compatibility list): **14 or higher**
   - MOBIKE: leave **disabled** (the documented default) unless remote clients need to roam between IP addresses while keeping the tunnel active
   - Lifetime: 28800s is the standard starting point
3. **Phase 2** (Show Phase 2 Entries → Add P2):
   - Mode: Tunnel IPv4; Protocol: **ESP**
   - Local Network: your internal subnet(s) — for mobile clients this field is also what gets pushed to clients for split tunneling (the official example also notes this can be set to `0.0.0.0/0` to intentionally tunnel *all* client traffic — the non-split-tunnel case)
   - Encryption algorithms: a documented compatibility set is AES/128, AES128-GCM, AES256-GCM; Hash algorithms: SHA256, SHA384, SHA512 — include only what your actual client mix needs
   - PFS: DH Group 14 (2048-bit) — note: Apple iOS configured manually (not via an exported profile) is not compatible with PFS in Phase 2; use an exported VPN profile for iOS instead of manual setup if PFS is required
   - Lifetime: **3600s** for Phase 2 (shorter than the Phase 1 lifetime — this is the documented example value, not a typo)
   - Note: the *first* child SA actually inherits the Phase 1 DH group — PFS only takes effect on rekeys, so a PFS mismatch can look fine initially and then break later
4. **Firewall**: allow UDP 500, UDP 4500, and ESP on WAN (Firewall → Rules → WAN)
5. **Mobile clients** (VPN → IPsec → Mobile Clients):
   - Enable "IPsec Mobile Client Support"
   - Authentication per above
   - Virtual Address Pool (e.g. 10.10.10.0/24) — must not overlap any existing subnet
   - DNS servers — point at internal resolvers so internal hostnames resolve
   - **Network List checkbox — this is the actual split-tunnel switch.** Checked = only the mobile Phase 2 Local Networks are routed into the tunnel (true split tunnel). Unchecked = client sends *all* traffic, including internet, through the tunnel. Netgate notes some clients (notably Windows in certain IKEv2 configurations) don't fully respect this and may need routes added client-side.
6. **Users**: System → User Manager → add VPN users (or configure the RADIUS server if using EAP-RADIUS)
7. **Certificates**: System → Cert Manager → export CA and user certificates
8. **Restart IPsec**: Status → Services (GUI), or restart the strongSwan daemon from the shell — existing tunnels renegotiate automatically, minimal interruption

### 5.2 IPsec Verification Note — CLI Commands

Netgate's current documentation for pfSense uses **strongSwan managed via `swanctl`** — not the older `ipsec` command wrapper. Documented commands:

```
swanctl --list-conns              # list configured connections
swanctl --list-sas                # list active security associations
swanctl --initiate --ike conX     # bring up a Phase 1
swanctl --initiate --child conX   # bring up a Phase 2 / child SA
swanctl --terminate --ike conX    # tear down a connection
```

`ipsec statusall` and `ipsec status` are commonly cited in older guides and third-party cheat sheets (including the source cheat sheet this guide was originally built from), but they did not turn up anywhere in current Netgate documentation during this review — treat them as unverified/possibly legacy for current pfSense versions rather than a confirmed working command. When in doubt, use the Status → IPsec GUI page (Overview / SAD / SPD tabs), which is confirmed current.

### 5.3 MTU / Fragmentation — Quick Reference

See **Section 3.2** for the full walkthrough (cause, fix progression, diagnosis commands, and related failure modes that mimic this symptom but have different fixes). Short version: enable MSS clamping under System → Advanced → Firewall & NAT → VPN Packet Processing, starting at 1400 and working downward if needed; diagnose with `tcpdump -ni enc0`.

### 5.4 Other pfSense VPN Options

OpenVPN (VPN → OpenVPN, Client Export package for easy client profiles), WireGuard (VPN → WireGuard — tunnel + peers + allowed IPs).

---

## 6. pfSense Troubleshooting & Diagnostics

| Task | GUI | CLI |
|---|---|---|
| Ping | Diagnostics → Ping | `ping 8.8.8.8` |
| Traceroute | Diagnostics → Traceroute | `traceroute 8.8.8.8` |
| Packet capture | Diagnostics → Packet Capture | `tcpdump -i em0` (or `enc0` for IPsec — see Section 3.2) |
| Routing table | — | `netstat -rn` |
| ARP table | — | `arp -a` |
| VPN status | Status → IPsec (Overview/SAD/SPD) | `swanctl --list-sas` (see Section 5.2 note) |
| CPU/load during a stall | Diagnostics → System Activity | `top -aSH` |
| Live logs | Status → System Logs | `tail -f /var/log/system.log` |
| Historical logs | Status → System Logs | `clog /var/log/system.log`, `clog /var/log/filter.log` |
| Locked out (sshguard) | Diagnostics → Tables → sshguard | `pfctl -T flush -t sshguard` |

**Console access** (serial): 115200 8N1, no flow control. Windows: PuTTY over the COM port with those settings. Linux: `dmesg | grep tty` to find the adapter, then `sudo screen /dev/ttyUSB0 115200` (exit with `Ctrl+A`, `K`, `Y`).

**Console menu reference** (confirmed against the current pfSense Documentation — numbering can vary slightly by version/platform, but this is the current layout):

| # | Option | Notes |
|---|---|---|
| 0 | Logout | SSH sessions only |
| 1 | Assign Interfaces | Restarts interface assignment; can create VLANs, reassign, or add interfaces |
| 2 | Set interface(s) IP address | Also offers to enable/disable DHCP on that interface, switch GUI to HTTP if HTTPS is broken, and re-enable the LAN anti-lockout rule if needed |
| 3 | Reset admin account and password | Also fixes a broken remote auth source (RADIUS/LDAP) or a disabled/removed admin account. Newer versions prompt for a custom password rather than resetting to the old default |
| 4 | Reset to factory defaults | Also removes installed packages. GUI equivalent: Diagnostics → Factory Defaults |
| 5 | Reboot system | GUI equivalent: Diagnostics → Reboot |
| 6 | Halt system | Clean shutdown/power-off. Never just cut power — always halt first |
| 7 | Ping host | Basic 3-packet ICMP test; uses `ping` or `ping6` automatically |
| 8 | Shell | Full `tcsh`/`sh` shell — powerful, and correspondingly easy to break something |
| 9 | pfTop | Real-time firewall state/bandwidth view |
| 10 | Filter Logs | Raw real-time firewall log view |
| 11 | Restart GUI | Restarts nginx; try option 16 next if this doesn't restore access |
| 12 | PHP shell + pfSense tools | |
| 13 | Update/Upgrade from console | Runs the equivalent of `pfSense-upgrade` — see Section 7.4 |
| 14 | Enable/Disable Secure Shell (sshd) | Toggles SSH |
| 15 | Restore recent configuration | Pulls from local Config History |
| 16 | Restart PHP-FPM | Use after option 11 if the GUI still isn't responding |

On **pfSense Plus 24.03 and later**, the first console/SSH connection after install or a factory reset forces a password change before the menu appears — this can also be done via the GUI Setup Wizard first, in which case pressing Ctrl-C at the console prompt lets it detect the change and proceed to the menu.

---

## 7. Common pfSense Operational Tasks

### 7.1 Backup & Restore

- GUI: Diagnostics → Backup & Restore. Two tabs: **Backup/Restore** (manual download of `config-<hostname>-<timestamp>.xml`, with options for Skip RRD Data, Include Extra Data, AES-256 encryption) and **Config History** (automatic local history, default 30 prior configs + current, with a diff/compare view between any two).
- CLI backup: `cp /cf/conf/config.xml /root/backup-config.xml`
- **Auto Config Backup (ACB)**: Netgate provides this free for both CE and Plus — it's not a paid-tier feature. It stores up to 100 encrypted backups per device on Netgate's servers. (ZFS Boot Environments — full OS-level rollback, not just config — *is* Plus-only.)
- Restore: upload a `config.xml`, or use Config History / console option 15.

### 7.2 Service Restart Reference

| Service | Command |
|---|---|
| Web GUI | `service nginx restart` |
| DNS | `service unbound restart` |
| DHCP (ISC backend) | `service isc-dhcpd restart` |
| IPsec | restart the strongSwan daemon via Status → Services, or the shell |
| OpenVPN | `service openvpn restart` |
| Packet filter reload | `pfctl -f /tmp/rules.debug` |

Prefer restarting a single affected service over broad restarts.

### 7.3 High-Risk Commands — Avoid Running Remotely

`reboot`, `service netif restart`, `ifconfig em0 down`, factory reset. These can cut off your only path back into the device — run them from console/out-of-band access only, or when you have a confirmed second path in.

### 7.4 Firmware/Version Upgrades

Current GUI path is **System → Update** (older references to "System → Firmware" are outdated terminology).

1. Read the release notes / version-specific upgrade notes first
2. Back up config (Diagnostics → Backup & Restore, "All") — ACB is a good redundant backup, not a substitute for a manual one before a major upgrade
3. Remove installed packages before upgrading, and don't let packages get ahead of the OS version
4. On low-memory or virtualized hardware, check the minimum requirements for your target version first
5. Run the upgrade via GUI (System → Update) or CLI: `pfSense-upgrade` (run as root from the shell or console option 13; wrap in `screen` if your SSH session is unstable)
6. System reboots automatically

Branch selection lives under System → Update → Update Settings: Latest Stable (default/recommended), Plus Upgrade path (CE→Plus, registered systems only), Previous Stable, or Development Snapshots (explicitly not for production).

### 7.5 High Availability (CARP) — If This Site Has a Failover Pair

- **CARP** (Firewall → Virtual IPs) provides IP redundancy between two nodes — never manage the GUI/SSH via the CARP VIP itself, always the node's real IP.
- **Config sync** (System → High Availability, XMLRPC-based) — enable only on the primary node; both nodes need identical interface assignments and matching pfSense versions.
- **pfsync** (state table sync) — enable on all nodes; use a dedicated/isolated sync interface, since pfsync has no built-in authentication.
- Only two-node clusters are officially supported. A surprisingly common "CARP failure" is actually a switch-layer issue (multicast handling, MAC address migration, port security blocking the failover) — check the switch before assuming pfSense itself is broken.

### 7.6 Traffic Shaping / QoS (if configured)

Two separate systems: the **Traffic Shaper** (ALTQ-based wizard, PF-integrated — can reduce max firewall throughput while active) for prioritization/guarantees (HFSC supports true bandwidth guarantees; PRIQ is simple priority but can starve low-priority queues; CBQ allows borrowing), versus **Limiters** (Firewall → Traffic Shaper → Limiters, `dummynet`-based) which is the only mechanism for hard per-IP/per-subnet caps — Limiters enforce ceilings only, they don't guarantee minimums.

### 7.7 Safe Change Management

1. Backup config
2. Make incremental changes
3. Test connectivity
4. Monitor logs
5. Save changes
6. Document modifications

---

## 8. Ubiquiti Switch/Router Administration

### 8.1 Platform Differences

| Platform | GUI | CLI | Notes |
|---|---|---|---|
| UniFi Gateway / Cloud Console | UniFi Network app | SSH (support-directed only) | Centralized management; routine config is GUI-only by design |
| UniFi Switch (USW-series) | UniFi Network app | SSH (support-directed only) | Nearly all configuration goes through the controller, not the switch directly |
| EdgeRouter | Standalone Web GUI | VyOS-style CLI | Full router CLI control — this is the one platform where CLI is a normal, first-class admin path |
| EdgeSwitch (ES-series, e.g. ES-24-250W/500W, ES-48-500W/750W) | Standalone Web GUI | Full Cisco-IOS-style standalone CLI | A completely different product line from UniFi switches — full CLI is a normal, first-class admin path here too. See Section 9. |

**Important:** "Ubiquiti switch" can mean either a UniFi USW-series switch (controller-managed, GUI-first, limited CLI) or a standalone EdgeSwitch (full independent CLI, no controller required). Confirm which one is actually deployed before following CLI instructions — they don't share firmware or command syntax.

### 8.2 Access

- GUI: `https://<controller-ip>:8443` (self-hosted) or the local Cloud Console UI
- SSH — **verification correction**: official Ubiquiti docs give different default credentials than commonly assumed. **UniFi Consoles/Gateways: `root` / `ui`** (`root`/`ubnt` on older devices). **UniFi Devices (switches, APs): `ui` / `ui`** (`ubnt`/`ubnt` on older devices). Neither uses `admin` as the SSH username — that's a common but incorrect assumption carried over from the web GUI login. SSH is **enabled by default on switches/APs** but **disabled by default on Consoles** (enable under Settings → Control Plane → Console).
- Ubiquiti's own guidance: don't use SSH for routine administration — "we do not recommend using SSH unless instructed by one of our Support Engineers." Treat CLI/SSH access on UniFi devices as a troubleshooting tool, not a primary admin interface (EdgeRouter is the exception).

### 8.3 Common Status Commands

`ip addr` (interfaces), `ip route` (routes), `uptime`, `top` (CPU), `df -h` (disk)

**Verification note:** commands like `mca-cli` / `mca-cli-op` / `info` show up frequently in community guides for UniFi switch shells, but a direct search of Ubiquiti's own help site returned zero official references to them — they appear to be undocumented/community-sourced rather than vendor-supported. The one officially documented shell command for adoption is `set-inform http://<controller-ip>:8080/inform`, run directly at the device's SSH prompt — this is the one legitimate end-user CLI use case Ubiquiti documents (cross-subnet adoption).

### 8.4 VLANs / Virtual Networks

- GUI: **Settings → Networks** → Create New Network (Ubiquiti's official term is "Virtual Network," used interchangeably with VLAN). Configurable fields include Gateway IP/Subnet, DHCP lease time (default 86400s), and DHCP reservations.
- **Verification note**: newer UniFi Network releases (9.0+) introduced Zone-Based Firewalls, which reorganized some Settings navigation between versions (e.g., zone/policy config moved between a couple of release cycles). If your controller's menu doesn't match exactly, look under Settings for "Networks," "Zones," or "Policy" — Ubiquiti has been actively restructuring this area.
- EdgeRouter CLI (VyOS-style):
  ```
  configure
  set interfaces ethernet eth1 vif 10 description STAFF
  set interfaces ethernet eth1 vif 10 address 192.168.10.1/24
  commit
  save
  exit
  ```

### 8.5 DHCP

- Show leases: `show dhcp leases`
- Restart safely: `systemctl restart dnsmasq`

### 8.6 Firewall (EdgeRouter)

- GUI: Settings → Security → Traffic & Firewall Rules
- CLI:
  ```
  configure
  set firewall name WAN_IN rule 10 action accept
  set firewall name WAN_IN rule 10 state established enable
  set firewall name WAN_IN rule 10 state related enable
  commit
  save
  exit
  ```

### 8.7 VPN on UniFi/EdgeRouter

- **Verification update**: for remote-access VPN on modern ("next-gen") UniFi gateways, Ubiquiti's own docs steer admins toward **WireGuard** or **Teleport** (which is built on WireGuard) rather than L2TP — quoting their guidance directly, L2TP is called out as "less secure than Teleport and Wireguard." If this site's UniFi gateway supports it, prefer WireGuard/Teleport over L2TP for any new remote-access VPN.
- Status (legacy IPsec-based setups): `ipsec status`
- Restart: `systemctl restart strongswan`

### 8.8 Service Restarts

| Service | Command |
|---|---|
| UniFi controller | `systemctl restart unifi` |
| SSH | `systemctl restart ssh` |
| Web UI | `systemctl restart nginx` |

**Caution:** avoid `systemctl restart networking` remotely — this can drop your management session.

### 8.9 EdgeRouter Initial Setup Reference

```
configure
set system host-name EdgeRouter01
set interfaces ethernet eth0 address 192.168.1.1/24
set protocols static route 0.0.0.0/0 next-hop 192.168.1.254
set system name-server 1.1.1.1
set system name-server 8.8.8.8
commit
save
exit
```

### 8.10 Backups (UniFi)

GUI: **Settings → Control Plane → Backups** — automatic cloud backups run weekly and before major updates; restore from the same screen with a "Restore All Applications and Settings" option. For EdgeRouter, use the standalone GUI's config backup/download before any change, the same way you would for pfSense.

---

## 9. EdgeSwitch (Standalone Managed Switch) CLI Reference

If the switches on this network are actual **EdgeSwitch** hardware (ES-24-250W, ES-24-500W, ES-48-500W, ES-48-750W, etc.) rather than UniFi USW-series switches, they run a completely different, full-featured standalone CLI — Cisco-IOS-style, not VyOS-style like EdgeRouter and not GUI-only like UniFi. Everything in this section is sourced directly from Ubiquiti's official *EdgeSwitch CLI Command Reference*. Do not apply this CLI to UniFi (USW-series) switches — they don't run it.

### 9.1 Access & Command Modes

- SSH, Telnet, or console to the switch; starts in User EXEC mode (`(UBNT EdgeSwitch)>`)
- `enable` → Privileged EXEC (`#`) — from here you can configure the device
- `configure` → Global Config (`(config)#`)
- `interface 0/1` (slot/port notation, e.g. slot 0 port 1) → Interface Config for a specific port
- `vlan database` → VLAN Config mode
- `line {console | telnet | ssh}` → Line Config mode (login/timeout/session settings)
- `do <privileged-exec-command>` — run a Privileged EXEC command from any config mode without leaving it

### 9.2 VLANs

- `vlan database` then `vlan 10` creates VLAN 10 (VLAN Config mode)
- Assign a port: `interface 0/1` → `vlan participation include 10` → `vlan pvid 10` (for an untagged/access port); leave a port tagged-only for a trunk
- `show vlan` lists configured VLANs

### 9.3 802.1X Port-Based Authentication

EdgeSwitch supports full 802.1X — this resolves an earlier gap in this guide, which could only confirm this for EdgeSwitch specifically, not for UniFi's controller-managed switches:

```
configure
dot1x system-auth-control          # enable 802.1X globally (disabled by default)
interface 0/1
dot1x port-control auto            # normal 802.1X negotiation
```

`dot1x port-control` modes: `force-unauthorized` (always block), `force-authorized` (always allow), `auto` (normal 802.1X — the default once enabled), `mac-based` (MAC-based 802.1X, for multiple devices on one port). Apply to every port at once with `dot1x port-control all {mode}` in Global Config. There's also a documented **monitor mode** (`dot1x system-auth-control monitor`) built specifically for testing an 802.1X rollout without disrupting hosts that haven't been enrolled yet.

### 9.4 SNMP

```
configure
snmp-server community MyCommunity ro ipaddress 10.0.0.5
snmp-server host 10.0.0.5 traps version 2 MyCommunity
snmp-server enable traps
```

`snmp-server community` sets the community string, access mode (`ro`/`rw`/`su`), and can restrict it to a specific manager IP. `snmp-server host` configures where traps (or SNMPv2 informs) get sent, including the destination UDP port (default 162) and an optional trap filter. `show snmp` displays the current configuration.

### 9.5 Cable Diagnostics ("Cable Test")

This resolves another earlier open question in this guide — EdgeSwitch has a documented, built-in cable test:

```
cablestatus 0/1
```

Run from Privileged EXEC mode. Copper only (not supported on fiber/SFP ports), and it can briefly drop an active link while it runs. Returns one of: **Normal**, **Open** (disconnected or bad connector), **Short** (electrical short), or **Cable Test Failed** (inconclusive — the cable may still be fine), plus an estimated cable length range where the port's PHY supports it at the current link speed.

### 9.6 ACLs (Firewall-Equivalent)

EdgeSwitch filters traffic with numbered ACLs rather than named firewall rules:

```
access-list 101 deny tcp any host 10.0.0.50 eq 23
access-list 101 permit every
interface 0/1
ip access-group 101 in
```

Standard ACLs are numbered 1–99 (source-IP match only); extended ACLs are 100–199 (protocol, source/destination IP and port, TCP flags, DSCP, and more). Rules are evaluated top-down with first-match-wins logic, the same model as pfSense's rule tabs.

### 9.7 Firmware Upgrades (Dual-Image)

EdgeSwitch keeps two firmware images (active + backup) so an upgrade doesn't leave the switch stuck if the new image has a problem:

```
copy tftp://<server-ip>/<path>/<image-file> backup
boot system backup
reload
show bootvar
```

`copy ... backup` downloads the new image into the inactive slot while the switch keeps running normally on the current one. `boot system backup` marks that image as active for the *next* boot (the currently-running image automatically becomes the new backup). `reload` restarts the switch into the new image. `show bootvar` confirms which image is active/backup and their versions. If the new image doesn't come up cleanly, running `boot system backup` again from the console reverts to the previous image.

### 9.8 PoE, Spanning Tree, and LACP (reference)

- PoE status: `show poe status`, `show poe port {all | slot/port}`, `show poe counters`
- Spanning tree: full STP/RSTP/MSTP support — `spanning-tree`, `spanning-tree bpduguard`, `spanning-tree forceversion 802.1w` (RSTP)
- Link aggregation: `port-channel` and the `lacp` command family for LACP-based LAGs

### 9.9 Basic Admin Reference

| Task | Command |
|---|---|
| Save running config to startup | `copy system:running-config nvram:startup-config` |
| Back up config off-box | `copy nvram:startup-config tftp://<server>/<path>/<file>` |
| View interface status | `show interfaces status [slot/port]` |
| Reboot | `reload` (Privileged EXEC) |
| Add a local user | `username <name> password <pw>` |
| Configure login auth method | `aaa authentication login` |

---

## 10. Ubiquiti Troubleshooting

| Task | Command |
|---|---|
| Ping | `ping 8.8.8.8` |
| Traceroute | `traceroute 8.8.8.8` |
| Packet capture | `tcpdump -i eth0` |
| ARP table | `arp -a` |

**Console access**: requires a USB serial adapter or RJ45 console cable; 115200 8N1, no flow control (consistent with pfSense's console settings above).

- Windows: install USB serial drivers → Device Manager to find the COM port → PuTTY, Serial connection, 115200/8/None/1/None → Enter to wake the session.
- Linux: `dmesg | grep tty` to find the device (typically `ttyUSB0`) → `sudo screen /dev/ttyUSB0 115200` (exit `Ctrl+A`, `K`, `Y`).
- macOS: `ls /dev/cu.*` to find the device → `screen /dev/cu.usbserial-0001 115200`.

**High-risk commands — avoid remotely**: `reboot`, `systemctl restart networking`, `ifdown eth0`, `ip link set eth0 down`, factory reset.

---

## 11. Common Sysadmin Procedures (All Platforms)

### 11.1 Safe Change Management

1. Back up current configuration
2. Make one small/incremental change at a time
3. Validate connectivity before moving on
4. Commit and save the change
5. Monitor logs afterward
6. Document what was changed and why

### 11.2 Best Practices

- Restrict GUI/management access to an admin workstation or VPN subnet; don't expose management interfaces to the internet, on any port
- Use certificate- or RADIUS-based VPN authentication where possible over plain PSK
- Enable IDS/IPS (Suricata/Snort) on pfSense
- Use GeoIP filtering for known-hostile regions
- Back up configuration before any upgrade or major change — and confirm ACB (pfSense) / cloud backup (UniFi) is actually enabled, not just assumed
- Keep firmware/software current on both pfSense and Ubiquiti devices; check release notes for backend-changing updates (e.g., ISC→Kea DHCP migration on pfSense) before upgrading
- Review logs regularly, not just when something breaks
- Use firewall aliases instead of maintaining individual IP rules
- Restrict outbound traffic by VLAN where practical, especially for segments that shouldn't reach the internet directly
- Don't assume default SSH usernames without checking — pfSense uses `admin`, but Ubiquiti gateways/consoles use `root` and Ubiquiti switches/APs use `ui` (or `ubnt` on older firmware)
- Before escalating a VPN or WAN "config" problem, rule out physical layer and MTU/fragmentation first (Section 3) — both are common, both are easy to miss, and both look like something else at first glance
- On EdgeSwitch, use `cablestatus slot/port` (Section 9.5) as a first-line physical-layer check before swapping cables blindly
- Enable 802.1X in monitor mode first when rolling it out on EdgeSwitch (Section 9.3) — it surfaces which devices would fail authentication without actually blocking them yet

### 11.3 Emergency Reference

| Situation | pfSense | Ubiquiti |
|---|---|---|
| Firewall not applying rules | `pfctl -f /tmp/rules.debug` | Re-provision device from controller |
| VPN down | Check Status → IPsec / `swanctl --list-sas` | `systemctl restart strongswan`, then `ipsec status` |
| DNS not resolving | `service unbound restart` | `systemctl restart dnsmasq` |
| Need interface status fast | `ifconfig` | `ip addr` |
| Need routing table fast | `netstat -rn` | `ip route` |
| Need to see live traffic | `tcpdump -i em0` (or `enc0` for IPsec) | `tcpdump -i eth0` |
| Locked out by brute-force protection | `pfctl -T flush -t sshguard` | Re-provision/adopt from controller |
| VPN up but hangs on large transfers | MSS clamping, see Section 3.2 | N/A (IPsec is legacy on UniFi; prefer WireGuard) |
| Suspect bad cable on an EdgeSwitch port | — | `cablestatus slot/port` |

---

## 12. Verification Notes — What Couldn't Be Fully Confirmed

All pfSense-related content in this guide has been cross-checked directly against the official **pfSense Documentation** (the full 2,502-page Netgate manual, current as of late June 2026) — every pfSense fact, GUI path, CLI command, and default value cited above was confirmed word-for-word against that manual, including the full console menu, the IKEv2/split-tunnel mechanics, MSS clamping defaults, DHCP backend status, sshguard defaults, ACB licensing, and firewall rule processing order. Earlier passes also cross-checked against the live `docs.netgate.com` site; both sources agree.

All **EdgeSwitch**-specific content in Section 9 (802.1X, SNMP, cable diagnostics, ACLs, dual-image firmware upgrade) has been directly verified against Ubiquiti's official *EdgeSwitch CLI Command Reference*. This resolves several gaps flagged in earlier passes — but only for EdgeSwitch hardware specifically, not for UniFi USW-series switches, which run entirely different firmware and don't expose this CLI.

Remaining Ubiquiti gaps — an earlier research pass against `help.ui.com` was cut short by a web-fetch rate limit, and the EdgeSwitch reference doesn't cover UniFi or EdgeRouter. Treat the following as **not yet vendor-verified** and worth double-checking against your specific hardware/firmware:

- 802.1X / port-based authentication on **UniFi USW-series switches specifically** (confirmed for EdgeSwitch in Section 9.3 — but that's a different product line and doesn't confirm anything about UniFi's controller-managed switches).
- SNMP monitoring setup on **UniFi devices and EdgeRouter specifically** (confirmed for EdgeSwitch in Section 9.4).
- The specific name/location of any "cable test" feature on a **UniFi** switch model (EdgeSwitch's is confirmed as `cablestatus` in Section 9.5 — UniFi's GUI-based equivalent, if any, wasn't verified).
- EdgeRouter CLI syntax beyond what's shown in Section 8 (full firewall rule-set reference, SNMP, 802.1X) — the VyOS-style syntax shown matches long-established EdgeOS conventions but wasn't re-confirmed against current docs.
- UniFi AP / wireless-specific troubleshooting (not covered in this guide at all — this is a router/switch guide).
- EdgeRouter firmware upgrade procedure specifics.
- Exact Kea DHCP service/restart command name on pfSense for your installed version (the pfSense manual confirms Kea vs. ISC status but doesn't list every version's exact service name).

If any of these matter for day-to-day operations here, it's worth a follow-up pass, or a quick check against Ubiquiti's own current help site.

---

## Sources

**pfSense / Netgate**:
- *The pfSense Documentation* (Netgate, official PDF manual, 2,502 pages, current as of late June 2026) — primary source for this revision; used to directly verify IPsec/IKEv2 mechanics and the official mobile VPN worked example, the full console menu, MSS clamping defaults, DHCP backend (Kea/ISC) status, sshguard defaults, Auto Config Backup licensing, and firewall/NAT rule processing order.
- docs.netgate.com (live site, same content, used in earlier passes):
- [IPsec overview](https://docs.netgate.com/pfsense/en/latest/vpn/ipsec/index.html) · [Phase 1](https://docs.netgate.com/pfsense/en/latest/vpn/ipsec/configure-p1.html) · [Phase 2](https://docs.netgate.com/pfsense/en/latest/vpn/ipsec/configure-p2.html) · [Mobile Clients](https://docs.netgate.com/pfsense/en/latest/vpn/ipsec/mobile-clients.html) · [IPsec status monitoring](https://docs.netgate.com/pfsense/en/latest/monitoring/status/ipsec.html) · [Client Routing and Gateway Considerations](https://docs.netgate.com/pfsense/en/latest/vpn/ipsec/client-routing.html)
- [Troubleshooting IPsec Traffic](https://docs.netgate.com/pfsense/en/latest/troubleshooting/ipsec-traffic.html) · [Troubleshooting IPsec Connections](https://docs.netgate.com/pfsense/en/latest/troubleshooting/ipsec-connections.html) · [tcpdump diagnostics](https://docs.netgate.com/pfsense/en/latest/diagnostics/packetcapture/tcpdump.html) · [Advanced Firewall & NAT (MSS clamping)](https://docs.netgate.com/pfsense/en/latest/config/advanced-firewall-nat.html)
- [Troubleshooting Low Interface Throughput](https://docs.netgate.com/pfsense/en/latest/troubleshooting/low-throughput.html) · [Troubleshooting Lost Traffic or Disappearing Packets](https://docs.netgate.com/pfsense/en/latest/troubleshooting/packet-loss.html) · [Interface Configuration (MTU/MSS/Speed-Duplex fields)](https://docs.netgate.com/pfsense/en/latest/interfaces/configure.html)
- [Firewall rule fundamentals](https://docs.netgate.com/pfsense/en/latest/firewall/fundamentals.html) · [Floating rules](https://docs.netgate.com/pfsense/en/latest/firewall/floating-rules.html) · [Aliases](https://docs.netgate.com/pfsense/en/latest/firewall/aliases-types.html) · [NAT process order](https://docs.netgate.com/pfsense/en/latest/nat/process-order.html)
- [VLAN configuration](https://docs.netgate.com/pfsense/en/latest/vlan/configuration.html) · [QinQ](https://docs.netgate.com/pfsense/en/latest/interfaces/qinq.html)
- [DHCP overview / Kea vs ISC](https://docs.netgate.com/pfsense/en/latest/services/dhcp/index.html) · [Advanced Networking (backend selector)](https://docs.netgate.com/pfsense/en/latest/config/advanced-networking.html)
- [High Availability overview](https://docs.netgate.com/pfsense/en/latest/highavailability/index.html) · [HA settings](https://docs.netgate.com/pfsense/en/latest/highavailability/settings.html) · [pfsync](https://docs.netgate.com/pfsense/en/latest/highavailability/pfsync.html)
- [Traffic Shaper](https://docs.netgate.com/pfsense/en/latest/trafficshaper/index.html) · [Limiters](https://docs.netgate.com/pfsense/en/latest/trafficshaper/limiters.html)
- [Backup & Restore](https://docs.netgate.com/pfsense/en/latest/backup/index.html) · [Auto Config Backup](https://docs.netgate.com/pfsense/en/latest/backup/acb.html)
- [Upgrade guide](https://docs.netgate.com/pfsense/en/latest/install/upgrade-guide.html)
- [Advanced Admin Access](https://docs.netgate.com/pfsense/en/latest/config/advanced-admin.html) · [Remote firewall administration](https://docs.netgate.com/pfsense/en/latest/recipes/remote-firewall-administration.html) · [User Manager defaults](https://docs.netgate.com/pfsense/en/latest/usermanager/defaults.html) · [User Manager settings](https://docs.netgate.com/pfsense/en/latest/usermanager/settings.html) · [Locked out recovery](https://docs.netgate.com/pfsense/en/latest/troubleshooting/locked-out.html)

**Ubiquiti**:
- *EdgeSwitch CLI Command Reference* (Ubiquiti Networks, official PDF manual) — primary source for Section 9 (EdgeSwitch access/modes, VLANs, 802.1X, SNMP, cable diagnostics, ACLs, dual-image firmware upgrade, PoE/STP/LACP commands).
- help.ui.com (UniFi/EdgeRouter, used in earlier passes):
- [Creating Virtual Networks (VLANs)](https://help.ui.com/hc/en-us/articles/9761080275607) · [Changing a Virtual Network Subnet](https://help.ui.com/hc/en-us/articles/16602606163991)
- [Connecting to UniFi with Debug Tools (SSH)](https://help.ui.com/hc/en-us/articles/204909374) · [Remote Adoption (Layer 3 / set-inform)](https://help.ui.com/hc/en-us/articles/204909754) · [UniFi Advanced Updating Techniques](https://help.ui.com/hc/en-us/articles/204910064)
- [UniFi Gateway L2TP VPN Server](https://help.ui.com/hc/en-us/articles/12594825307927) · [UniFi Gateway WireGuard VPN Server](https://help.ui.com/hc/en-us/articles/115005445768) · [Introduction to VPNs](https://help.ui.com/hc/en-us/articles/7951513517079)
- [Backups and Migration in UniFi](https://help.ui.com/hc/en-us/articles/360008976393)
- [Zone-Based Firewalls in UniFi](https://help.ui.com/hc/en-us/articles/115003173168)

*Original source material for the first draft of this guide: pfSense Administration and Security Cheat Sheet; Ubiquiti Router and Switch Administration Cheat Sheet (unsourced/community-style references, since superseded above where they conflicted with vendor docs).*

# pfSense + Snort Firewall / IDS Administration Guide

> Status: initial draft, 2026-10-04. Contains [FILL IN] markers — see INDEX.md.

## Overview

This is the day-2 administration guide for the organization's pfSense edge firewall at the PO (head office) and the Snort IDS/IPS package that runs on it. It covers how the box is laid out (WAN circuits, the 10.60.0.0/16 server hub and the other segments, NAT, the IPsec tunnels to the partner site and the cloud VPS and the mobile IKEv2 VPN), how to administer rules, aliases and NAT, how Snort is configured and how to unblock, pass-list and suppress, and how to back up, upgrade, monitor and rebuild the firewall.

It is written for the sysadmin and for whoever covers when the sysadmin is away. It deliberately does **not** repeat three things that already exist in the library:

- **How to make a firewall change safely** (intake, review, snapshot, two-path verification, rollback, change log) — `Networking Guide/firewall-change-procedure.md`. Every widening change described in this guide is done *through* that procedure.
- **Generic pfSense mechanics** (console menu, VLAN creation, DHCP backends, MTU/MSS walkthrough, CARP) — `Networking Guide/Network-Administration-Guide.md` §4–7 and `Networking Guide/pfsense.txt`.
- **The address plan** — `Networking Guide/ipam-vlan-topology-reference.md`. This guide records the facts already visible in screenshots and KB pages so that table can be populated from them.

What this guide adds is the environment-specific picture, the Snort side (which has no runbook other than the one-page unblock KB), and the box-level operations: backup, upgrade, logging, monitoring, DR and hardening.

## Quick Facts

| Field            | Value |
|------------------|-------|
| Owner            | IT lead (it@example.com), IT Ops |
| Environment      | prod — production edge firewall for the PO site. No staging firewall. A second pfSense Plus firewall (`partner-router.example.com`, LAN `192.168.150.1`) runs at the partner site and is the far end of the partner tunnel. |
| Platform         | Netgate **pfSense Plus** (the GUI banner in every screenshot reads "pfSense+" / "Netgate pfSense Plus"). Version: [FILL IN: System → Update, or Status → Dashboard → System Information → Version] |
| Hardware         | [FILL IN: Netgate appliance model or whitebox/VM spec, serial number, rack location — Status → Dashboard → System Information] |
| Management name  | `fw01.example.com` [CONFIRM: which name(s) resolve, to which internal address, and which should be the documented one. Older KB articles use variant spellings, including one with an underscore, which is not valid in a DNS hostname.] |
| Management IP    | [FILL IN: LAN/management interface address the GUI is reached on — `ifconfig` on the box, or Interfaces → LAN] |
| Access           | Web GUI over HTTPS from the internal network or VPN; SSH and console as in `Network-Administration-Guide.md` §4.1 and §6. Admin credentials in 1Password — [FILL IN: vault and item name]. GUI must not be reachable from the internet. |
| WAN circuits     | `CARRIER_WAN` — the carrier, public block including 203.0.113.179–203.0.113.186 (used for all published services). `VPN2_WAN` — the interface the IPsec Phase 1s bind to; the partner tunnel peers to `198.51.100.52`. [CONFIRM: that 198.51.100.52 is the VPN2_WAN address, which ISP provides that circuit, and whether there is gateway failover between the two] |
| Internal hub     | `10.60.0.0/16` — Servers and Systems (the hub network all others connect to). Internal DNS answering at `10.60.1.221` (seen as the resolver in the Snort KB `dig` output). |
| IDS/IPS          | Snort package (Services → Snort). Blocks into pf table `snort2c`. Custom pass list "ORG custom pass list" includes the `Venue` alias. Interfaces/mode/rule sets: [FILL IN — see §Snort Configuration] |
| Config file      | `/cf/conf/config.xml` (the entire configuration, including Snort settings) |
| Dependencies     | Carrier circuit(s); internal DNS; Graylog (graylog01.example.com) for off-box logs; Netgate ACB for off-box config; Snort rule download sites (snort.org, emergingthreats.net) |
| Dependents       | Everything internet-facing (mail, barracuda, www, agm, sign, dms, send, static, retro, wordpress, fc), partner and cloud VPS tunnels, mobile IKEv2 VPN, WireGuard, inter-VLAN routing for every segment |
| Last reviewed    | 2026-10-04 (new draft — UNVERIFIED) |

## How It Works

### Topology at a glance

```
                     Internet
          ┌─────────────┴──────────────┐
     CARRIER_WAN                  VPN2_WAN
 203.0.113.179–.186           198.51.100.52 [CONFIRM]
 (published services,         (IPsec P1s: mobile IKEv2,
  manual outbound NAT)         cloud VPS, partner)
          └─────────────┬──────────────┘
                 pfSense Plus (PO)  ── Snort on [FILL IN: interfaces]
                        │
     ┌──────────┬───────┼─────────┬──────────┬────────────┐
 10.60.0.0/16   PO Staff  PO Guest  PO Phones   VPN pools
 Servers hub   .30.0/24  .20.0/24   .3.0/24   .40.0/24 (systems)
                                              .60.0/24 (staff)
                        │
             IPsec ─────┼───────────────────────────────┐
     Partner (partner-router, 192.0.2.26)     Cloud VPS (192.0.2.90)
     192.168.150.0/24 Staff, .2 Pocket,       192.168.50.0/24 "Cloud VPS Pocket"
     .4 Phones, 10.0.1.0/24 Guest
```

### Address plan (from `[internal KB: network subnet list]`)

| Subnet | Purpose | Where it lives |
|---|---|---|
| 10.60.0.0/16 | Servers and Systems — the hub network everything connects to | PO |
| 192.168.20.0/24 | PO Guest | PO |
| 192.168.30.0/24 | PO Staff | PO |
| 192.168.3.0/24 | PO Phones | PO |
| 192.168.40.0/24 | VPN — to become the Systems-only VPN. Appears as the second Phase 2 on the mobile IKEv2 P1 | PO (mobile client pool) |
| 192.168.60.0/24 | VPN — to become the Staff VPN | PO [FILL IN: which VPN — WireGuard or IPsec — uses this pool] |
| 192.168.50.0/24 | Cloud VPS Pocket — reached over IPsec P1 #2 | Cloud VPS |
| 192.168.150.0/24 | Partner Staff (partner-router LAN is 192.168.150.1) | Partner |
| 192.168.2.0/24 | Partner Pocket | Partner |
| 192.168.4.0/24 | Partner Phones | Partner |
| 10.0.1.0/24 | Partner Guest | Partner |

VLAN IDs, pfSense interface names and gateway addresses for each segment are not recorded anywhere yet. [FILL IN: populate the VLAN table in `ipam-vlan-topology-reference.md` from Interfaces → Assignments → VLANs, then link back here rather than copying it.]

### Published services — Outbound NAT (from `pfsense_outbound_nat_mappings.jpeg`)

Outbound NAT is in **Manual Outbound NAT (AON)** mode. Each published server has a /32 mapping on `CARRIER_WAN` so its outbound traffic leaves from the same public address its inbound DNS points to (this matters for mail PTR/SPF and for vendor allow-lists).

| Internal source | NAT address | Description in rule | Notes |
|---|---|---|---|
| 10.60.1.241/32 | 203.0.113.179 | barracuda.example.com | Shares .179 with mail and fc |
| 10.60.1.212/32 | 203.0.113.179 | mail.example.com | Mail sends from .179 — PTR must match |
| 10.60.1.114/32 | 203.0.113.179 | fc.example.com | |
| 10.60.1.216/32 | 203.0.113.180 | www.example.com | |
| 10.60.1.230/32 | 203.0.113.181 | retro.example.com | Retrospect, Synology Drive, Munki |
| 10.60.1.128/32 | 203.0.113.182 | wordpress.example.com | |
| 10.60.1.93/32  | 203.0.113.183 | dms.example.com | |
| 10.60.1.124/32 | 203.0.113.184 | send.example.com | |
| 10.60.1.244/32 | 203.0.113.185 | static.example.com | |
| 10.60.1.252/32 | 203.0.113.186 | agm.example.com | |

The screenshot is cropped at the bottom. [FILL IN: remaining outbound mappings, in particular the catch-all rule(s) for 10.60.0.0/16, the staff/guest subnets and the VPN pools — in Manual mode, any subnet without a mapping gets **no** NAT and cannot reach the internet.] The Static Port column shows the shuffle icon on every row, meaning source ports are randomised (static port off) — [CONFIRM].

### Published services — Port forwards (from `firewall_nat_port_forward_edit.jpeg`, `carrier_wan_tcp_https_rule.jpeg`, `pfsense_port_forward_https_rule.jpeg`)

The pattern: one Virtual IP per public address on `CARRIER_WAN`, one port forward per service, each with a **linked** filter rule ("Filter rule association: Rule NAT …") so the WAN rule is created and maintained by the NAT entry.

| Interface | Proto | Destination | Port | Redirect target | Description | Status |
|---|---|---|---|---|---|---|
| CARRIER_WAN | TCP | 203.0.113.186 (agm.example.com) | 443 | 10.60.1.252:443 | agm.example.com (https) | Confirmed by two screenshots |
| CARRIER_WAN | TCP | 203.0.113.186 (agm.example.com) | 443 | 10.60.1.241:443 | sign.example.com (https) | **Anomaly** — see below |

**Anomaly to resolve.** `pfsense_port_forward_https_rule.jpeg` shows an edit form with destination `203.0.113.186 (agm.example.com)`, redirect target `10.60.1.241` (the Barracuda, per outbound NAT) and description `sign.example.com (https)`, while its filter rule association still reads "Rule NAT agm.example.com (https)". Two forwards cannot share destination/port; the first match wins and the second is dead. Either this was a screenshot taken mid-edit, or sign.example.com is published through the Barracuda on a different VIP and the destination was mis-selected. [CONFIRM: on Firewall → NAT → Port Forward, which public IP sign.example.com uses, whether it passes through the Barracuda WAF at 10.60.1.241, and that no duplicate agm 443 forward exists.]

[FILL IN: the full port-forward list — Firewall → NAT → Port Forward, every row, transcribed into the inbound NAT table in `ipam-vlan-topology-reference.md`. Mail ports 25/465/587/993/995/443 to mail.example.com are expected per `Mail & Messaging/zimbra-mail-administration-guide.md`.]

### IPsec (from `pfsense_ipsec_tunnels_config.jpeg` and `pfsense_ipsec_status_overview.jpeg`)

PO side, VPN → IPsec → Tunnels:

| P1 | IKE | Remote gateway | Auth | P1 crypto | Description | Phase 2s |
|---|---|---|---|---|---|---|
| 1 | v2 | Mobile clients on VPN2_WAN | EAP-MSCHAPv2 | AES-256 / SHA256 / DH 14 | ORG IPsec VPN | P2 1: tunnel, LAN subnet, ESP AES-256/SHA256. P2 2: tunnel, 192.168.40.0/24, ESP AES-256/SHA256 |
| 2 | v2 | 192.0.2.90 on VPN2_WAN | Mutual PSK | AES-256 / SHA256 / DH 14 | Connection to cloud VPS (192.168.50.0/24) | P2 3: LAN subnet ↔ 192.168.50.0/24, ESP AES128-GCM |

Partner side, Status → IPsec → Overview on `partner-router.example.com` (192.168.150.1):

| ID | Description | Local | Remote | Role | Algorithms | State seen |
|---|---|---|---|---|---|---|
| con1 | Connect to PO | 192.0.2.26 | 198.51.100.52 | IKEv2 **Initiator** | AES_CBC-256, HMAC_SHA2_256_128, PRF_HMAC_SHA2_256, MODP_2048 (DH 14) | Established, 1 child SA, reauth disabled |

The PO Tunnels screenshot shows only P1 1 and P1 2 — there is no visible partner Phase 1. [CONFIRM: whether the PO-side partner P1 exists (screenshot older than the tunnel, list cropped, or the partner site terminates as a mobile/responder-only peer), and record its P1/P2 IDs and Phase 2 selectors here.] The partner site initiates; if PO is responder-only, an outage on the partner side cannot be fixed from PO — see the jump-host procedure in `[internal KB: partner VPN jump-host procedure]`.

Note "LAN subnet" in the Phase 2s is a pfSense macro that resolves to whatever the LAN interface's network is. [FILL IN: what LAN's network actually is — if LAN is 10.60.0.0/16 then the hub is what the tunnels carry, and staff/guest VLANs are *not* reachable across them, which is probably correct.]

### Snort

Snort runs as a pfSense package. On each enabled interface it inspects traffic against the downloaded rule sets; in **Legacy (blocking) mode** a matching rule with "Block Offenders" enabled adds the offending IP to the pf table `snort2c`, and an automatic pf rule drops all traffic to/from addresses in that table. Blocks are not firewall rules and do not appear in Firewall → Rules. Addresses on the interface's **Pass List** are never blocked. **Suppress lists** stop specific alerts from firing at all.

Field symptom, from the Snort unblock KB: a site or service fails on wired and ORG_Wireless but works on Guest — that means Snort, not a rule. [CONFIRM: why Guest is unaffected — Snort's block table applies across all interfaces, so the likely reason is that the Guest network egresses by a different path or public IP than staff; record which.]

### Where things live on the box

| Item | Location |
|---|---|
| Full configuration | `/cf/conf/config.xml` |
| Local config history | `/cf/conf/backup/` (GUI: Diagnostics → Backup & Restore → Config History) |
| Generated ruleset | `/tmp/rules.debug` |
| Firewall log | `/var/log/filter.log` |
| System / IPsec / auth / resolver logs | `/var/log/system.log`, `/var/log/ipsec.log`, `/var/log/auth.log`, `/var/log/resolver.log` |
| Snort per-interface config | `/usr/local/etc/snort/snort_<id>_<iface>/` |
| Snort per-interface logs | `/var/log/snort/snort_<iface><id>/` (alert file is `alert`) |
| Snort rules (downloaded) | `/usr/local/etc/snort/rules/` |

Log format note: pfSense 2.5 / Plus 21.02 and later write **plain-text** logs rotated by newsyslog. The `clog` command quoted in `pfsense.txt` and older runbooks only applies to the old circular-log format and will not exist on current pfSense Plus. Use `tail`, `grep`, `less`.

---

## Access & Administration

### Getting in

| Method | How | Use when |
|---|---|---|
| Web GUI | `https://<management-name>/` from the internal network or VPN | Normal administration |
| SSH | `ssh <your-admin-user>@<management-ip>` | CLI diagnostics, pfctl, logs |
| Console | Serial 115200 8N1, no flow control, or VGA/HDMI | GUI locked out, interface misconfigured, upgrade recovery |
| Partner firewall | TeamViewer web client → `partner-jump` → `https://192.168.150.1` | Bringing the partner tunnel up from the partner side |

[FILL IN: physical location of the console port, and whether a USB-serial adapter is kept with the firewall.]

### Accounts

- Use individual named admin accounts, not the shared `admin`, so Config History and `/var/log/auth.log` show who made a change. [FILL IN: current accounts in System → User Manager and whether the built-in `admin` is disabled]
- [CONFIRM: proposal — authenticate admins against AD via LDAP or RADIUS (System → User Manager → Authentication Servers, auth2.example.com / auth4.example.com), with a local break-glass account whose credentials live in the IT vault. External RADIUS is also the way to get MFA on the GUI; there is no native TOTP toggle.]
- VPN users for the mobile IKEv2 P1 (EAP-MSCHAPv2) are local pfSense users with the "IPsec xauth Dialin"/IPsec privilege — [FILL IN: where VPN users are managed, and the offboarding step that removes them]. Add pfSense VPN user removal to the leaver checklist.

### GUI orientation (where admin tasks live)

| Task | GUI path |
|---|---|
| Rules | Firewall → Rules → [CARRIER_WAN / VPN2_WAN / LAN / each VLAN / IPsec / Floating] |
| Aliases | Firewall → Aliases (IP / Ports / URLs tabs) |
| Port forwards / 1:1 / Outbound | Firewall → NAT |
| Virtual IPs (public .179–.186) | Firewall → Virtual IPs |
| IPsec config / status | VPN → IPsec → Tunnels; Status → IPsec → Overview / SADs / SPDs |
| Snort | Services → Snort (Interfaces, Global Settings, Updates, Alerts, Blocked, Pass Lists, Suppress, IP Lists, SID Mgmt, Log Mgmt) |
| Logs | Status → System Logs (System, Firewall, VPN → IPsec, Packages → Snort) |
| Backup | Diagnostics → Backup & Restore; Services → Auto Configuration Backup |
| Upgrades | System → Update; System → Boot Environments |
| States | Diagnostics → States (filter + kill) |
| pf tables (snort2c, sshguard, aliases) | Diagnostics → Tables |

---

## Rules, Naming and Aliases

### Rule structure

Rules are evaluated per interface tab, top-down, first match wins, after Floating rules; the full evaluation order is in `firewall-change-procedure.md` "How It Works" and is not repeated here. Environment-specific structure to keep:

| Tab | Intended content | Reason |
|---|---|---|
| CARRIER_WAN | Only NAT-linked pass rules for published services, plus explicit blocks if needed. No GUI/SSH. | Every rule here is internet exposure; linking to NAT keeps rule and forward in step |
| VPN2_WAN | UDP 500, UDP 4500, ESP — restricted to the known peers (192.0.2.90, 192.0.2.26) where possible; mobile clients need `any` source | IPsec only; [CONFIRM: proposal to source-restrict site-to-site IKE to peer aliases] |
| LAN / server hub | Explicit permits out; management access to the firewall only from admin hosts | Default-deny between segments |
| Guest | Block to all RFC1918 (use an alias), then pass to internet | Guest must not reach anything internal |
| IPsec | Per-tunnel permits for the remote subnets — do not scope to TCP only (breaks ICMP/DNS, see `Network-Administration-Guide.md` §3.2) | Tunnel traffic is filtered here, not on WAN |
| Floating | Avoid. Anything here must be documented in the rule description | Floating rules shadow interface rules invisibly |

[FILL IN: confirm actual tab names from Firewall → Rules and whether each tab's contents match the intent above. Mismatches are findings for the quarterly review.]

### Naming

Observed convention: port forwards are described as `<fqdn> (<service>)`, e.g. `agm.example.com (https)`, and pfSense names the linked filter rule `NAT agm.example.com (https)`. Outbound NAT rows are described by FQDN alone. Keep that for NAT. For filter rules use the change-ID format from `firewall-change-procedure.md`:

```
CHG-2026-0NN | <ticket> | <requester> | purpose | expires YYYY-MM-DD or PERMANENT
```

[CONFIRM: proposal — NAT descriptions stay `<fqdn> (<service>)` so the linked rule name stays readable, and the change ID goes in the NAT entry's linked rule description via an edit after creation.]

### Aliases

Aliases are the main lever. Known and proposed:

| Alias | Type | Contents | Used by | Status |
|---|---|---|---|---|
| `Venue` | Host(s) | Public IPs of the current AGM/DSM/training venue | Snort pass list "ORG custom pass list" | Exists (Large Meeting KB). Empty it after each event. |
| `EasyRuleBlockHostsCARRIER_WAN` (or similar) | Host(s) | IPs blocked via easyrule / log "block" button | Auto-created block rule on WAN | Created automatically on first use — [CONFIRM: exact name on this box] |
| `RFC1918` | Network | 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 | Guest block rule | [CONFIRM: create if absent] |
| `ORG_Admin_Hosts` | Host(s) | Admin workstations allowed to reach GUI/SSH | Management rules, anti-lockout replacement | [CONFIRM: proposal] |
| `IPsec_Peers` | Host(s) | 192.0.2.90, 192.0.2.26 | VPN2_WAN IKE/ESP rules | [CONFIRM: proposal] |
| `Published_Servers` | Host(s) | 10.60.1.241, .212, .114, .216, .230, .128, .93, .124, .244, .252 | Snort Home Net review, outbound rules | [CONFIRM: proposal] |

Before editing an alias, find every rule that uses it — adding a host to an alias in a pass rule is a widening change:

```
grep -n '<alias-name>' /tmp/rules.debug              # every generated rule that references the alias
pfctl -t <alias-name> -T show                         # current resolved members of the alias table
```

URL Table aliases refresh on a schedule (default daily). Force a refresh from the shell:

```
/etc/rc.update_urltables now forceupdate              # re-download all URL table aliases now
```

---

## NAT Administration

### Adding a published service (port forward)

Do this inside `firewall-change-procedure.md` (it is a Reviewed change). The Environment-specific steps:

1. Confirm the public address: pick an unused Virtual IP from Firewall → Virtual IPs, or reuse the host's existing one. [FILL IN: which of .179–.186 are free, and whether more of the carrier's block exists beyond .186]
2. Firewall → NAT → Port Forward → Add. Interface `CARRIER_WAN`, protocol TCP (or as needed), Destination = the VIP (select it by name, e.g. `203.0.113.186 (agm.example.com)`), destination port, Redirect target IP = internal host, Redirect target port.
3. Description `<fqdn> (<service>)`. Filter rule association: **Add associated filter rule** (creates the linked "NAT …" rule on CARRIER_WAN).
4. If the service is web, decide whether it goes direct or via the Barracuda WAF (10.60.1.241). Do not expose a backend port — see `Linux & Servers/reverse-proxy-and-tls-automation-guide.md`.
5. Add a matching Outbound NAT /32 mapping so outbound traffic leaves from the same public IP (Manual mode will not do it for you).
6. Verify from outside (phone hotspot), both allow and deny paths, and record in the IPAM inbound NAT table.

```
pfctl -sn | grep -i '203.0.113.186'                   # confirm the rdr rule was generated for this VIP
pfctl -sr | grep -i 'NAT agm'                          # confirm the linked pass rule exists
```

### Outbound NAT

Because the box is in Manual mode, every new internal subnet (new VLAN, new VPN pool) needs an outbound mapping or it has no internet. Firewall → NAT → Outbound → Add: Interface `CARRIER_WAN`, Source = new subnet, Translation = Interface Address or a specific VIP. Then:

```
pfctl -sn | grep -E '^nat'                             # list active outbound NAT rules in order
```

| Setting | Value | Reason |
|---|---|---|
| Outbound NAT mode | Manual (AON) | Per-server /32 mappings pin each published host to its own public IP; needed for mail PTR/SPF and vendor allow-lists |
| Per-host mapping, static port | Off (randomised) | Static port only needed for a few UDP protocols; randomising is safer [CONFIRM] |
| NAT reflection | System default | [FILL IN: System → Advanced → Firewall & NAT → NAT Reflection mode — if internal users reach published names by public IP, reflection or split DNS is required; record which is used] |

---

## IPsec Site-to-Site Operations

### Status

GUI: Status → IPsec → Overview. Each P1 shows Established/Connecting/Disconnected; "Show child SA entries" lists the Phase 2s. Connect / Disconnect buttons per P1 and P2.

```
swanctl --list-sas                                     # active IKE and child SAs, bytes in/out per child
swanctl --list-conns                                   # configured connections and traffic selectors
swanctl --list-sas --ike con2                          # one connection only (conN = P1 ID, e.g. con2 = cloud VPS)
tail -n 100 /var/log/ipsec.log                         # recent IKE negotiation messages
```

The legacy `ipsec statusall` in `pfsense.txt` is not the current tool — use `swanctl`.

### Bring a tunnel up / down

```
swanctl --initiate --ike con2                          # bring up cloud VPS P1 (and its children)
swanctl --initiate --child con2_3                      # bring up a specific P2 (format conX_Y — confirm with --list-conns)
swanctl --terminate --ike con2                         # tear down cloud VPS P1
pfSsh.php playback svc restart ipsec                   # restart the whole IPsec service — drops ALL tunnels incl. mobile users
```

Restarting the service drops every mobile VPN user and every site tunnel; prefer per-connection initiate/terminate.

### Partner tunnel

The partner router is the initiator. If it is down, the documented fix is from the partner side: TeamViewer web client (not the macOS client) → `partner-jump` → `https://192.168.150.1` → Status → IPsec → Overview → Connect. `partner-static` is the office-side equivalent. See `[internal KB: partner VPN jump-host procedure]`.

### Troubleshooting — IPsec

#### Symptom: P1 will not establish
- Check: `grep -iE 'no proposal|AUTH_FAILED|NO_PROPOSAL_CHOSEN|timeout' /var/log/ipsec.log | tail -20` # which side rejects and why
- Check: `tcpdump -ni <vpn2_wan_if> host 192.0.2.90 and udp port 500 or udp port 4500` # are IKE packets arriving at all
- Fix: mismatch in proposal → align P1 algorithms/DH both ends; AUTH_FAILED → PSK or identifier mismatch (PSK lives in the IT vault, never here); no packets → peer's WAN, ISP, or our VPN2_WAN rules.

#### Symptom: P1 up, no traffic for one subnet
- Likely cause: Phase 2 selectors differ between ends, or the subnet is not in any P2.
- Check: `swanctl --list-sas` # is there a child SA for that subnet pair; are bytes increasing
- Check: Status → IPsec → SPDs # is there a policy for the subnet
- Fix: add/match the P2 on both ends. Then check the IPsec tab firewall rules permit it.

#### Symptom: tunnel works, then breaks hourly
- Likely cause: PFS / P2 lifetime mismatch — the first child SA inherits P1 DH, so a PFS mismatch only bites on rekey.
- Check: `grep -i rekey /var/log/ipsec.log | tail` # failures at rekey time
- Fix: match P2 PFS group both ends.

#### Symptom: small things work, large transfers hang
- Likely cause: MTU/MSS. Full walkthrough in `Network-Administration-Guide.md` §3.2.
- Check: `tcpdump -ni enc0 host <remote-host>` # traffic entering the tunnel
- Fix: System → Advanced → Firewall & NAT → MSS clamping for VPN, start at 1400.

---

## Snort Configuration

### Current state — to be recorded

| Item | Value | How to find |
|---|---|---|
| Package version | [FILL IN] | System → Package Manager → Installed Packages |
| Interfaces with Snort enabled | [FILL IN] | Services → Snort → Snort Interfaces |
| Mode per interface | [FILL IN: Legacy blocking / Inline IPS / IDS only] | Interface → Settings → Block Offenders, IPS Mode |
| Which IP to block | [FILL IN: SRC / DST / BOTH] | Interface → Settings |
| Kill states on block | [FILL IN] | Interface → Settings |
| Rule sets | [FILL IN: Snort Subscriber (oinkcode), GPLv2 Community, ET Open, ET Pro, OpenAppID] | Global Settings |
| IPS policy | [FILL IN: Connectivity / Balanced / Security / none] | Interface → Categories |
| Update interval | [FILL IN] | Global Settings → Update Rules Automatically |
| Remove blocked hosts interval | [FILL IN] | Global Settings → Remove Blocked Hosts Interval |
| Pass list(s) | "ORG custom pass list" (contains `Venue`) — [FILL IN: other members, and which interfaces use it] | Pass Lists tab |
| Suppress list(s) | [FILL IN] | Suppress tab |
| Send alerts to system log | [FILL IN] | Interface → Settings → Send Alerts to System Log |
| Keep settings after deinstall | [FILL IN: must be on] | Global Settings |

The oinkcode for Snort Subscriber rules is a credential — it lives in the IT vault, not in this document.

### Recommended configuration

| Setting | Value | Reason |
|---|---|---|
| Interfaces | CARRIER_WAN (and VPN2_WAN) [CONFIRM] | Inspect inbound attack traffic before NAT; internal-interface sensors add lateral-movement visibility (ransomware playbook wants internal SMB alerts) but double CPU cost — [CONFIRM: also LAN/server hub] |
| Mode | Legacy blocking with "Block Offenders" | What is already running (snort2c table, unblock KB). Inline mode needs netmap-capable NICs. |
| Which IP to block | SRC [CONFIRM] | BOTH blocks our own server's IP when it is the destination of an alert, which is exactly the "service unreachable" pattern in the KB |
| Kill states | On | Otherwise an attacker's established session survives the block |
| IPS policy | Balanced, start at Connectivity if false positives are high | Balanced is the Snort default tuning trade-off |
| Remove blocked hosts | 1 hour [CONFIRM] | Self-heals false-positive blocks; persistent attackers get re-blocked quickly |
| Rule update | Every 12 hours | Current signatures without update churn |
| Keep settings after deinstall | On | Package reinstall during upgrade otherwise wipes the configuration |
| Home Net | Default (all local networks + VPN pools) plus `Published_Servers` | Snort must know what "ours" is for rule direction |
| Pass list | Custom "ORG custom pass list": `Venue`, IPsec peers, upstream DNS, carrier gateway, critical SaaS (M365, carrier Jabber subnets per `jabber-dns-and-routing.md`) | Never block infrastructure we depend on |
| Send alerts to system log | On, facility LOG_AUTH, priority LOG_ALERT [CONFIRM] | Gets Snort alerts to Graylog via remote syslog |

### Unblocking an IP

Primary procedure is `[internal KB: unblocking an IP from Snort]`. Summary and CLI equivalents:

1. Confirm it is Snort: fails on wired and ORG_Wireless, works on Guest.
2. Find the service's IP: `dig <hostname>` from a workstation.
3. GUI: Services → Snort → Blocked → find the IP → red X. The Alerts tab (same IP) tells you *why*; note the GID:SID before unblocking.
4. Emergency only: "Clear" removes all blocks; known-bad IPs repopulate quickly.

```
pfctl -t snort2c -T show                               # list currently blocked addresses
pfctl -t snort2c -T show | grep 192.0.2.134          # is a specific address blocked
pfctl -t snort2c -T delete 192.0.2.134               # unblock one address (same as the red X)
pfctl -k 192.0.2.134                                 # kill any lingering states for it (usually not needed after unblock)
grep 192.0.2.134 /var/log/snort/snort_*/alert        # which rule blocked it
```

An unblock is temporary — if the traffic trips the same rule, it is blocked again. If it recurs, either pass-list the IP or suppress the rule (below).

### Pass lists

For addresses that must never be blocked (partners, venues, SaaS endpoints).

1. Firewall → Aliases → edit the alias (e.g. `Venue`) → add the IP → Save → Apply.
2. Because `Venue` is already in "ORG custom pass list", nothing else changes. For a new alias: Services → Snort → Pass Lists → edit "ORG custom pass list" → Assigned Aliases → add → Save.
3. Restart Snort on the interface (Snort Interfaces → restart icon) so the pass list reloads.

Large-meeting prep (AGM, DSM, training events) touches several systems — fail2ban `ignoreip` and Zimbra `zimbraHttpThrottleSafeIPs` on mail.example.com, the `Venue` alias, pausing Staff Retrospect, and suspending carrier DDoS protection. Follow `[internal KB: large meeting IP whitelist prep]` and set a reminder to reverse every step after the event.

### Suppress lists (false positives)

A pass list protects an *address*; a suppression silences a *rule* (for everyone, or for one address). Suppressions are permanent reductions in coverage — treat them as Reviewed changes in `firewall-change-procedure.md`.

Quick path: Services → Snort → Alerts → select interface → on the alert row, click the "+" next to the SID to suppress it entirely, or the "+" next to the source/destination IP to suppress it for that address only. That writes into the interface's suppress list.

Manual entries (Services → Snort → Suppress → edit list), one per line:

```
suppress gen_id 1, sig_id 2013504                                  # silence a rule everywhere — last resort
suppress gen_id 1, sig_id 2013504, track by_src, ip 10.60.1.212   # silence only when our mail server is the source
suppress gen_id 120, sig_id 3, track by_dst, ip 203.0.113.179     # http_inspect preprocessor alert, only for traffic to .179
```

Prefer `track by_src`/`by_dst` with an IP over a global suppression. Put the reason and change ID in a `#` comment line above each entry.

### Tuning workflow

1. Run each newly enabled interface or ruleset in alert-only mode (Block Offenders off) for [CONFIRM: 7 days] first.
2. Alerts tab → sort by SID → identify the top talkers. For each: true positive, or benign traffic from our own services?
3. Benign + rule is irrelevant to our estate → disable the SID (Interface → Rules → click to disable, or SID Mgmt with a `disablesid.conf` list so it survives updates).
4. Benign only for specific hosts → targeted suppression.
5. Turn blocking on. Watch the Blocked tab daily for a week.
6. Record each disabled SID and suppression in the change log.

```
awk -F'[][]' '/\]/{print $4}' /var/log/snort/snort_*/alert | sort | uniq -c | sort -rn | head -20   # top alert messages
grep -c '' /var/log/snort/snort_*/alert                                                              # alert volume per interface log
```

### Snort service control

```
/usr/local/etc/rc.d/snort.sh restart                   # restart Snort on all enabled interfaces
/usr/local/etc/rc.d/snort.sh stop                      # stop all instances (blocks already in snort2c stay until cleared/expired)
ps aux | grep '[s]nort'                                # one process per enabled interface expected
```

Rule updates: Services → Snort → Updates → "Update Rules" (or "Force Update"). Update log: Services → Snort → Updates → view log.

### Troubleshooting — Snort

#### Symptom: site/service unreachable from wired and ORG_Wireless, fine on Guest
- Check: `pfctl -t snort2c -T show | grep <ip>` # is it blocked
- Fix: unblock (above); if recurring, pass-list or suppress.

#### Symptom: one of our own servers is unreachable from outside
- Likely cause: Snort blocked the server's *public* address because "Which IP to block" is BOTH or DST.
- Check: `pfctl -t snort2c -T show | grep 203.0.113.` # is one of our VIPs in the table
- Fix: unblock, add the VIPs to the pass list, set block to SRC.

#### Symptom: Snort interface will not start after update/upgrade
- Check: Status → System Logs → Packages → Snort, or `grep -i snort /var/log/system.log | tail -30` # look for rule parse errors
- Fix: Updates → Force Update; if a specific rule file fails to parse, disable that category and restart. If the package was wiped by an upgrade, reinstall — settings return only if "Keep Snort settings after deinstall" was on.

#### Symptom: high CPU / dropped packets
- Check: `top -aSH` # snort processes at 100% of a core
- Fix: drop unneeded categories, switch IPS policy to Connectivity, choose Search Method AC-BNFA, or remove Snort from internal interfaces.

---

## Shell Reference

| Command | Purpose |
|---|---|
| `pfctl -sr` | active filter rules in evaluation order |
| `pfctl -vsr` | rules with packet/byte counters |
| `pfctl -sn` | active NAT (rdr and nat) rules |
| `pfctl -si` | pf status and counters |
| `pfctl -ss \| grep <ip>` | states for an address |
| `pfctl -k <ip>` | kill states from an address |
| `pfctl -s Tables` | list pf tables (aliases, snort2c, sshguard, bogons) |
| `pfctl -t <table> -T show` | members of a table |
| `pfctl -t sshguard -T flush` | clear sshguard lockouts |
| `pfctl -f /tmp/rules.debug` | reload the generated ruleset |
| `/etc/rc.filter_configure` | regenerate rules from config.xml and reload (what Apply does) |
| `easyrule block CARRIER_WAN 203.0.113.50` | block an address on an interface (creates/uses EasyRule block alias) — confirm interface name with `easyrule` usage output |
| `easyrule showblock CARRIER_WAN` | list addresses easyrule has blocked |
| `easyrule unblock CARRIER_WAN 203.0.113.50` | remove an easyrule block |
| `easyrule pass CARRIER_WAN tcp 203.0.113.50 10.60.1.252 443` | quick pass rule — a widening change; use the change procedure |
| `pfSsh.php playback svc restart <service>` | restart a service (ipsec, unbound, nginx, …) the way the GUI does |
| `pfSsh.php playback listpkg` | list installed packages and versions |
| `pfSsh.php playback pftabledrill` | dump every pf table and its contents |
| `pfSsh.php playback enablesshd` | enable SSH from the console if it was turned off |
| `tail -f /var/log/filter.log` | live raw firewall log |
| `tail -f /var/log/ipsec.log` | live IPsec negotiation log |
| `tcpdump -ni <if> host <ip>` | packet capture on an interface |
| `swanctl --list-sas` | IPsec SA status |
| `top -aSH` | CPU per thread, incl. snort |

`pfSsh.php playback enableallowallwan` exists and opens WAN to everything — never use it on this firewall.

---

## Backup & Restore

### What is backed up

| Mechanism | What | Where | Schedule |
|---|---|---|---|
| Config History | Last N configs (default 30) | On the firewall, `/cf/conf/backup/` | Every save |
| Auto Configuration Backup (ACB) | Encrypted config.xml, up to 100 | Netgate servers | Every change [FILL IN: is ACB enabled; which Netgate account / device key] |
| Manual download | config.xml incl. Snort settings | [FILL IN: path on IT share, and whether it is inside the Retrospect backup set] | Before every change and every upgrade |

### Manual backup

GUI: Diagnostics → Backup & Restore → Backup area **All**, tick **Encrypt this configuration file** [CONFIRM: proposal — always encrypt; password in IT vault], optionally Skip RRD data → Download.

```
cp /cf/conf/config.xml /root/config-$(date +%F).xml    # local copy on the box (not an off-box backup)
scp <admin>@<management-ip>:/cf/conf/config.xml ./config-po-$(date +%F).xml   # pull a copy off-box from an admin workstation
```

config.xml contains password hashes, the IPsec PSKs, certificates' private keys and the Snort oinkcode. Treat every unencrypted copy as a secret.

### ACB

Services → Auto Configuration Backup → Settings: enable, set the encryption password (store in the IT vault — without it, ACB backups are unrecoverable), confirm the **Device Key** and record it in the vault (needed to restore to *replacement* hardware). Restore tab lists backups by timestamp and change description.

### Restore

| Situation | Method |
|---|---|
| Undo a recent change | Config History → Revert the pre-change entry (diff first) |
| Restore a downloaded file | Diagnostics → Backup & Restore → Restore → choose area (All, or just one area such as Firewall Rules) → upload |
| GUI unreachable | Console option 15 "Restore recent configuration" |
| Replacement hardware | Fresh install → ACB restore with device key, or upload config.xml — see Disaster Recovery |

Partial restore (single area) is useful: you can restore only "Firewall Rules" or "NAT" from an old backup without touching IPsec or Snort.

---

## Upgrades with Rollback

pfSense Plus uses ZFS **Boot Environments**: an upgrade creates a snapshot of the current system you can boot back into.

1. Read the release notes and the Snort package notes for the target version.
2. Schedule outside business hours, through the change procedure. Notify that the VPN and partner/cloud VPS tunnels will drop.
3. Manual encrypted backup (All) + confirm ACB has a current entry.
4. Record the current state:

```
pfSsh.php playback listpkg > /root/pkgs-before.txt     # package list
pfctl -t snort2c -T show | wc -l                       # Snort block count baseline
swanctl --list-sas | grep -c ESTABLISHED               # tunnel baseline
```

5. System → Boot Environments → confirm a pre-upgrade BE will be created (or create one manually and name it `pre-<version>-<date>`).
6. System → Update → confirm branch (Latest Stable) → Confirm. Or from console/SSH (inside `screen` for SSH): `pfSense-upgrade`.
7. After reboot: dashboard version, Status → Gateways (both WANs up), Status → IPsec (all P1 established — the partner site initiates, give it a few minutes), Services → Snort (each interface green), Status → System Logs for errors, test a published service from outside.
8. Snort package: check it updated to the matching version and rule sets downloaded.

**Rollback:** System → Boot Environments → activate the pre-upgrade BE → reboot. If the GUI is dead, choose the BE from the boot menu at the console.

```
bectl list                                             # list boot environments (shell)
bectl activate <pre-upgrade-be-name>                   # make it the default for next boot
shutdown -r now                                        # reboot into it — console access recommended
```

[FILL IN: confirm the box boots from ZFS — `zpool list` returns pfSense. If UFS, there are no boot environments and rollback is reinstall + config restore.]

---

## Logging to Graylog

Configure: Status → System Logs → Settings → Remote Logging Options.

| Setting | Value | Reason |
|---|---|---|
| Send log messages to remote syslog server | On | Local logs rotate quickly and are lost with the box |
| Source address | LAN / server-hub interface [FILL IN] | Graylog sees a stable internal source |
| IP protocol | IPv4 | |
| Remote log servers | `graylog01.example.com:<port>` [FILL IN: Graylog syslog input port; use IP if DNS is served through the firewall] | |
| Remote syslog contents | Everything [CONFIRM] or at least System, Firewall, VPN, Packages (Snort) | Snort alerts come through as package/system log entries when "Send Alerts to System Log" is on |
| Log message format | syslog (RFC 5424) [CONFIRM: match the Graylog input type] | RFC 5424 carries full timestamps and hostname |

Rule logging: only log rules you need evidence from (default block, WAN pass rules for published services, any new permit per the change procedure). Logging the default LAN pass rule floods the log.

Snort 3 / `alert_json` / `parse_json` pipeline material in `Security Procedures/modules/detection-validation.md` describes a standalone Snort 3 sensor. The pfSense Snort package is Snort 2.9.x and ships alerts as syslog text, so `snort_*` JSON fields will **not** exist for pfSense alerts unless a Graylog extractor/pipeline parses the syslog line. [FILL IN: what pfSense alerts look like in Graylog today, and whether a parsing pipeline exists.]

Verify flow:

```
tcpdump -ni <lan_if> host <graylog01-ip> and udp port <port> -c 5   # are syslog packets leaving the firewall
logger -t pfsense-test "graylog flow test $(date)"                  # write a test line to the local syslog
```

Then in Graylog search `source:<firewall-hostname> AND pfsense-test` over the last 5 minutes.

---

## Monitoring & Alerting

Baseline today: no automated monitoring of the firewall itself (see `SysAdmin Procedures/monitoring-alerting-guide.md` gap table: "Edge firewall / IDS — pfSense + Snort — [FILL IN]").

| Check | How | Threshold / alert | Status |
|---|---|---|---|
| Gateway up/loss/latency per WAN | Status → Gateways; dpinger to monitor IP | Notify on down — System → Advanced → Notifications (SMTP) | [FILL IN: configured?] |
| IPsec P1 down | Graylog search on `/var/log/ipsec.log` messages for con1/con2 disconnect | Alert on partner or cloud VPS tunnel down > 10 min | [CONFIRM: proposal] |
| Snort not running | Graylog: absence of Snort messages for 30 min; or Services → Status | Alert | [CONFIRM: proposal] — `log_coverage.sh` lists "snort-sensor 30m" as expected |
| Snort block spike | Count of block messages per hour | > [CONFIRM: 3x baseline] | [CONFIRM: proposal] |
| Config change outside window | Graylog: `/var/log/system.log` "Configuration Change" lines | Alert on any outside an approved window | Highest-value single alert per `firewall-change-procedure.md` |
| Admin login | `/var/log/auth.log` success/failure; sshguard blocks | Alert on failures from external IPs or admin login off-hours | [CONFIRM: proposal] |
| Disk / memory / state table | Status → Dashboard; `df -h`; `pfctl -si` | Disk > 80%, states > 80% of limit | [FILL IN] |
| Snort rule updates failing | Snort update log | Alert if no successful update in 48 h | [CONFIRM: proposal] |

Normal looks like: [FILL IN: typical state count, typical snort2c table size, typical CPU — capture from the dashboard on a normal weekday].

```
pfctl -si | grep -i 'current entries'                  # current state count
pfctl -sm | grep states                                # state table hard limit
pfctl -t snort2c -T show | wc -l                       # current Snort block count
df -h /                                                # root filesystem usage
```

---

## General Troubleshooting

#### Symptom: published service unreachable from internet, works internally
- Check: `pfctl -sn | grep <public-ip>` # rdr rule present
- Check: `pfctl -t snort2c -T show | grep <public-or-client-ip>` # Snort block
- Check: `tcpdump -ni <carrier_wan_if> host <public-ip> and tcp port 443` # SYNs arriving / replies leaving
- Fix: missing rdr → NAT config; Snort → unblock; SYNs arrive but no reply → internal host/gateway; check Barracuda if it fronts the service. Also check carrier DDoS protection status.

#### Symptom: internal subnet has no internet
- Likely cause: no Manual Outbound NAT mapping for it.
- Check: `pfctl -sn | grep -E '^nat' | grep <subnet>` # mapping exists
- Fix: add outbound NAT mapping on CARRIER_WAN.

#### Symptom: locked out of GUI
- Check from console: `pfctl -t sshguard -T show` # your IP listed
- Fix: `pfctl -t sshguard -T flush`; or console option 2 to re-enable the anti-lockout rule; option 11/16 to restart GUI/PHP-FPM.

#### Symptom: block rule added, attacker still connected
- Fix: `pfctl -k <ip>` — rules only apply to new states.

---

## Security Hardening of the Firewall

| Control | Setting | Reason |
|---|---|---|
| WAN management | No GUI/SSH on CARRIER_WAN or VPN2_WAN; confirm no WAN rule permits 443/22 to "This Firewall" | Netgate's own guidance; GUI exposure is the #1 pfSense compromise path |
| Anti-lockout rule | Disable once an `ORG_Admin_Hosts` management rule exists (System → Advanced → Admin Access) [CONFIRM] | Anti-lockout allows the whole LAN to reach the GUI |
| GUI certificate | Valid internal or public cert for the management name, not the self-signed default (partner-site screenshot shows "Not Secure") | Trains admins not to ignore certificate warnings |
| HTTPS only, HSTS | System → Advanced → Admin Access | |
| SSH | Enabled only if used; "Public Key Only" [CONFIRM]; keys per admin | Removes password guessing |
| sshguard | Default thresholds; whitelist admin hosts in Login Protection pass list | Brute-force protection for GUI + SSH |
| Accounts | Named admins, built-in `admin` disabled or vault-only, bcrypt hashing, session timeout ≤ 240 min | Accountability and audit trail |
| MFA | Via RADIUS (auth2/auth4) [CONFIRM] | No native GUI TOTP |
| Packages | Only what is used (Snort, ACB, WireGuard if present) | Smaller attack surface |
| DNS rebind / referer checks | Leave enabled | Protects GUI from browser-based attacks |
| Console | Password-protect console menu (System → Advanced → Admin Access → Console Options) | Physical access ≠ automatic admin |
| Bogons / private on WAN | Block private networks and bogons enabled on CARRIER_WAN | Cheap spoofing protection |
| NTP | Sync to reliable sources | Log correlation with Graylog |
| Secrets | PSKs, oinkcode, ACB password, device key, admin passwords in the IT vault only | See `Security & Hardening/secrets-management-guide.md` |
| Review | Quarterly rule review + Snort suppression review per `firewall-change-procedure.md` | Rulesets only grow |

Check from outside that nothing management-related answers:

```
nmap -Pn -p 22,80,443,8443 203.0.113.179-186          # from an external host — only intended published ports should be open
nmap -Pn -sU -p 500,4500 198.51.100.52                # IKE expected open on VPN2_WAN only
```

---

## Disaster Recovery

| Item | Value |
|---|---|
| RPO | Last config backup — ACB makes this "last change" if enabled |
| RTO | [CONFIRM: 4 hours to a working firewall on replacement hardware] |
| Spare hardware | [FILL IN: is there a spare appliance or a VM plan? model, location] |
| Escalation | [FILL IN: Netgate TAC entitlement, carrier business support number, backup admin] |

**Rebuild from config.xml:**

1. Install pfSense Plus on replacement hardware at the **same or newer** version (Netgate installer; Plus requires the hardware to be registered/entitled — [FILL IN: Netgate account]).
2. At the console, assign at least WAN and LAN so the GUI is reachable from a laptop on LAN.
3. Restore: either Services → Auto Configuration Backup → Restore using the old **device key** (from the vault) and ACB password, or Diagnostics → Backup & Restore → upload config.xml (decrypt with the password from the vault).
4. On different hardware the NIC names change (e.g. `igb0` → `ix0`). The console will prompt for interface reassignment — map CARRIER_WAN, VPN2_WAN, LAN and the VLAN parent correctly. [FILL IN: record current NIC → interface mapping here: `ifconfig -l` and Interfaces → Assignments.]
5. Packages reinstall automatically after restore (needs internet). Watch System → Package Manager; Snort reinstalls with its settings from config.xml; trigger Services → Snort → Updates → Force Update.
6. Verify: Gateways, every published service from outside, IPsec (the partner site will re-initiate; cloud VPS con2 — initiate if needed), mobile VPN, inter-VLAN, Snort running, remote syslog arriving in Graylog.
7. If the WAN public IP changed (unlikely with the same carrier circuit): update the cloud VPS and partner peer addresses and public DNS.
8. Record the incident and the rebuild time; feed lessons into this section.

Full-site loss sequence (firewall → switches → controllers) is in `ipam-vlan-topology-reference.md` → Disaster Recovery.

---

## Decisions & History (ADR-lite)

| Date | Decision / Change | Why / Source |
|---|---|---|
| [FILL IN] | Outbound NAT set to Manual with per-host /32 mappings to .179–.186 | Observed in `pfsense_outbound_nat_mappings.jpeg`; reason [FILL IN] |
| 2024-02-14 | Snort unblock KB written (fw01.example.com) | `Unblocking an IP from Snort` PDF |
| 2025-02-11 | `Venue` alias added to Snort "ORG custom pass list" for large meetings | Large Meeting Whitelist KB |
| 2026-10-04 | This guide created; notes that `clog`/`ipsec statusall` in older notes are legacy on current pfSense Plus | New draft |

## References

- `Networking Guide/firewall-change-procedure.md` — how every widening change here is made, verified and logged
- `Networking Guide/Network-Administration-Guide.md` — pfSense mechanics, console menu (§6), backup (§7.1), upgrades (§7.4), IKEv2 setup (§5), MTU (§3.2)
- `Networking Guide/pfsense.txt` — GUI/CLI cheat sheet (note legacy `clog` and `ipsec statusall`)
- `Networking Guide/ipam-vlan-topology-reference.md` — VLAN, subnet, public IP and NAT tables to populate from this guide
- `[internal KB: network subnet list]` — subnet list
- `Networking Guide/dns-dhcp-administration-guide.md` — Unbound resolver and DHCP on pfSense
- `Networking Guide/pfsense_ipsec_tunnels_config.jpeg`, `pfsense_ipsec_status_overview.jpeg`, `pfsense_outbound_nat_mappings.jpeg`, `pfsense_port_forward_https_rule.jpeg`, `firewall_nat_port_forward_edit.jpeg`, `firewall_destination_ip_config.jpeg`, `carrier_wan_tcp_https_rule.jpeg` — source screenshots
- `Networking Guide/the-pfsense-documentation.pdf` — Netgate manual
- `[internal KB: large meeting IP whitelist prep]` — Venue alias, fail2ban/Zimbra whitelists, carrier DDoS suspension
- `[internal KB: unblocking an IP from Snort]` — Snort unblock KB
- `[internal KB: partner VPN jump-host procedure]` — partner jump-host procedure
- `SysAdmin Procedures/monitoring-alerting-guide.md` — Graylog inputs and monitoring gap analysis
- `Security Procedures/modules/detection-validation.md` — Snort → Graylog detection chain (Snort 3 example)
- `Security Procedures/ransomware-response-playbook.md` — firewall role in isolation and recovery
- `Mail & Messaging/zimbra-mail-administration-guide.md` — mail NAT ports and Snort/fail2ban interplay
- `Linux & Servers/reverse-proxy-and-tls-automation-guide.md` — publishing web services through pfSense
- `Security & Hardening/secrets-management-guide.md` — where PSKs, oinkcode, ACB keys live
- `Cheat Sheets/Security & Forensics/Snort_Master_Cheatsheet.txt` — Snort rule syntax
- Netgate docs: https://docs.netgate.com/pfsense/en/latest/ (Snort package, ACB, Boot Environments, Remote Logging)

## Change log

| Date | Author | Change |
|---|---|---|
| 2026-10-04 | IT lead (it@example.com) | Initial draft from library sources and screenshots — UNVERIFIED |

---
*Tier 3 document. Review annually, after any pfSense/Snort upgrade, or after any material firewall change.*

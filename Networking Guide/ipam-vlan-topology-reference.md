> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# IPAM, VLAN and Topology Reference (example.com)

## Overview

This is the reference document for how addresses and VLANs are allocated on the organization's network: which VLAN is which, what subnet it uses, what gateways and DHCP scopes belong to it, which infrastructure hosts hold static addresses, what public addresses exist and what they NAT to, and how the physical topology is laid out. It exists so that nobody has to reverse-engineer the address plan from a firewall ruleset at 2am, and so that allocating a new subnet does not collide with something already in use.

This is deliberately a reference rather than a narrative guide. Most of it is tables. The tables are structured and complete; the values are marked `[FILL IN]` because they must be read off the live equipment rather than guessed. The "Discovering Current State" section at the end gives the exact commands to populate every table here. **Populating this document is a discovery exercise, not a writing exercise** — run the commands, record what is actually there, and treat any surprise as a finding worth investigating rather than a typo to smooth over.

## Quick Facts

| Field            | Value                                                                                         |
|------------------|-----------------------------------------------------------------------------------------------|
| Owner            | IT Ops (it@example.com)                                                    |
| Environment      | prod                                                                                           |
| Location         | Authoritative config sources: pfSense edge firewall (VLAN interfaces, DHCP, NAT, firewall rules); NetGear M5300 switch stack(s) and Ubiquiti EdgeSwitch (L2 VLAN membership, trunks); Omada controller (omada.example.com) and UniFi controller (wireless VLAN mapping). This document is a transcription of those, not a source of truth in itself. |
| Access           | pfSense GUI and SSH from the management network or over WireGuard; switch web GUI / SSH / serial console; omada.example.com and the UniFi controller over the internal network — [FILL IN: management network address range and jump path, if any] |
| Dependencies     | Accurate transcription depends on access to every device listed above. Nothing in here is generated automatically. |
| Dependents       | DNS/DHCP administration, firewall change work, new service deployment, 802.1x rollout, wireless SSID-to-VLAN mapping, incident triage |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                                                       |

## How It Works

The network is segmented into VLANs. Layer 2 tagging and trunking are handled by the switches — NetGear M5300 series stacks and Ubiquiti EdgeSwitch units. Layer 3 routing between VLANs, NAT to the public internet, and the firewall policy that governs which VLAN may talk to which, all live on the pfSense edge firewall. Each VLAN that carries clients has a gateway address on pfSense and, normally, a DHCP scope.

Wireless is delivered by two controller-managed systems in parallel: Ubiquiti UniFi and TP-Link Omada (omada.example.com). Each SSID is mapped to a VLAN by its controller, and that mapping is a piece of network configuration that lives outside the switches and the firewall — which is exactly why it gets forgotten. It is recorded in the SSID table below.

Wired and wireless authentication uses 802.1x with RADIUS against Active Directory (auth2.example.com, auth4.example.com). On an 802.1x-enabled port or SSID, the VLAN a client lands in may be assigned dynamically by RADIUS rather than statically by the port's PVID. Where that is the case, the static port map below is not the whole story, and the RADIUS policy has to be read alongside it.

Remote access arrives by two paths: WireGuard as the site VPN, and a partner IPsec tunnel reached through a jump host. Both introduce address space and routes that must not collide with internal subnets, so both are recorded here.

Three terms are used precisely throughout, matching the switch configuration semantics recorded in `Networking Guide/Switch Port to Server VLAN/SwitchConfiguration.txt`:

- **PVID** — the VLAN an untagged frame arriving on this port is placed into.
- **VLAN Member** — the set of VLANs this port belongs to at all.
- **VLAN Tag** — the subset of those VLANs for which this port sends and accepts *tagged* frames.

An access port is typically a member of one VLAN, with that VLAN as its PVID and no tagging. A trunk port is a member of several, tags all or most of them, and has a PVID for whatever is left untagged. Getting these three confused is the most common cause of "the port links up but nothing works."

---

## VLAN Table

The master table. One row per VLAN. Everything else in this document cross-references this.

| VLAN ID | Name | Subnet (CIDR) | Gateway (pfSense interface IP) | pfSense interface name | Purpose | DHCP scope | DHCP served by | 802.1x | Notes |
|---|---|---|---|---|---|---|---|---|---|
| [FILL IN] | [FILL IN: e.g. Staff] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN: range, or "none — static only"] | [FILL IN] | [FILL IN: yes/no] | |
| [FILL IN] | [FILL IN: e.g. Server] | [FILL IN] | [FILL IN] | [FILL IN] | Hosts in the server room | [FILL IN] | [FILL IN] | [FILL IN] | Referenced by `Putting a switch port into Server VLAN/` |
| [FILL IN] | [FILL IN: e.g. Voice] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Check for provisioning DHCP options |
| [FILL IN] | [FILL IN: e.g. Guest Wireless] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Should be isolated from all internal VLANs |
| [FILL IN] | [FILL IN: e.g. Management] | [FILL IN] | [FILL IN] | [FILL IN] | Switch, AP, and firewall management | [FILL IN] | [FILL IN] | [FILL IN] | Should be reachable only from admin workstations / VPN |
| [FILL IN] | [FILL IN: e.g. Printers] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| [FILL IN] | [FILL IN: e.g. Cameras / physical security] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| [FILL IN] | [FILL IN: remaining VLANs discovered] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |

**VLAN 1 / the native VLAN.** Record explicitly what VLAN 1 is used for here, even if the answer is "nothing deliberate." The switch config notes in the library show ports with `PVID 1` and `VLAN Member 1,4,5`, so VLAN 1 is carrying at least untagged traffic somewhere. [FILL IN: what is VLAN 1's role, and is any production traffic in it?]

**VLANs defined but unused.** A VLAN that exists on the switches but has no pfSense interface and no scope is a trap — it will link up and silently blackhole. List them: [FILL IN: any VLAN IDs configured on switches with no L3 presence]

---

## Subnet Allocation Table

Every subnet in use anywhere, including ones that are not VLANs — VPN pools, tunnel transit networks, point-to-point links. The purpose of this table is collision avoidance: before allocating anything new, it must be checked against this list in full.

| Subnet (CIDR) | Usable range | Allocated to | Type | Routed by | Reserved band (non-DHCP) | Free capacity | Notes |
|---|---|---|---|---|---|---|---|
| [FILL IN] | [FILL IN] | [FILL IN: VLAN name/ID] | VLAN | pfSense | [FILL IN: e.g. .1–.20 for statics] | [FILL IN] | |
| [FILL IN] | [FILL IN] | [FILL IN: VLAN name/ID] | VLAN | pfSense | [FILL IN] | [FILL IN] | |
| [FILL IN] | [FILL IN] | WireGuard client pool | VPN | pfSense (WireGuard) | n/a | [FILL IN] | Must not overlap any internal subnet |
| [FILL IN] | [FILL IN] | partner IPsec — local selector | VPN | pfSense | n/a | n/a | Phase 2 local network |
| [FILL IN] | [FILL IN] | partner IPsec — remote selector | VPN | remote peer | n/a | n/a | Phase 2 remote network |
| [FILL IN] | [FILL IN] | [FILL IN: IPsec mobile client pool, if mobile IKEv2 is still in use] | VPN | pfSense | n/a | [FILL IN] | |
| [FILL IN] | [FILL IN] | [FILL IN: WAN transit / ISP-assigned link network] | WAN | ISP | n/a | n/a | |
| [FILL IN] | [FILL IN] | [FILL IN: any lab, test, or legacy subnet] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |

**Supernet / overall plan.** [FILL IN: is there an overarching allocation scheme — e.g. one /16 carved into /24s per VLAN, with a convention for which octet means what? Recording the *scheme* matters more than the individual rows, because it tells the next person where a new subnet should go.]

**Addresses never to allocate.** [FILL IN: any ranges that must stay clear — remote partner networks reachable over the partner tunnel, ranges used by a vendor appliance, home-network ranges that VPN clients commonly use (192.168.0.0/24 and 192.168.1.0/24 are the usual offenders and will break split-tunnel VPN clients who have them at home)]

---

## Static IP Registry — Infrastructure

Hosts and devices with addresses configured on the device itself, not handed out by DHCP. DHCP reservations do **not** belong here — they belong in the reservation table in `dns-dhcp-administration-guide.md`. The distinction matters: a static address is invisible to the DHCP server and will be handed out to someone else if the subnet's reserved band is not respected.

### Servers and services

| Hostname | IP address | VLAN | Role / service | OS / platform | Public DNS? | Cert? | Notes |
|---|---|---|---|---|---|---|---|
| data.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | MySQL host per `Databases/MySQL Set up/` |
| wiki.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | MySQL host per `Databases/MySQL Set up/` |
| sign.example.com | [FILL IN] | [FILL IN] | Document signing (DocuSeal) | [FILL IN] | [FILL IN] | [FILL IN] | |
| appdb2.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | MySQL host per archived setup notes |
| appdb1.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Per archived setup notes |
| erpdb.example.com | [FILL IN] | [FILL IN] | DMS database | [FILL IN] | [FILL IN] | [FILL IN] | See `Dynamics GP & DMS/` |
| prod01.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| graylog01.example.com | [FILL IN] | [FILL IN] | Log aggregation | [FILL IN] | [FILL IN] | [FILL IN] | Receives syslog from network gear |
| files.example.com | [FILL IN] | [FILL IN] | File services | [FILL IN] | [FILL IN] | [FILL IN] | |
| forums.example.com | [FILL IN] | [FILL IN] | phpBB forums | [FILL IN] | [FILL IN] | [FILL IN] | See `Linux & Servers/phpbb update.txt` |
| auth2.example.com | [FILL IN] | [FILL IN] | RADIUS / auth | [FILL IN] | [FILL IN] | [FILL IN] | 802.1x backend |
| auth4.example.com | [FILL IN] | [FILL IN] | RADIUS / auth | [FILL IN] | [FILL IN] | [FILL IN] | Current Omada RADIUS target per `Security & Hardening/Change Radius for WIfi.txt` |
| mail.example.com | [FILL IN] | [FILL IN] | Mail | [FILL IN] | [FILL IN] | [FILL IN] | fail2ban / Graylog unban procedures exist in `Security & Hardening/` |
| [FILL IN: domain controllers] | [FILL IN] | [FILL IN] | AD DS + DNS | [FILL IN] | no | [FILL IN] | Internal only |
| [FILL IN: hypervisor hosts — Proxmox nodes per `Linux & Servers/lxc_backup_restore_proxmox91.txt`] | [FILL IN] | [FILL IN] | Virtualization | [FILL IN] | no | [FILL IN] | |
| [FILL IN: backup / Retrospect host, Synology] | [FILL IN] | [FILL IN] | Backup | [FILL IN] | no | [FILL IN] | See `Hardware & Backup/`, `Retrospect Restore/` |
| [FILL IN: remaining servers] | | | | | | | |

### Network infrastructure

| Device | Management IP | VLAN | Type / model | Location | Role | Notes |
|---|---|---|---|---|---|---|
| [FILL IN: pfSense hostname] | [FILL IN] | [FILL IN] | pfSense | [FILL IN: rack/room] | Edge firewall, inter-VLAN routing, NAT, WireGuard, partner IPsec | Snort IDS and fail2ban run here |
| [FILL IN: core switch name] | [FILL IN] | [FILL IN] | NetGear M5300 | [FILL IN] | Core / distribution | |
| [FILL IN: stack member names] | [FILL IN] | [FILL IN] | NetGear M5300 | [FILL IN: e.g. "West stack on 7" appears in the switch config notes] | [FILL IN] | Stacked — record stack member IDs |
| [FILL IN: edge switch names] | [FILL IN] | [FILL IN] | Ubiquiti EdgeSwitch | [FILL IN] | Access layer | Full standalone CLI — see `Network-Administration-Guide.md` §9 |
| omada.example.com | [FILL IN] | [FILL IN] | TP-Link Omada controller | [FILL IN] | Wireless controller | Main office site selected via location menu top-left |
| [FILL IN: UniFi controller host] | [FILL IN] | [FILL IN] | Ubiquiti UniFi | [FILL IN] | Wireless controller | |
| [FILL IN: partner jump host] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Jump host to partner IPsec | Reached via TeamViewer web client |
| [FILL IN: remaining devices — UPS, PDU, out-of-band] | | | | | | |

### Wireless access points

Record every AP. Do not guess names — pull the list from each controller.

| AP name | Management IP | VLAN | Model | Controller | Physical location | Switch / port | PoE | Notes |
|---|---|---|---|---|---|---|---|---|
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN: UniFi / Omada] | [FILL IN] | [FILL IN] | [FILL IN] | |
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |

### SSID to VLAN mapping

| SSID | Controller | VLAN | Auth method | RADIUS profile / server | Client isolation | Notes |
|---|---|---|---|---|---|---|
| [FILL IN: e.g. ExampleCorp-WiFi] | Omada | [FILL IN] | 802.1x (WPA2/3-Enterprise) | auth4.example.com | [FILL IN] | RADIUS profile switched from auth.example.com to auth4.example.com — see `Security & Hardening/Change Radius for WIfi.txt` |
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| [FILL IN: guest SSID] | [FILL IN] | [FILL IN] | [FILL IN: PSK / portal] | n/a | [FILL IN: should be yes] | Must not reach internal VLANs |

---

## Public IP and NAT Mapping

### Public address inventory

| Public IP | Assigned by | Bound to | Reverse DNS (PTR) | In use for | Notes |
|---|---|---|---|---|---|
| [FILL IN] | [FILL IN: ISP] | pfSense WAN | [FILL IN] | [FILL IN] | Primary WAN |
| [FILL IN] | [FILL IN] | [FILL IN: pfSense Virtual IP, if additional publics exist] | [FILL IN] | [FILL IN] | |
| [FILL IN: secondary WAN / failover circuit, if one exists] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |

**PTR records** for public addresses are set by the ISP, not in the organization's DNS. Mail delivery from mail.example.com depends on the PTR matching the HELO name. [FILL IN: confirm PTR for the address mail.example.com sends from, and who at the ISP changes it]

### Inbound NAT / port forwards

One row per port forward. This table is the answer to "what of ours is exposed to the internet," so it needs to be complete.

| Public IP | Proto | Public port | Internal host | Internal port | Service | Source restriction | Firewall rule / alias | Notes |
|---|---|---|---|---|---|---|---|---|
| [FILL IN] | TCP | 443 | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN: any / alias name] | [FILL IN] | |
| [FILL IN] | TCP | 80 | [FILL IN] | [FILL IN] | HTTP → redirect / ACME HTTP-01 | any | [FILL IN] | Required if Let's Encrypt HTTP-01 validation is used |
| [FILL IN] | TCP | 25 | [FILL IN] | [FILL IN] | SMTP inbound | [FILL IN] | [FILL IN] | |
| [FILL IN] | UDP | [FILL IN] | [FILL IN] | [FILL IN] | WireGuard | any | [FILL IN] | |
| [FILL IN] | UDP | 500, 4500 + ESP | [FILL IN] | n/a | partner IPsec | [FILL IN: restrict to peer IP] | [FILL IN] | Should be restricted to the remote peer address |
| [FILL IN: remaining forwards] | | | | | | | | |

### 1:1 NAT

| Public IP | Internal IP | Host | Reason a 1:1 was used rather than a port forward | Notes |
|---|---|---|---|---|
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |

### Outbound NAT

| Mode | Source | Translated to | Reason | Notes |
|---|---|---|---|---|
| [FILL IN: Automatic / Hybrid / Manual] | [FILL IN] | [FILL IN] | [FILL IN] | The library contains a `pfsense_outbound_nat_mappings` screenshot — transcribe it here |

**Precedence reminder**: inbound port forward beats 1:1 NAT; outbound 1:1 beats outbound NAT. Modes are Automatic / Hybrid / Manual / Disabled.

---

## Inter-VLAN Routing and Firewall Policy Summary

pfSense routes between all VLANs it has interfaces on. Whether traffic is *permitted* is entirely a matter of the per-interface firewall rules. This section summarizes the intended policy so that an actual ruleset can be checked against an intent; it is not a substitute for reading the rules.

### Intended policy matrix

Rows are sources, columns are destinations. Fill each cell with Allow / Deny / Limited (and note which ports for Limited).

| From ↓ / To → | Staff | Server | Voice | Printers | Management | Guest | Internet |
|---|---|---|---|---|---|---|---|
| Staff | — | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN: should be deny or tightly limited] | [FILL IN: deny] | [FILL IN] |
| Server | [FILL IN] | — | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN: deny] | [FILL IN: limited — many servers need only update repos] |
| Voice | [FILL IN] | [FILL IN] | — | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Printers | [FILL IN] | [FILL IN] | [FILL IN] | — | [FILL IN] | [FILL IN] | [FILL IN: usually deny] |
| Management | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | — | [FILL IN] | [FILL IN] |
| Guest | [FILL IN: deny] | [FILL IN: deny] | [FILL IN: deny] | [FILL IN: deny] | [FILL IN: deny] | — | Allow |
| WireGuard clients | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN: deny] | [FILL IN: split tunnel — internal only] |
| Partner site (over IPsec) | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN: deny] | [FILL IN] |

Adjust the column set to match the actual VLAN table above. Where the real ruleset does not match the intent, that is a finding — record it rather than editing the intent to match.

### Notable rules and exceptions

| Rule / alias | Interface | Purpose | Why it exists | Risk if removed |
|---|---|---|---|---|
| [FILL IN: alias name, e.g. a management-hosts alias] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: any floating rules] | Floating | [FILL IN] | [FILL IN] | Floating rules run after NAT and can shadow interface rules — document carefully |
| [FILL IN: any deliberate any-any rules] | [FILL IN] | [FILL IN] | [FILL IN] | |

Rule evaluation order on pfSense, for reference when a rule appears not to apply: Ethernet rules → Outbound NAT → Inbound NAT → automatic/internal rules → user rules (Floating, then interface-group, then interface tabs, top-down first-match) → automatic VPN rules. A rule higher up silently shadows one below it.

### Aliases in use

| Alias name | Type | Contents | Used by which rules | Notes |
|---|---|---|---|---|
| [FILL IN] | [FILL IN: Host/Network/Port/URL Table] | [FILL IN] | [FILL IN] | |

### IDS / IPS scope

Snort runs on pfSense. Record which interfaces it is enabled on and in what mode, because a blocked host here presents identically to a routing failure.

| Interface | Snort enabled | Mode | Rule sets | Notes |
|---|---|---|---|---|
| [FILL IN] | [FILL IN] | [FILL IN: IDS / inline IPS] | [FILL IN] | Unblocking procedure: the internal KB article "Unblocking an IP from Snort" |

---

## Physical Topology

Describe the actual cabling and device layout. This section is prose plus a small number of tables because the useful information — which closet feeds which floor, which uplink is the one that takes everything down — does not fit a matrix well.

### Edge and core

The internet circuit from [FILL IN: ISP] terminates at [FILL IN: modem/CPE model and location] and hands off to the pfSense WAN interface. pfSense is located in [FILL IN: rack and room]. It is the only routing device between VLANs and the only path to the internet.

pfSense connects to the core switch over [FILL IN: single link or LAG? which physical ports on each end?]. This link is a trunk carrying every VLAN that pfSense gateways. **[FILL IN: is this link redundant? If it is a single cable, it is the single point of failure for the entire network and that should be recorded explicitly.]**

The core is [FILL IN: which switch or stack — the NetGear M5300 stack, per the library's switch config notes which reference a "West stack on 7"]. Record the stack composition:

| Stack member | Model | Serial | Stack ID | Uplinks to | Notes |
|---|---|---|---|---|---|
| [FILL IN] | NetGear M5300 | [FILL IN] | [FILL IN] | [FILL IN] | |
| [FILL IN] | NetGear M5300 | [FILL IN] | [FILL IN] | [FILL IN] | |

### Access layer and closets

| Closet / location | Switch(es) | Model | Uplink to | Uplink ports | VLANs trunked | Serves | Notes |
|---|---|---|---|---|---|---|---|
| [FILL IN: e.g. 7th floor west] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |

**Do not populate a full per-port map here unless it is genuinely maintained.** A stale port map is worse than none — it will be trusted and it will be wrong. If a port map is kept, keep it generated from the switch rather than hand-written; the discovery commands below produce it. Record instead the ports that matter and would not be obvious: uplinks, trunks, server ports, AP ports.

| Switch | Port | Role | PVID | VLAN Member | VLAN Tag | Connected to | Notes |
|---|---|---|---|---|---|---|---|
| [FILL IN] | [FILL IN] | Uplink to core | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| [FILL IN] | [FILL IN] | AP | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | PoE |
| [FILL IN] | [FILL IN] | Server | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |

### Server room

[FILL IN: describe the server room — which rack holds what, where the hypervisor hosts sit, where storage and backup targets are, how the server VLAN reaches them, and whether there is a separate management/out-of-band path.]

Specific things worth recording because they are not discoverable from a config file:

- Which physical ports on which switch are in the Server VLAN, and the procedure to add one (see `Networking Guide/Switch Port to Server VLAN/SwitchConfiguration.txt`)
- PoE budget per switch versus actual draw — a switch at its PoE ceiling drops APs and phones without any obvious link failure
- UPS coverage: [FILL IN: which devices are on UPS, runtime, and what is *not* protected]
- Out-of-band / console access: [FILL IN: is there a console server, or is serial access physical-only? 115200 8N1 no flow control on both pfSense and Ubiquiti gear]

### Wireless coverage

[FILL IN: describe AP placement by floor/area, which controller manages which APs, and any known coverage gaps. The AGM wireless survey material in `WifiSurvey/` is the reference for survey methodology.]

### Remote access paths

- **WireGuard** — the site VPN. Terminates on pfSense. Client pool: see the subnet allocation table. Split-tunnel behaviour and allowed-IPs configuration determine what clients can reach. See `WireGuard/`.
- **partner IPsec** — reached through a jump host: log in via the TeamViewer web client (not the macOS client), connect to `partner-jump`, then manage the tunnel from the pfSense at that side under Status → VPN → IPsec → Overview. The office side is `partner-static` and can be connected from there the same way, though during an outage the office side may not be reachable. [FILL IN: the local and remote Phase 2 selectors, recorded in the subnet table above]

---

## Operations (Day-2)

### Allocating a new subnet or VLAN

Follow this in order. The verification steps are not optional — a half-created VLAN is a port that links up and blackholes, which is much harder to diagnose than a port that is obviously dead.

**1. Justify and scope.** What is the VLAN for, what will live in it, roughly how many addresses, and what must it be able to reach? If the answer is "it needs to reach everything," it probably should not be a separate VLAN.

**2. Pick the VLAN ID and subnet.** Check the VLAN table and the subnet allocation table above — *both*, in full. Check the VPN pools and the partner remote selectors too; a new subnet that overlaps a remote network over the IPsec tunnel will break routing in a way that looks like a firewall problem. Follow the existing allocation scheme if there is one.

**3. Back up first.**

```
# pfSense: Diagnostics → Backup & Restore → download config.xml before starting
copy system:running-config nvram:startup-config          # EdgeSwitch: save current running config
copy nvram:startup-config tftp://<server>/<path>/<file>   # EdgeSwitch: copy it off-box
```

**4. Create the VLAN on pfSense.** Interfaces → Assignments → VLANs tab → Add. Set Parent Interface (the physical port trunking to the core), VLAN Tag Type C-Tag/0x8100 (standard), and the VLAN Tag. Save. Then Interfaces → Assignments → add the new VLAN as an interface, enable it, and set a static IPv4 address — this becomes the gateway.

**5. Create the VLAN on the switches.** Every switch that must carry it needs the VLAN defined, and every trunk between those switches needs it added to the tagged set. Missing it on one intermediate trunk is the classic cause of "it works in one closet but not the other."

EdgeSwitch:

```
configure
vlan database
vlan 30                               # create VLAN 30
exit
interface 0/1
vlan participation include 30         # port is a member of VLAN 30
vlan tagging 30                       # port carries VLAN 30 tagged (trunk)
exit
exit
show vlan                             # confirm it exists
```

For an access port instead of a trunk:

```
interface 0/5
vlan participation include 30         # member of VLAN 30
vlan pvid 30                          # untagged frames land in VLAN 30
```

On the NetGear M5300, the equivalent is set per port as PVID / VLAN Member / VLAN Tag. Per the note in `Networking Guide/Switch Port to Server VLAN/SwitchConfiguration.txt`, on at least some firmware you must flip the view to **All** before **MAC**, or the setting reverts — and the same source notes it can now be done entirely in Port PVID. [FILL IN: confirm the current M5300 firmware behaviour and exact GUI path]

**6. Create the DHCP scope** (if the VLAN carries clients). Services → DHCP Server → the new interface tab. Set range, gateway, DNS servers, domain name (`example.com`), and lease time. Leave a reserved band outside the range for statics and reservations. Record the scope in `dns-dhcp-administration-guide.md`.

**7. Write the firewall rules.** A new pfSense interface has **no rules**, which means it denies everything — including DHCP to its own server in some configurations, and including access to its own gateway. Write the rules deliberately against the policy matrix above. Resist the temptation to start with an any-any rule "just to test," because it will still be there in two years.

**8. Add DNS.** Create the reverse lookup zone for the new subnet on the DCs so PTRs resolve. Add any host records needed.

**9. Verify, in this order:**

```
# On a test client in the new VLAN:
ipconfig /all                                    # correct IP, mask, gateway, DNS, domain suffix
ping <new-gateway-ip>                            # can reach its own gateway — if not, it is L2/VLAN, not firewall
ping <an-allowed-internal-host>                  # inter-VLAN routing + firewall rule working
nslookup <internal-name>                         # DNS reachable and answering
ping 1.1.1.1                                     # outbound NAT working
nslookup <external-name>                         # external resolution working
```

Then verify the *deny* side — confirm the new VLAN cannot reach what it should not. A VLAN that works is only half-tested.

```
# From the test client, attempt something that should be blocked:
ping <a-host-that-should-be-unreachable>         # expect failure
```

On pfSense, watch it being blocked: Status → System Logs → Firewall, filtered to the new interface.

**10. Document.** Add rows to the VLAN table, subnet allocation table, policy matrix, and any port map here. Add the scope to the DNS/DHCP guide. Add an entry to the change log below. A VLAN created and not recorded is a VLAN that will be rediscovered by accident.

### Decommissioning a VLAN or subnet

Reverse order, and slower. Confirm nothing is using it (check the ARP table, the DHCP lease table, and the switch MAC table over a period of days, not minutes), remove firewall rules, remove the DHCP scope, remove the pfSense interface, remove it from switch trunks, remove the reverse DNS zone, then release the subnet in the allocation table. Leave the row in the table marked as released with a date rather than deleting it — reusing a subnet too soon causes stale-route and stale-ARP problems.

### Changing an existing subnet

Do not. Renumbering a live VLAN touches DHCP, static hosts, firewall rules, NAT, VPN selectors, reverse DNS, and anything with the old address hardcoded. If it is unavoidable, plan it as a project with a maintenance window, not as a change.

---

## Discovering Current State

Run these to populate the tables above. Record the raw output somewhere before transcribing — the raw output is evidence, the table is interpretation.

### pfSense

GUI paths that give the fastest complete picture:

- **VLANs**: Interfaces → Assignments → VLANs tab (every VLAN and its parent interface)
- **Interfaces and gateways**: Interfaces → Assignments, then each interface for its IP
- **DHCP scopes**: Services → DHCP Server (one tab per interface)
- **Static mappings**: Services → DHCP Server → [interface] → Static Mappings
- **NAT**: Firewall → NAT → Port Forward / 1:1 / Outbound
- **Firewall rules**: Firewall → Rules, every tab including Floating
- **Aliases**: Firewall → Aliases
- **VPN**: VPN → WireGuard (tunnels, peers, allowed IPs); VPN → IPsec → Tunnels (Phase 1/2 selectors for the partner)
- **Virtual IPs**: Firewall → Virtual IPs (additional public addresses)

CLI, from the pfSense shell:

```
ifconfig                                    # every interface including VLAN sub-interfaces and their IPs
netstat -rn                                 # full routing table — shows every subnet pfSense knows about
arp -a                                      # live hosts per subnet; good for spotting undocumented statics
pfctl -sr                                   # the active ruleset as PF sees it, after all GUI translation
pfctl -sn                                   # active NAT rules
cat /var/dhcpd/var/db/dhcpd.leases          # ISC DHCP leases (ISC backend only)
cat /cf/conf/config.xml                     # the complete configuration — the definitive source
swanctl --list-conns                        # IPsec connections and their traffic selectors (partner)
swanctl --list-sas                          # active IPsec SAs
wg show                                     # WireGuard interfaces, peers, and allowed IPs
```

`cat /cf/conf/config.xml` is the single most complete source — every VLAN, interface, scope, rule, NAT entry, and alias is in it. It is large; pull it off-box and read it locally rather than paging through it on the console.

### EdgeSwitch (Ubiquiti, standalone CLI)

```
enable                                      # Privileged EXEC
show vlan                                   # every configured VLAN
show vlan id <vlan-id>                      # which ports belong to one VLAN, tagged vs untagged
show interfaces status                      # link state, speed, duplex per port
show mac-addr-table                         # what MAC is on which port — maps devices to ports
show ip interface                           # management addressing
show running-config                         # complete config; the definitive source for this switch
show poe status                             # PoE budget vs draw — needed for the AP table
show port-channel all                       # LAGs, i.e. which uplinks are aggregated
show spanning-tree                          # root bridge and port roles — tells you the real L2 topology
cablestatus 0/<port>                        # per-port copper cable test, if a physical fault is suspected
```

`show running-config` is the one to capture in full. Save it off-box:

```
copy nvram:startup-config tftp://<server>/<path>/<file>    # archive the config
```

### NetGear M5300

The M5300 runs NetGear's managed-switch firmware with both a web GUI and a CLI. [FILL IN: confirm whether the CLI is enabled and reachable on these units, and the exact command syntax for this firmware version — it is broadly Cisco-IOS-like but not identical to EdgeSwitch.]

Likely equivalents to verify:

```
show vlan                                   # VLAN list
show vlan port all                          # per-port PVID and membership
show interfaces status all                  # link state per port
show mac-address-table                      # MAC-to-port mapping
show running-config                         # complete config
```

Via the web GUI, the per-port VLAN settings are found under the switching/VLAN section, where each port shows PVID, VLAN Member, and VLAN Tag as three separate fields. The library note for this switch records that you may need to flip the view to **All** before **MAC**, and that PVID can now be set directly in the Port PVID view.

### Omada (omada.example.com)

1. Select the **main office site** in the location menu, top-left. Everything below is scoped to the selected site.
2. **Settings → Wired Networks → LAN** — VLAN definitions, subnets, gateways, and DHCP ranges for networks Omada manages.
3. **Settings → Wireless Networks → WLAN** — each SSID, its VLAN, and its security/RADIUS profile.
4. **Settings → Network Profile → RADIUS Profile** — the RADIUS servers referenced by the SSIDs.
5. **Devices** — the AP inventory, each AP's IP, model, and uplink switch/port.
6. **Insight / Clients** — current clients and their addresses, useful for sanity-checking scope usage.

### UniFi

1. **Settings → Networks** — each Virtual Network (Ubiquiti's term for a VLAN): VLAN ID, gateway/subnet, DHCP range and lease time, DHCP reservations.
2. **Settings → WiFi** — SSIDs and their network/VLAN assignment.
3. **Devices → [switch] → Ports** — per-port link state, VLAN/network profile, and PoE draw.
4. **Devices** — AP inventory with IPs and uplinks.

Note that UniFi Network 9.0+ reorganized firewall and zone configuration; if the menus do not match, look under Settings for Networks, Zones, or Policy.

### Active Directory (for the DNS side of IPAM)

```
Get-DnsServerZone                                         # every zone, including reverse zones — reveals which subnets have PTR coverage
Get-DnsServerResourceRecord -ZoneName example.com -RRType A     # every internal A record — cross-check against the static registry
Get-DhcpServerv4Scope -ComputerName <dhcp-server>          # Windows DHCP scopes, if Windows serves any
Get-DhcpServerv4Reservation -ComputerName <dhcp-server> -ScopeId <scope>   # reservations
```

A subnet with no reverse zone is a gap worth recording.

### Cross-checking

Once the tables are populated, check them against each other. Discrepancies are findings:

- Every VLAN on a switch should have a pfSense interface, or be documented as deliberately L2-only.
- Every pfSense interface with a DHCP scope should have a matching reverse DNS zone.
- Every host in the static registry should have a DNS A record and a PTR.
- Every address in the static registry should be outside its subnet's DHCP range.
- Every port forward should correspond to a service that still exists.
- Every alias should be referenced by at least one rule.

---

## Troubleshooting

Failures specific to addressing and VLAN structure. For general network triage, start with `Network-Administration-Guide.md` Section 2, which works outward from the client in layers.

### Symptom: a port links up but the device gets nothing

- **Likely cause**: PVID, VLAN Member and VLAN Tag are not consistent on that port — most often the port is a member of the VLAN but the PVID still points elsewhere, so untagged frames from the device land in the wrong VLAN.
- **Check**: read all three values for that port, not just one.

```
show vlan id <vlan>                        # EdgeSwitch: which ports are in this VLAN, tagged vs untagged
show mac-addr-table interface 0/<port>     # is the device's MAC being learned at all
show interfaces status                     # confirm link, speed, duplex
```

- **Fix**: set the port's membership and PVID together. See `Networking Guide/Switch Port to Server VLAN/SwitchConfiguration.txt` for the working example — `PVID 1`, `VLAN Member 1,4,5`, `VLAN Tag 4,5` describes a port that drops untagged traffic into VLAN 1 while also carrying VLANs 4 and 5 tagged.
- **If 802.1x is enabled on the port**, a device failing RADIUS authentication produces exactly this symptom. Check the switch's dot1x port status and the RADIUS logs on auth2/auth4 before touching the VLAN config.

### Symptom: a VLAN works in one closet but not another

- **Likely cause**: the VLAN is missing from the tagged set on an intermediate trunk. It only has to be missing on one link in the path.
- **Check**: walk the path. On every switch between the working closet and the broken one:

```
show vlan id <vlan>                        # is the VLAN defined here at all
show running-config | grep -A5 "interface 0/<uplink-port>"   # is it in the uplink's tagged set
```

- **Fix**: add the VLAN to every trunk in the path, then re-test. Record the correction in the port table above.

### Symptom: two devices claim the same IP address

- **Likely cause**: a statically configured host inside a DHCP range, or a DHCP reservation that duplicates a static address.
- **Check**:

```
arp -a | grep <ip>                         # on pfSense: see which MAC currently answers
```

Then compare that MAC against the static registry and the DHCP reservation table. A static address that is inside the scope range and not in the registry is the usual answer.

- **Fix**: move the static host outside the DHCP range, or shrink the range. Then add the host to the static registry — the registry existing is what prevents a repeat.

### Symptom: a subnet is unreachable over the VPN but fine on the LAN

- **Likely cause**: the subnet is not in the WireGuard peer's allowed-IPs, or not in the partner IPsec Phase 2 selectors. A subnet added after the VPN was configured will not be carried by it.
- **Check**:

```
wg show                                    # WireGuard allowed-IPs per peer
swanctl --list-conns                       # IPsec traffic selectors
netstat -rn                                # is there a route at all
```

- **Fix**: add the subnet to the relevant selector or allowed-IPs list on both ends. A Phase 2 selector that matches on one side and not the other brings the tunnel up but passes no traffic for that subnet.

### Symptom: a new subnet overlaps something and traffic vanishes

- **Likely cause**: the new subnet collides with a VPN pool, a remote network over the partner tunnel, or a VPN client's home network.
- **Check**: compare against the subnet allocation table in full, including VPN rows. On pfSense, watch for traffic disappearing into a tunnel:

```
tcpdump -ni enc0                           # IPsec traffic; the kernel routes matching selectors here even if the tunnel is down
netstat -rn                                # look for a more-specific route than expected
```

- **Fix**: renumber the new subnet. This is why the allocation table must be checked before allocating, not after.

### Symptom: devices drop off a switch with no obvious link failure

- **Likely cause**: the switch has hit its PoE budget and is shedding lower-priority powered devices — APs, phones, cameras.
- **Check**: `show poe status` on EdgeSwitch, or the PoE column in the UniFi/Omada device view. Compare total draw against the switch's budget.
- **Fix**: redistribute powered devices across switches, or move to a higher-budget unit. Record actual draw in the physical topology section so the ceiling is known before it is hit again.

### Symptom: this document disagrees with the live equipment

- **The equipment is right.** This document is a transcription. Correct the document, add a row to the change log explaining what was found, and consider why the drift happened — usually a change made without documenting it.

---

## Security

- **Management plane**: switch, AP controller, and firewall management interfaces should live in a management VLAN reachable only from admin workstations or over VPN, never from general user VLANs and never from the internet. [FILL IN: confirm current management exposure]
- **Guest isolation**: the guest VLAN must not reach any internal VLAN, including printers and the management VLAN. Verify by testing, not by reading the rules.
- **Default-deny**: every VLAN interface should have an explicit, minimal ruleset. A permissive rule left from a troubleshooting session is the most common finding in a firewall review.
- **802.1x**: wired and wireless auth via RADIUS to AD. On EdgeSwitch, roll out with `dot1x system-auth-control monitor` first — it surfaces which devices would fail without actually blocking them.
- **Snort and fail2ban** run on pfSense; unblock procedures are documented in `Security & Hardening/`.
- **SNMP**: if SNMP is enabled on the switches for monitoring, community strings are credentials. Restrict by manager IP and do not use v1/v2c community strings on an untrusted segment. [FILL IN: is SNMP enabled, with what version, and restricted to which manager?]
- **Secrets**: switch and controller credentials, SNMP strings, and RADIUS shared secrets live in 1Password, never here.

## Monitoring & Alerting

- [FILL IN: are switch and pfSense syslog shipped to graylog01.example.com? Which facilities?]
- [FILL IN: is there SNMP monitoring of interface utilization, PoE draw, and switch health?]
- Worth alerting on: core uplink down, stack member down, PoE budget above threshold, new device on a management VLAN, Snort block volume spike.
- Baseline: [FILL IN: normal uplink utilization and PoE draw, so an abnormal reading is recognizable]

## Disaster Recovery

- **pfSense config**: `config.xml` via Diagnostics → Backup & Restore, plus Auto Config Backup (free on both CE and Plus, up to 100 encrypted backups). Take a manual backup before any change here.
- **Switch configs**: `copy nvram:startup-config tftp://...` on EdgeSwitch; equivalent export on the M5300. [FILL IN: are switch configs archived on a schedule, and where?]
- **Controller configs**: UniFi cloud backups (Settings → Control Plane → Backups) run weekly and before major updates. Omada backup location: [FILL IN]
- **Rebuild order** after a total loss: pfSense (WAN, VLAN interfaces, DHCP, rules, NAT) → core switch VLANs and trunks → access switches → controllers and APs → verify per-VLAN with the verification sequence above.
- **This document is part of DR.** If it is accurate, a rebuild is a day. If it is not, it is a week of rediscovery. That is the argument for filling it in.

## Decisions & History (ADR-lite)

| Date       | Decision / Change                                                | Why / Ticket |
|------------|------------------------------------------------------------------|--------------|
| 2026-09-11 | This reference created as the structure for the address plan      | No single place recorded VLAN/subnet allocation |
| [FILL IN]  | [FILL IN: past VLAN or subnet changes worth remembering]          | [FILL IN] |

Record every VLAN and subnet allocation here going forward. The change log is what prevents the same /24 being allocated twice.

## References

- `Networking Guide/Network-Administration-Guide.md` — pfSense and switch administration, EdgeSwitch CLI reference, layered troubleshooting
- `Networking Guide/dns-dhcp-administration-guide.md` — DHCP scopes and reservations, DNS zones, reverse zone coverage
- `Networking Guide/Switch Port to Server VLAN/SwitchConfiguration.txt` — PVID / VLAN Member / VLAN Tag semantics, M5300 GUI quirks
- `Security & Hardening/Change Radius for WIfi.txt` — Omada site selection and RADIUS profile procedure
- `WireGuard/` — site VPN configuration and client management
- `Networking & Switches/` — switch manuals, rack switch config, partner VLAN switch config photo, pfSense NAT/port-forward screenshots worth transcribing into the NAT tables above
- `WifiSurvey/` — wireless survey methodology and AP placement material

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

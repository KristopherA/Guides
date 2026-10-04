> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# DNS and DHCP Administration (example.com)

## Overview

This guide covers name resolution and address assignment for the example.com network: which resolver answers which query, how to add and change records safely, how DNSSEC is configured on the Active Directory-integrated zones, how DHCP scopes and reservations are managed, and how to triage a "DNS is broken" report from the client outward. DNS and DHCP are the two services that make nearly every other failure look like something else — a slow login, a printer that "disappeared", a web app that 502s, an 802.1x port that authenticates but never gets traffic. Checking them first is almost always cheaper than not.

Audience: IT Ops staff with administrative access to the Active Directory DNS servers, pfSense, and the Omada/UniFi controllers. This guide supersedes and absorbs the older `_ARCHIVE/superseded-stubs/enable dnssec.txt` (archived 2026-09-11) note, whose content is carried forward in full under "DNSSEC" below.

## Quick Facts

| Field            | Value                                                                                     |
|------------------|-------------------------------------------------------------------------------------------|
| Owner            | IT Ops (it@example.com)                                                |
| Environment      | prod                                                                                       |
| Location         | Internal authoritative DNS: Active Directory-integrated zones on the domain controllers — [FILL IN: DC hostnames and IPs serving DNS]. Edge resolver/forwarder: pfSense (Services → DNS Resolver, Unbound). External/public DNS for example.com: [FILL IN: external DNS provider and account/portal URL] |
| Access           | DNS Manager (`dnsmgmt.msc`) or PowerShell on a DC, over RDP from an admin workstation; pfSense web GUI over the internal management network or WireGuard VPN; external DNS via [FILL IN: external DNS provider portal, login location in 1Password] |
| Dependencies     | Active Directory (zones are AD-integrated and replicate via AD), domain controller availability, pfSense WAN uplink for recursion/forwarding, upstream forwarders — [FILL IN: upstream forwarder IPs configured on pfSense DNS Resolver] |
| Dependents       | Effectively everything: AD logon and Kerberos (SRV records), 802.1x/RADIUS (auth2.example.com, auth4.example.com), mail.example.com, Apache vhosts on data/wiki/sign/appdb2/appdb1/erpdb/prod01/files/forums, Graylog ingestion (graylog01.example.com), WireGuard clients, Let's Encrypt HTTP-01 and DNS-01 validation |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                                                   |

## How It Works

### Split-horizon: which resolver is authoritative for what

example.com is served by two separate sets of records that share a name but not content. This is the single most important thing to understand before touching anything.

**Internal view (authoritative: Active Directory DNS on the domain controllers).** Domain-joined clients, and anything else pointed at the internal resolvers, resolve `example.com` and its subdomains against the AD-integrated forward lookup zone. This zone holds the internal-only records: host records for infrastructure (data, wiki, sign, appdb2, appdb1, erpdb, prod01, graylog01, files, forums, auth2, auth4, mail), the `_msdcs.example.com` delegation and the `_ldap._tcp` / `_kerberos._tcp` SRV records AD depends on, and dynamically registered workstation records. Internal clients that resolve `sign.example.com` get its internal address, not its public one.

**External view (authoritative: the public/hosted zone).** The internet resolves `example.com` against the externally hosted zone at [FILL IN: external DNS provider]. This zone holds only what the public needs to see: the public A/AAAA records for internet-facing services, MX records for mail, SPF/DKIM/DMARC TXT records, and any CAA record governing which CA may issue certificates for example.com. It does not contain internal infrastructure records and should not.

**Recursion path for internal clients.** A domain-joined client asks an internal DC. If the name is inside an AD-integrated zone, the DC answers authoritatively. If not, the DC forwards to its configured forwarder — here that is [FILL IN: do the DCs forward to pfSense, or directly to an upstream resolver? confirm forwarder list in DNS Manager → server Properties → Forwarders]. pfSense, if it is in the path, runs Unbound (Services → DNS Resolver) and either recurses to the root servers or forwards upstream depending on how "Enable Forwarding Mode" is set.

The practical consequence: **the same name can resolve to two different addresses depending on where you ask from.** When a user says "the site is down," always establish which view they were in. A record added to the public zone will not be visible internally, and vice versa. Records that must work in both views have to be created in both zones, independently, and they will drift apart unless someone is deliberate about it.

### Where config and data live

| Component                   | Where it lives                                                                       |
|-----------------------------|--------------------------------------------------------------------------------------|
| Internal zone data          | AD database (`NTDS.dit`), replicated to all DCs; viewed via `dnsmgmt.msc`             |
| Internal DNS server settings | Registry on each DC plus the DNS server object in AD; `Get-DnsServerSetting`          |
| pfSense resolver config     | pfSense `config.xml`; GUI at Services → DNS Resolver; runtime at `/var/unbound/`       |
| pfSense DHCP leases (ISC)   | `/var/dhcpd/var/db/dhcpd.leases` on pfSense                                            |
| Windows DHCP (if used)      | [FILL IN: is DHCP served by pfSense, by Windows DHCP on a DC, by the Omada/UniFi controllers, or a mix? This determines which half of the DHCP sections below applies] |
| DNS query/audit logs        | DNS Server operational log on each DC (Event Viewer); optional analytical logging       |
| Aggregated logs             | graylog01.example.com — [FILL IN: which DNS/DHCP log streams are shipped to Graylog]         |

### DHCP topology

Address assignment is split across the segmented VLAN design described in `Network-Administration-Guide.md`. Which device serves DHCP for which VLAN is a per-VLAN decision and is captured in the scope inventory below. pfSense serves DHCP for the VLANs it gateways (Services → DHCP Server, one tab per interface); the Omada and UniFi controllers can serve DHCP for wireless networks they define; and a Windows DHCP server, if present, would serve scopes with AD-integrated DNS registration. Do not assume a single answer — confirm per VLAN before changing anything.

Note that pfSense currently ships two DHCP backends, **Kea** and **ISC**, selected under System → Advanced → Networking. The commands and file paths differ between them. Confirm which backend is running before following any CLI step in this guide.

---

## Zone Inventory

Populate one row per zone. "Authoritative on" means the server that holds the master copy and answers without forwarding.

### Internal zones (Active Directory-integrated)

| Zone name | Type | Authoritative on | Replication scope | Dynamic updates | DNSSEC signed | Purpose | Notes |
|---|---|---|---|---|---|---|---|
| example.com | Forward, AD-integrated | [FILL IN: DC hostnames] | [FILL IN: forest-wide / domain-wide / DC-specific] | [FILL IN: Secure only / Nonsecure and secure / None] | [FILL IN: yes/no] | Primary internal namespace | Should be "Secure only" — see Security |
| _msdcs.example.com | Forward, AD-integrated | [FILL IN: DC hostnames] | [FILL IN: forest-wide expected] | [FILL IN] | [FILL IN] | AD locator SRV records | Breaking this breaks domain logon |
| [FILL IN: reverse zone, e.g. x.y.10.in-addr.arpa] | Reverse | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | PTR records for [FILL IN: which subnet] | One reverse zone per subnet in use |
| [FILL IN: any additional internal zones — lab, test, legacy, conditional forwarders] | | | | | | | |

### External / public zones

| Zone name | Hosted at | Registrar | Record types present | DNSSEC at registrar | Notes |
|---|---|---|---|---|---|
| example.com | [FILL IN: external DNS provider] | [FILL IN: registrar] | A, MX, TXT (SPF/DKIM/DMARC), CAA, [FILL IN: others] | [FILL IN: is the DS record published at the registrar?] | Public view only |
| [FILL IN: any additional public domains the organization owns, e.g. example.org — the Apache vhost doc references example.org addresses] | | | | | |

### Conditional forwarders and stub zones

| Forwarded zone | Forward to | Configured on | Reason |
|---|---|---|---|
| [FILL IN: e.g. a partner or partner-side namespace reachable over the IPsec tunnel] | [FILL IN: target DNS server IPs] | [FILL IN: DCs / pfSense] | [FILL IN: why this exists] |

---

## Operations (Day-2)

### Adding, changing, or removing a DNS record

The procedure differs by view. Decide first: does this record need to resolve internally, externally, or both? If both, you are doing the procedure twice.

**Before any change:** note the current value so you can revert. Query it and keep the output.

```
nslookup -type=ANY sign.example.com <internal-dns-ip>    # current internal answer
dig @<external-resolver> sign.example.com ANY +noall +answer   # current external answer
```

#### Internal record (AD DNS)

GUI path: `dnsmgmt.msc` → Forward Lookup Zones → example.com → right-click → New Host (A or AAAA) / New Alias (CNAME) / Other New Records.

PowerShell, run on a DC or from an admin workstation with RSAT:

```
Add-DnsServerResourceRecordA -ZoneName example.com -Name newhost -IPv4Address <ip> -CreatePtr   # add A + matching PTR
Set-DnsServerResourceRecord -ZoneName example.com -OldInputObject $old -NewInputObject $new      # change an existing record
Remove-DnsServerResourceRecord -ZoneName example.com -Name oldhost -RRType A                     # remove
```

Use `-CreatePtr` so the reverse zone stays in step. Missing PTRs are a recurring cause of slow SSH logins, confusing Graylog entries, and mail delivery problems.

**Verification step (do not skip).** A change is not done until it is observed from a client, not from the server that made it.

```
Get-DnsServerResourceRecord -ZoneName example.com -Name newhost      # confirm it exists on the DC that made the change
nslookup newhost.example.com <second-dc-ip>                          # confirm AD replication carried it to the other DC
ipconfig /flushdns                                               # clear the client resolver cache first
nslookup newhost.example.com                                          # confirm from an ordinary domain-joined client
```

If the second DC does not have it, that is an AD replication problem, not a DNS problem — see `Active Directory/AD-Admin-Security-Guide.md` and run `repadmin /replsummary`.

#### External record (public zone)

Changes are made in the [FILL IN: external DNS provider] portal. Before changing anything, record the current TTL — the old value stays cached in the world's resolvers for that long.

```
dig sign.example.com A +noall +answer +ttlid    # current value and remaining TTL
```

If a change is planned (a migration, a new public IP), **lower the TTL first**, wait for the old TTL to expire, then make the change, then raise the TTL back. Lowering the TTL at the same time as making the change accomplishes nothing.

Verification, against a resolver that is not yours:

```
dig @1.1.1.1 sign.example.com A +short      # public resolver, not the internal view
dig @8.8.8.8 sign.example.com A +short      # second opinion — propagation is not uniform
```

#### Records that require extra care

- **MX, SPF, DKIM, DMARC** — a typo silently breaks inbound or outbound mail for mail.example.com, often without any error visible to you. Validate the SPF record's syntax and lookup count before saving, and test delivery both ways afterward.
- **CAA** — a CAA record restricts which CA may issue certificates for example.com. If it does not list the Let's Encrypt identifier, ACME issuance fails with a CAA error, and the failure appears at renewal time, not at record-change time. See `Linux & Servers/reverse-proxy-and-tls-automation-guide.md`.
- **SRV records under `_msdcs.example.com`** — do not hand-edit these. They are registered by Netlogon. If they are missing, restart Netlogon or run `nltest /dsregdns` on the affected DC rather than creating them manually.
- **Anything a certificate depends on** — moving a host's A record without thinking about its cert is how you get a working DNS change and a broken TLS handshake.

### DNSSEC

DNSSEC signs zone data so a resolver can verify the answer was not tampered with in transit. Here this applies in two separate places that are often confused: **signing the internal AD zone** (below) and **publishing a DS record at the registrar for the public zone** ([FILL IN: is the public example.com zone DNSSEC-signed at the provider, and is the DS record published at the registrar?]).

The following procedure is carried forward from `_ARCHIVE/superseded-stubs/enable dnssec.txt` (archived 2026-09-11) and applies to Windows Server 2019 DNS. Confirm the OS version on the current DCs before following it — [FILL IN: current DC OS version; the AD guide references Windows Server 2022].

#### Signing an internal zone

1. Open DNS Manager: Start → type `dnsmgmt.msc` → Enter.
2. Navigate to the DNS server, expand **Forward Lookup Zones**, and find the zone to sign.
3. Right-click the zone → **DNSSEC** → **Sign the Zone**.
4. In the Zone Signing Wizard, click **Next**. Choose **Customize zone signing parameters** to adjust settings, or accept the defaults. Click **Next**.
5. **Key master**: select the server that will store the keys — normally the local server for a first-time setup. Click **Next**.
6. **Key Signing Key (KSK)**: click **Add**, then **OK** to accept defaults or customize. Click **Next**.
7. **Zone Signing Key (ZSK)**: click **Add**, then **OK**. Click **Next**.
8. Choose **NSEC3** rather than NSEC. NSEC3 hashes the record names in the denial-of-existence proofs, which prevents an attacker from walking the zone to enumerate every host in it. On an internal zone full of infrastructure hostnames this matters.
9. If the DNS server is AD-integrated, **enable distribution of trust anchors for this zone**. This is what lets other DCs and domain-joined validating clients trust the signatures. Click **Next**.
10. Proceed through the remaining steps with defaults unless there is a specific requirement, and click **Finish**.

**Verify signing**: a small lock icon appears next to the zone in DNS Manager. Also confirm from a client that RRSIG records are being returned:

```
dig @<internal-dns-ip> example.com SOA +dnssec +multiline    # expect RRSIG records alongside the answer
Resolve-DnsName example.com -Server <internal-dns-ip> -DnssecOk   # PowerShell equivalent
```

#### Enabling DNSSEC validation on the server

The GUI toggle for validating DNSSEC on responses from remote servers was removed in Windows Server 2019. Enable it from the command line instead.

PowerShell:

```
Get-DnsServerSetting | Set-DnsServerSetting -EnableDnsSec $true    # enable validation of remote responses
```

Or `dnscmd`:

```
dnscmd /config /EnableDnsSec 1     # same setting via the legacy tool
```

Then restart the DNS service:

```
Restart-Service DNS                # brief resolution outage on this DC — do one DC at a time
```

Restart DCs one at a time, not in parallel. If every DC's DNS service is down simultaneously, domain logon stops.

#### Configuring clients to require DNSSEC validation

Clients only benefit if they are told to ask for and require validation. This is done with the Name Resolution Policy Table (NRPT) via Group Policy.

1. Create or edit a GPO and link it to the OU containing the client computers.
2. Navigate to **Computer Configuration → Policies → Windows Settings → Name Resolution Policy**.
3. Add a rule:
   - Enter the DNS suffix for the domain (`example.com`).
   - Check **Enable DNSSEC in this rule**.
   - Check **Require DNS clients to check that name and address data has been validated by the DNS server**.
4. Click **Create** to save the rule.

After applying, run `gpupdate /force` on a test client and confirm resolution still works before rolling the GPO out broadly. A DNSSEC misconfiguration plus a "require validation" client policy produces total resolution failure for that suffix, which looks exactly like the network being down.

Verify the client policy is in effect:

```
Get-DnsClientNrptPolicy                 # shows the effective NRPT rules on the client
gpresult /r                             # confirm the GPO actually applied
```

#### DNSSEC operational cautions

- **Key rollover**: Windows handles ZSK/KSK rollover automatically once configured, but if the KSK rolls and the DS record at the parent is not updated, validation fails everywhere. For the internal zone the "parent" is the trust anchor distributed through AD; for the public zone it is the DS record at the registrar. [FILL IN: confirm rollover schedule and who is responsible for updating the DS record]
- **Signed zone + broken time sync = validation failure.** Signatures have validity windows. A DC with badly skewed clock will reject valid signatures. Check `w32tm /query /status` when DNSSEC failures appear out of nowhere.
- **Unsigning** is done from the same menu (right-click zone → DNSSEC → Unsign the Zone) and is the correct emergency action if signing is causing an outage and the cause is not immediately findable.

### DHCP scope inventory

One row per scope. Cross-reference against the VLAN table in `ipam-vlan-topology-reference.md` — every VLAN that has clients should have exactly one scope, served by exactly one device.

| VLAN ID | VLAN name | Subnet | Scope range (start–end) | Gateway (option 3) | DNS servers (option 6) | Domain (option 15) | Lease time | Served by | Notes |
|---|---|---|---|---|---|---|---|---|---|
| [FILL IN] | [FILL IN: e.g. Staff] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | example.com | [FILL IN] | [FILL IN: pfSense / Windows DHCP / Omada / UniFi] | |
| [FILL IN] | [FILL IN: e.g. Server] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | example.com | [FILL IN] | [FILL IN] | Servers are normally statically addressed — confirm whether this scope exists at all |
| [FILL IN] | [FILL IN: e.g. Guest Wireless] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN: likely external resolvers, not internal DCs] | [FILL IN] | [FILL IN: short, guests churn] | [FILL IN: Omada / UniFi] | Should not reach internal DNS |
| [FILL IN] | [FILL IN: e.g. Voice] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Check for option 66/150 (TFTP/provisioning server) if phones use it |
| [FILL IN] | [FILL IN: e.g. Management] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Switch/AP management |
| [FILL IN: remaining VLANs] | | | | | | | | | |

**Non-default DHCP options in use.** Record anything set beyond gateway/DNS/domain, because these are invisible until they break something:

| Option | Number | Value | Applies to which scope | Why |
|---|---|---|---|---|
| [FILL IN: e.g. TFTP server name] | 66 | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: e.g. NTP servers] | 42 | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN] | | | | |

### Static reservation inventory

Reservations are for devices that need a predictable address but should still be centrally managed — printers, APs, cameras, appliances. Servers generally get true static addresses configured on the host instead; those belong in the static IP registry in `ipam-vlan-topology-reference.md`, not here.

| Hostname | MAC address | Reserved IP | VLAN | Device type | Served by | Owner / contact | Notes |
|---|---|---|---|---|---|---|---|
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |

### Adding a static DHCP reservation

**First, get the MAC address and confirm it is the right one.** A reservation against the wrong MAC produces a device that intermittently gets the wrong address — worse than no reservation.

```
arp -a | grep <current-ip>                          # from a host on the same VLAN
ip neigh show | grep <current-ip>                   # Linux equivalent
```

On the switch, find which port and MAC the device is actually on. On an EdgeSwitch:

```
show mac-addr-table                                 # full MAC table
show mac-addr-table interface 0/<port>              # MACs learned on one port
```

**Second, check the address you intend to reserve is outside the dynamic pool** or that the platform handles in-pool reservations correctly. The safest convention is a dedicated reservation band at one end of each subnet, outside the scope range. [FILL IN: what is the organization's convention — is there a reserved band per subnet?]

**Third, create it on whichever platform serves that VLAN:**

- **pfSense**: Services → DHCP Server → [interface tab] → Static Mappings → Add. Enter MAC, IP, hostname, description. Save, then Apply Changes. (On the Kea backend the equivalent lives under the Kea Settings tab — the GUI layout differs; confirm your backend under System → Advanced → Networking.)
- **Omada (omada.example.com)**: Settings → Wired Networks / Wireless Networks → the relevant LAN → DHCP Reservation. Select the main office site in the location menu top-left first, the same way as for the RADIUS profile change in `Security & Hardening/Change Radius for WIfi.txt`.
- **UniFi**: Settings → Networks → [network] → DHCP reservations, or set "Fixed IP" on the client object under Client Devices.
- **Windows DHCP**: DHCP console → scope → Reservations → New Reservation.

**Fourth, force the device to take it.** A reservation does not apply until the device's current lease is released or expires.

```
ipconfig /release && ipconfig /renew                # Windows client
sudo dhclient -r && sudo dhclient                   # Linux client
sudo ipconfig set en0 DHCP                          # macOS, renew on en0
```

For an unmanaged device (printer, camera, AP), power-cycle it, or delete its active lease from the DHCP server so it has to ask again.

**Fifth, verify:**

```
ping <reserved-ip>                                  # it answers
nslookup <reserved-ip>                              # PTR resolves if dynamic DNS registration is working
```

Then confirm the lease on the server side shows the reservation as active and bound to the expected MAC — see the lease commands below. Finally, add the row to the reservation inventory table above. A reservation that exists only in the DHCP server and not in this doc will be a mystery to the next person.

### Viewing and clearing DHCP leases

**pfSense (ISC backend):**

```
cat /var/dhcpd/var/db/dhcpd.leases                  # raw lease database
service isc-dhcpd restart                           # restart; existing leases survive, minimal disruption
```

GUI: Status → DHCP Leases. This page also offers a per-lease delete, which is the cleanest way to force one device to re-request.

**pfSense (Kea backend):** manage via Status → Services in the GUI. The exact service name for CLI restart varies by version — confirm before scripting it. [FILL IN: which DHCP backend is pfSense running, and the exact Kea service name if applicable]

**Omada / UniFi:** the controller's client list shows the current IP and lease per client; leases are released from the client detail panel.

**Windows DHCP:**

```
Get-DhcpServerv4Lease -ComputerName <dhcp-server> -ScopeId <scope>       # list leases in a scope
Remove-DhcpServerv4Lease -ComputerName <dhcp-server> -IPAddress <ip>     # free a specific lease
Get-DhcpServerv4ScopeStatistics -ComputerName <dhcp-server>              # pool utilization — check this before blaming anything else
```

### Routine checks

Run these periodically, not just during an incident.

```
Get-DhcpServerv4ScopeStatistics -ComputerName <dhcp-server>   # any scope near 100% will bite you soon
dcdiag /test:dns /v                                            # DNS health across the DCs
repadmin /replsummary                                          # AD replication — internal zone data depends on it
dig @<internal-dns-ip> example.com SOA +dnssec                      # confirm zone still signed and serving
```

---

## Troubleshooting

### Runbook: "DNS is broken"

Work from the client outward and stop at the first layer that fails. Most reports that arrive as "DNS is broken" are actually a client cache, a wrong resolver, or a routing problem — not the DNS servers.

**Step 0 — Establish what "broken" means.** Get the exact name that failed, the exact error, and where the user was (office wired, office wireless, guest wireless, WireGuard VPN, home). A name that fails only on VPN and a name that fails everywhere are different problems. Remember the split horizon: confirm which view they were in.

**Step 1 — Is it DNS at all?** Test IP connectivity before name resolution. If IPs do not work either, this is a routing or physical problem and DNS is a red herring — go to `Network-Administration-Guide.md` Section 2.

```
ping 1.1.1.1                                       # raw IP connectivity
ping <internal-dns-ip>                             # can the client even reach its resolver
```

**Step 2 — What resolver is the client actually using?** This is the single most common finding. A client that got the wrong DHCP options, has a stale VPN resolver, or has a hardcoded public resolver will fail on internal names while "the internet works fine."

```
ipconfig /all                                       # Windows: check DNS Servers and Connection-specific suffix
resolvectl status                                   # Linux (systemd-resolved): per-link resolvers
scutil --dns | head -30                             # macOS: resolver order
Get-DnsClientNrptPolicy                             # Windows: is an NRPT rule redirecting or requiring DNSSEC
```

Expect the internal DC addresses on a domain-joined client, plus the `example.com` search suffix. A guest-network client will correctly have external resolvers and will correctly fail on internal names.

**Step 3 — Clear the client cache and retest.** Negative answers cache too. A record you created five minutes ago may be cached as NXDOMAIN on the client.

```
ipconfig /flushdns                                  # Windows
sudo resolvectl flush-caches                        # Linux (systemd-resolved)
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder   # macOS
nslookup <failing-name>                             # retest
```

If it works now, the fault was cache. Note the TTL of the record — if it is long, other clients are still holding the bad answer.

**Step 4 — Query each resolver directly, bypassing the client's configuration.** This separates "the client is asking the wrong thing" from "the server is answering wrong."

```
nslookup <failing-name> <dc1-ip>                    # ask DC1 directly
nslookup <failing-name> <dc2-ip>                    # ask DC2 directly — a difference means replication lag
dig @<pfsense-ip> <failing-name>                    # ask the pfSense resolver
dig @1.1.1.1 <failing-name>                         # ask the outside world (external view)
```

Interpreting the split:

| Result | Meaning |
|---|---|
| DC1 answers, DC2 does not | AD replication problem — `repadmin /replsummary`, not a DNS problem |
| Both DCs fail, pfSense answers | Record missing from internal zone, or zone not loading on the DCs |
| Internal answers, external does not | Public record missing or not propagated |
| External answers, internal does not | Expected for public-only names; a problem only if the name should exist internally too |
| Everything returns SERVFAIL | Suspect DNSSEC validation failure or a dead forwarder — see below |

**Step 5 — Check the authoritative server itself.** On the DC:

```
Get-Service DNS                                     # is it running
Get-DnsServerZone                                   # are the zones loaded
Get-DnsServerResourceRecord -ZoneName example.com -Name <host>   # does the record exist at all
dcdiag /test:dns /v                                 # broad DNS health check
```

Check the DNS Server operational log in Event Viewer for load failures, and check that the zone is not paused.

**Step 6 — Check forwarding and recursion.** If internal names resolve but external ones do not, the fault is in the forwarding path, not the zone.

```
Get-DnsServerForwarder                              # what the DC forwards to
dig @<forwarder-ip> google.com                      # is that forwarder actually answering
```

On pfSense: Services → DNS Resolver. Check whether forwarding mode is on and what upstreams are configured, then:

```
service unbound restart                             # restart the resolver; brief outage for clients using it
drill google.com                                    # pfSense CLI lookup
```

Also confirm pfSense's own WAN is up — a resolver that cannot reach the internet fails every external lookup while internal lookups still work, which reads as "DNS is broken."

**Step 7 — Consider DNSSEC.** SERVFAIL on names that are known-good, especially appearing suddenly and affecting a whole zone, is the signature of a DNSSEC validation failure.

```
dig <failing-name> +cd                              # +cd disables validation; if this works and the normal query fails, DNSSEC is the cause
w32tm /query /status                                # clock skew on the DC breaks signature validity windows
```

If the internal zone's signing is the cause and the fix is not immediate, unsigning the zone (DNS Manager → right-click zone → DNSSEC → Unsign the Zone) restores service. Do this deliberately and re-sign once the root cause is understood.

**Step 8 — Escalation.** If the fault is at the public zone, the change has to be made at [FILL IN: external DNS provider] and you are then waiting on TTL expiry, which cannot be rushed. Communicate the TTL as the expected recovery time.

### Symptom: client has no IP, or a 169.254.x.x APIPA address

- **Likely cause**: DHCP is not reachable from that VLAN — no scope, exhausted pool, wrong switch port VLAN, or a DHCP relay that is not relaying.
- **Check**: `ipconfig /all` on the client — APIPA confirms no DHCP response was received. Then check pool utilization on the server (`Get-DhcpServerv4ScopeStatistics`, or Status → DHCP Leases on pfSense).
- **Check the switch port** — a port in the wrong VLAN will link up and get nothing. On an EdgeSwitch or the NetGear M5300, confirm PVID and VLAN membership. See `Networking Guide/Switch Port to Server VLAN/SwitchConfiguration.txt` for the PVID / VLAN Member / VLAN Tag distinction: PVID is the VLAN untagged frames are placed into, VLAN Member is which VLANs the port belongs to, VLAN Tag is which VLANs the port accepts tagged.
- **Check 802.1x** — on a port with 802.1x enabled, a device that fails RADIUS authentication may be left unauthorized or placed in a guest/restricted VLAN with no usable scope. The symptom looks identical to a DHCP failure. Check the RADIUS logs on auth2.example.com / auth4.example.com and the switch's dot1x port status before assuming DHCP.
- **Fix**: correct the port VLAN, expand or free the pool, or resolve the 802.1x failure. Then force a renew on the client.

### Symptom: client gets an IP but from the wrong subnet

- **Likely cause**: the port or SSID is in the wrong VLAN, or two DHCP servers are answering on the same segment.
- **Check**: compare the address received against the scope inventory above. Then look for a rogue or duplicate DHCP server — on pfSense, a packet capture on the affected interface filtered to UDP 67/68 will show which server answered:

```
tcpdump -ni <interface> port 67 or port 68          # watch DHCP offers; more than one server offering is the fault
```

- **Fix**: correct the VLAN assignment, or disable the second DHCP service. A consumer router plugged in backwards is a classic source of this.

### Symptom: a device intermittently changes IP despite having a reservation

- **Likely cause**: the reservation is against the wrong MAC (common with devices that have both wired and wireless interfaces, or with MAC randomization on mobile clients), or the device is roaming between VLANs, or two DHCP servers each have a different idea about it.
- **Check**: confirm the MAC in the reservation matches the MAC actually seen in the lease table and on the switch MAC address table.
- **Fix**: correct the MAC. For devices with MAC randomization, disable randomization for the corporate SSIDs on that device, or use a different identification method.

### Symptom: DHCP pool exhausted

- **Likely cause**: lease time too long relative to device churn (very common on guest wireless), or a large one-off event, or a device looping through addresses.
- **Check**: `Get-DhcpServerv4ScopeStatistics`, or the pfSense DHCP Leases page. Look for many leases to similar-looking MACs.
- **Fix**: shorten the lease time on high-churn scopes, expand the range if the subnet has room, or free expired leases. Do not simply widen the range into space allocated to another purpose — check `ipam-vlan-topology-reference.md` first.

### Symptom: a host resolves to the wrong address from inside the office

- **Likely cause**: split-horizon drift — the internal zone still holds an old address, or the record was only ever updated externally.
- **Check**: query both views side by side (Step 4 above). A mismatch that is not intentional is the answer.
- **Fix**: update the internal record. Then add a note to the zone inventory that this name exists in both views, so the next change updates both.

### Symptom: stale DNS records for decommissioned hosts

- **Likely cause**: dynamic registration created the record, the host went away, and scavenging is off or misconfigured.
- **Check**: `Get-DnsServerZoneAging -Name example.com` and `Get-DnsServerScavenging` on the DC. [FILL IN: is DNS scavenging enabled on the example.com zone, and with what refresh/no-refresh intervals?]
- **Fix**: enable scavenging with intervals appropriate to the DHCP lease time (no-refresh + refresh interval should exceed the longest lease), or remove the records manually. Enabling scavenging with intervals shorter than the lease time deletes records for live hosts — set this carefully.

---

## Security

- **Exposure**: internal DNS servers must not be reachable from the internet and must not answer recursive queries for arbitrary external clients — an open resolver is an amplification-attack participant. Confirm pfSense WAN rules do not permit inbound UDP/TCP 53 to any internal resolver. [FILL IN: confirm no inbound 53 on the pfSense WAN ruleset]
- **Dynamic updates**: AD-integrated zones should be set to **Secure only** so only authenticated machine accounts can register or overwrite their own records. "Nonsecure and secure" lets any host on the network overwrite any record, including a server's.
- **Zone transfers**: restrict or disable. An unrestricted AXFR hands an attacker a complete inventory of internal hostnames. DNS Manager → zone Properties → Zone Transfers → allow only to named servers, or disable.
- **DnsAdmins**: historically able to load an arbitrary DLL into the DNS service, which runs as SYSTEM on a DC — effectively domain compromise. Treat membership as privileged and audit it. See `Active Directory/AD-Admin-Security-Guide.md` Section 17.
- **DNSSEC**: see the DNSSEC section above. NSEC3 rather than NSEC on internal zones to prevent zone walking.
- **Patching**: DNS server vulnerabilities on DCs are high priority. SIGRed (CVE-2020-1350) was a wormable RCE in Windows DNS Server.
- **DHCP**: on segments where it matters, DHCP snooping on the switches prevents a rogue server from answering. [FILL IN: is DHCP snooping configured on the NetGear M5300 / EdgeSwitch stacks?]
- **Secrets**: RADIUS shared secrets, DNS provider API credentials, and registrar logins live in 1Password, never in this document.

## Monitoring & Alerting

- [FILL IN: are DNS Server event logs and pfSense DHCP logs shipped to graylog01.example.com? Which streams?]
- [FILL IN: is there an uptime/synthetic check that resolves a known internal name and a known external name on a schedule?]
- Worth alerting on, if not already: DHCP scope utilization above 85%, DNS service stopped on any DC, SERVFAIL rate spike, zone load failure events, and DNSSEC signature expiry approaching.
- Baseline for "normal": [FILL IN: typical query volume and scope utilization, so an abnormal reading is recognizable]

## Disaster Recovery

- **Internal DNS**: AD-integrated zones are recovered with Active Directory. A single DC loss is survivable if a second DC holds the zone and clients have both resolvers configured — confirm every scope hands out at least two DNS servers. Full AD recovery procedure is in `Active Directory/AD-Admin-Security-Guide.md`.
- **pfSense resolver and DHCP config**: contained in the pfSense `config.xml`. Covered by Auto Config Backup and by the manual backup at Diagnostics → Backup & Restore.
- **Public zone**: [FILL IN: is there an exported copy of the public zone file, and where is it kept? A zone export is the only thing that makes provider loss recoverable in reasonable time]
- **RTO/RPO**: [FILL IN: agreed targets]
- **Escalation**: [FILL IN: who to contact if the owner is unavailable]

## Decisions & History (ADR-lite)

| Date       | Decision / Change                                                   | Why / Ticket |
|------------|---------------------------------------------------------------------|--------------|
| 2026-09-11 | This guide created; absorbs and supersedes `_ARCHIVE/superseded-stubs/enable dnssec.txt` (archived 2026-09-11) | Consolidating DNS/DHCP knowledge into one doc per the documentation project |
| [FILL IN]  | [FILL IN: when was DNSSEC signing originally enabled, and on which zones] | [FILL IN] |
| [FILL IN]  | [FILL IN: prior DNS/DHCP changes worth remembering]                  | [FILL IN] |

## References

- `Networking Guide/Network-Administration-Guide.md` — pfSense and switch administration, layered troubleshooting model
- `Networking Guide/ipam-vlan-topology-reference.md` — VLAN, subnet, and static IP registry that the scope inventory above depends on
- `Active Directory/AD-Admin-Security-Guide.md` — AD replication, DNS security, DC recovery
- `Security & Hardening/Change Radius for WIfi.txt` — Omada RADIUS profile procedure, referenced for the omada.example.com navigation pattern
- `Networking Guide/Switch Port to Server VLAN/SwitchConfiguration.txt` — PVID / VLAN Member / VLAN Tag semantics
- `Linux & Servers/reverse-proxy-and-tls-automation-guide.md` — CAA records and DNS-01 ACME validation
- `_ARCHIVE/superseded-stubs/enable dnssec.txt` (archived 2026-09-11) — superseded by this document; retained for history only

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

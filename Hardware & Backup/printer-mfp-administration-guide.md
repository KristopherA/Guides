> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Printer and MFP Administration

## Overview

This guide covers the printers and multifunction devices (MFPs) on the organization
network: what they are, where they sit, how print queues are built on macOS and
Windows 11, how scan-to-folder and scan-to-email are configured, how consumables
and firmware are maintained, and how the devices are securely disposed of at end
of life.

Until now the only printing documentation in the library was a six-line note on
resetting an HP maintenance-kit counter. That note is correct and is carried
forward verbatim below. Everything around it is new, and most of the
device-specific values are `[FILL IN]` because they must be read off the
equipment rather than guessed.

Two things make printing more than a convenience problem at the organization. First, MFPs
scan member and personnel documents, which means an MFP hard drive and a
scan-to-folder share are both places member data lands. Second, printers are
full network hosts with a web server, a filesystem, stored credentials, and
firmware that almost nobody patches — they are a genuine attack surface that
happens to be beige.

Audience is IT Ops and anyone covering the help desk. Tier 3 depth.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Owner            | IT lead (it@example.com), IT Ops `[CONFIRM: proposed default]` |
| Environment      | prod                                                                   |
| Location         | Devices distributed across the office — see the inventory table below. `[FILL IN: which floors/areas have printers]` |
| Access           | Each device's embedded web server over HTTPS from the management or admin network; physical control panel at the device. `[FILL IN: is there a central print server, or are queues direct-IP on each workstation?]` |
| Dependencies     | Network switching and the printer VLAN; DNS; DHCP reservations or static addressing; `files.example.com` (Synology) for scan-to-folder; the SMTP relay for scan-to-email; Active Directory if authentication or LDAP address lookup is used; the DMS for the fax path |
| Dependents       | All staff printing, scanning and copying; the scan-to-folder workflow into the file server; e-fax send/receive; any DMS workflow that starts with a scanned document |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                               |

| Item | Value |
|------|-------|
| Central print server in use? | `[FILL IN: yes/no — if yes, hostname and OS]` |
| Printer VLAN ID and subnet | `[FILL IN: the IPAM reference has a Printers/IoT VLAN row, unpopulated]` |
| Managed print / service contract vendor | `[FILL IN: is there a managed print services contract, and with whom?]` |
| Scan-to-folder target | `\\files.example.com\[FILL IN: share name]` |
| SMTP relay used for scan-to-email | `[FILL IN: mail.example.com, or a dedicated relay?]` |
| Fax path | DMS fax — see `Dynamics GP & DMS/DMS Manual/Sending a fax through DMS.pdf`; the internal e-fax tip sheet |

## Inventory

Populate one row per device. Read every value off the device or its embedded
web server — **do not guess model numbers, addresses or serials.** A printer
inventory that is half-invented is worse than none, because the wrong row will
be trusted during an outage.

| Device | Model | Location | IP | Hostname | Queue name | Driver | Serial | Service contract | Notes |
|--------|-------|----------|----|----------|------------|--------|--------|------------------|-------|
| `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | |
| `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | |
| `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | |
| `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | |
| `[FILL IN: remaining devices]` | | | | | | | | | |

Column notes, so the table is filled consistently:

- **Queue name** — the name as it appears to users in the print dialog. Keep it
  identical on macOS and Windows for the same device; divergent names make
  "print to the one by the kitchen" unanswerable.
- **Driver** — the exact driver or PPD, including IPP Everywhere / AirPrint
  where that is what is used. "HP driver" is not an answer that helps at 4pm.
- **Serial** — needed for every warranty and service call, and it is on a label
  that will be behind the device when you need it.
- **Service contract** — vendor, contract number, expiry, and what it covers
  (parts only, parts and labour, consumables included). `[FILL IN: does a
  managed print contract exist? If toner arrives without anyone ordering it,
  it does.]`

The one device model established in the library is the **HP LaserJet
M604/M605/M606 series**, referenced by the maintenance-kit reset note carried
forward below. That confirms at least one HP LaserJet is or was in service; it
does not tell us how many or where. `[FILL IN: confirm which M60x units are
still in service and where.]`

Discovery commands to populate the table rather than guessing:

```
nmap -p 9100,631,515,161 --open [FILL IN: printer subnet CIDR]   # find anything listening on print ports
dns-sd -B _ipp._tcp .                                             # macOS: browse Bonjour/IPP advertisements
dns-sd -B _pdl-datastream._tcp .                                  # macOS: browse raw JetDirect (9100) advertisements
snmpwalk -v2c -c public <printer-ip> 1.3.6.1.2.1.25.3.2.1.3       # device description via standard printer MIB
snmpwalk -v2c -c public <printer-ip> 1.3.6.1.2.1.43.11.1.1.9      # supply levels (toner, kits) per the Printer MIB
lpstat -v                                                          # macOS: existing queues and their device URIs
```

```
Get-Printer | Format-Table Name,DriverName,PortName,Shared       # Windows: queues on this machine
Get-PrinterPort | Format-Table Name,PrinterHostAddress           # Windows: the actual IP behind each queue
```

`Get-PrinterPort` is the one worth running early — it turns a list of friendly
queue names into a list of IP addresses, which is half the inventory table.

## Network Placement

### Why printers belong on their own VLAN

Printers are embedded Linux hosts with a web server, often an FTP or SMB
client, stored credentials, an open raw-print port, and firmware that gets
patched rarely if ever. They are also frequently the oldest software on the
network. Left on the staff VLAN they are a lateral-movement stepping stone that
nobody monitors.

The specific risks, which are not theoretical:

- **Port 9100 is unauthenticated by design.** Anything that can reach it can
  print, and on many devices can send PJL or PostScript commands that read and
  write the device filesystem.
- **Stored credentials.** The scan-to-folder service account password lives on
  the device, often retrievable through the web interface or a configuration
  export by anyone who can reach the admin page.
- **Default admin passwords.** Printers ship with blank or well-known admin
  credentials and are the device class most likely to still have them.
- **Print job content.** Jobs traverse the network, often unencrypted, and sit
  on the device's disk. For Example Org that includes member files and HR documents.
- **SNMP community strings** are credentials, and `public` is still the default
  on a great many devices.

`Networking Guide/ipam-vlan-topology-reference.md` already carries a
Printers/IoT VLAN row in its VLAN table, unpopulated, and a policy-matrix row
for Printers with "usually deny" against Internet. That is the right shape.
Populate it there — this document should reference the address plan, not
duplicate it.

### Intended policy

Proposed rules for the printer VLAN, to be checked against the live pfSense
ruleset rather than assumed `[CONFIRM: proposed defaults]`:

| Direction | Rule | Reason |
|-----------|------|--------|
| Staff → Printers | Allow TCP 9100, 631 (IPP/IPPS), 515 (LPD) | The actual print path. Do not allow the whole VLAN |
| Staff → Printers | Allow TCP 443 only from admin workstations | Device admin pages are not for everyone |
| Printers → files.example.com | Allow TCP 445 only, to that host only | Scan-to-folder. Nothing else on the file server |
| Printers → SMTP relay | Allow TCP 25 or 587, to that host only | Scan-to-email |
| Printers → DNS/NTP | Allow to internal servers only | Name resolution and correct timestamps on scans |
| Printers → Internet | **Deny** | A printer has no business reaching the internet. Firmware is fetched by an admin, not by the device |
| Printers → any other VLAN | **Deny** | This is the containment that makes the VLAN worth having |
| Guest → Printers | **Deny** | Guests do not print |

The "Printers → Internet: deny" row is the one that gets argued about. Cloud
print features and automatic supply reordering need it. If a managed print
contract requires the device to phone home for meter reads, permit **that
vendor endpoint specifically**, by destination, and record it in the notable
rules table in the IPAM reference — not a blanket allow.

### Addressing

Printers must have stable addresses. Either is fine; be consistent:

- **DHCP reservation** — preferred. Central record, changeable without touching
  the device, and it appears in the DHCP lease table.
  See `Networking Guide/dns-dhcp-administration-guide.md`.
- **Static on the device** — then it must go in the static registry in
  `Networking Guide/ipam-vlan-topology-reference.md`, and outside the DHCP
  range. A static printer inside a DHCP scope is a duplicate-address ticket
  waiting for its moment.

Give every printer a **DNS name** and build queues against the name, not the
IP. Re-addressing a printer then touches DNS and nothing else, instead of every
workstation queue. `[FILL IN: is there a naming convention for printer DNS
records?]`

## Print Queue Setup

### macOS

Preferred method, in order:

**1. IPP Everywhere / AirPrint (driverless)** — the right default on macOS 11+.
No driver to install, no driver to update, survives OS upgrades.

```
lpadmin -p "[FILL IN: queue name]" -E -v ipp://[FILL IN: printer-hostname]/ipp/print -m everywhere
lpadmin -p "[FILL IN: queue name]" -E -v ipps://[FILL IN: printer-hostname]/ipp/print -m everywhere   # encrypted variant, prefer this
lpadmin -p "[FILL IN: queue name]" -o printer-is-shared=false                  # do not re-advertise the queue
lpstat -p "[FILL IN: queue name]"                                               # confirm the queue exists and is enabled
```

Use `ipps://` where the device supports it. The `everywhere` model asks the
device what it can do rather than trusting a driver, so duplex, stapling and
trays appear correctly without option tinkering.

**2. Vendor driver / PPD** — only when driverless does not expose a feature the
device has (finisher options, secure print release, accounting codes) or the
device is too old to speak IPP Everywhere.

```
lpadmin -p "[FILL IN: queue name]" -E -v socket://[FILL IN: printer-hostname]:9100 -P "[FILL IN: path to PPD]"
lpadmin -p "[FILL IN: queue name]" -o [FILL IN: option]=[FILL IN: value]        # e.g. installed duplexer, extra trays
lpoptions -p "[FILL IN: queue name]" -l                                          # list every option and its current value
```

**3. Defaults and housekeeping**

```
lpoptions -d "[FILL IN: default queue name]"                                     # set the default printer for this user
lpadmin -p "[FILL IN: queue name]" -o sides=two-sided-long-edge -o print-color-mode=monochrome
#   ^ duplex mono by default: the single highest-value consumables saving
lpstat -p -d                                                                     # all queues plus the default
lpstat -s                                                                        # scheduler status and device URIs
```

Duplex and monochrome as the *default* (users can override per job) is proposed
standard `[CONFIRM: proposed default]`.

The CUPS web interface at `http://localhost:631` is available for interactive
work, and System Settings → Printers & Scanners is what to walk a user through.

### Deploying printers via Munki

Queues can be deployed to the Mac fleet with Munki rather than added by hand on
each machine, which is the only way this stays consistent. See
`Windows & Mac Workstations/munki-server-administration-guide.md` for the repo
workflow; the printing-specific parts:

- **Driver packages.** A vendor `.pkg` imports into Munki like any other
  package: import via MunkiAdmin, set it to the `development` catalog, test,
  then promote. Nothing special — drivers are just packages.
- **Queue creation.** Munki supports printer definitions natively via the
  `printer` pkginfo type, which runs `lpadmin` on the client and removes the
  queue on uninstall. Where that is not used, the same effect comes from a
  package with a postinstall script containing the `lpadmin` lines above.
- **Order matters.** A queue that needs a vendor PPD must have the driver
  installed first — set the driver package as a `requires` dependency in the
  queue item's pkginfo, or the queue creation fails silently on a fresh machine
  and nobody notices until someone prints.
- **Prefer driverless.** A driverless queue has no dependency, no driver
  package to maintain, and does not break on a macOS upgrade. Reserve driver
  packages for the devices that genuinely need them.
- **Scope by group.** Put the queues for a floor or department in that group's
  manifest, not in `site_default`. People should get the printers near them.
- **Use `optional_installs` for the rest.** A user who needs the printer on
  another floor gets it from Managed Software Center without a ticket.
- `[FILL IN: are any printer queues currently deployed via Munki, or is every
  queue added by hand?]`

Test a Munki-deployed queue the same way as any package — a second
`managedsoftwareupdate` run must report nothing to do. A printer item whose
install check is wrong will recreate the queue on every check-in, which
duplicates it endlessly in the print dialog.

### Windows 11

**Direct IP queue (no print server):**

```powershell
Add-PrinterPort -Name "[FILL IN: port name]" -PrinterHostAddress "[FILL IN: printer hostname]"
Add-PrinterDriver -Name "[FILL IN: exact driver name as installed]"
Add-Printer -Name "[FILL IN: queue name]" -DriverName "[FILL IN: driver name]" -PortName "[FILL IN: port name]"
Set-Printer -Name "[FILL IN: queue name]" -Shared $false        # workstation queues are not shared
Get-Printer | Format-Table Name,DriverName,PortName,PrinterStatus
```

**Shared queue from a print server:**

```powershell
Add-Printer -ConnectionName "\\[FILL IN: print server]\[FILL IN: queue name]"
Get-Printer -ComputerName "[FILL IN: print server]"              # what the server offers
```

**Driver installation from an INF, without the vendor's installer:**

```
pnputil /add-driver "[FILL IN: path]\*.inf" /subdirs /install    # stage the driver package into the driver store
pnputil /enum-drivers                                             # confirm it landed; note the oemNN.inf name
```

Windows notes that cause real tickets:

- **Point and Print restrictions.** Since the PrintNightmare hardening, a
  non-admin cannot install a driver from a print server by default. Either
  pre-stage the driver on workstations (`pnputil` above, or by GPO), or use
  driverless IPP. Pre-staging is the reliable answer.
- **Prefer the "Microsoft IPP Class Driver"** for modern devices — it is the
  Windows equivalent of driverless and avoids the whole driver-distribution
  problem.
- **Type 3 vs Type 4 drivers.** Type 4 (v4) drivers are the modern model and do
  not need matching architecture packages on the server. Prefer them.
- **Architecture.** The Parallels VMs are Arm64. A printer with only x86/x64
  driver packages **has no driver on Arm64 Windows** — see
  `Windows & Mac Workstations/windows-11-desktop-runbook.md` §11 and §12. The
  documented workaround there is the right one: route printing through
  Parallels' shared printer bridge to the Mac's queue rather than fighting for a
  native driver in the VM. Where a native queue is needed, use the IPP class
  driver, which is architecture-independent.

### Print server or direct IP?

| | Direct IP queues | Central print server |
|---|---|---|
| Setup effort | Per workstation (unless deployed by Munki/GPO) | Once |
| Driver updates | Every workstation | One place |
| Accounting / job history | None | Available |
| Secure print release | Device-side only | Server-side possible |
| Single point of failure | No | **Yes — the server down means nobody prints** |
| Cross-platform | Each platform independently | Works, but mixed macOS/Windows driver management on one server is its own problem |

For a fleet of the organization's size with both macOS and Windows, **direct IP with
driverless queues, deployed by Munki on the Mac side and by GPO or pre-staged
drivers on the Windows side**, is the proposal `[CONFIRM: proposed default]`. It
removes a single point of failure and the cross-platform driver headache. Revisit
if job accounting or centralised secure-print release becomes a requirement.
`[FILL IN: what is the current arrangement?]`

## Scan-to-Folder and Scan-to-Email

### Scan-to-folder (SMB to files.example.com)

The device authenticates to an SMB share as a service account and writes the
scan there. Configuration lives in the device's web interface under a name like
"Scan to Network Folder" or "Save to Network Folder".

What to configure:

| Setting | Value | Note |
|---------|-------|------|
| Protocol | SMB (not FTP) | FTP sends the password in clear text. If a device only offers FTP, that is a reason to replace it |
| SMB version | SMB2 or SMB3 | SMB1 is disabled on modern DSM and should stay that way. An MFP that needs SMB1 is a real problem — see Troubleshooting |
| Server | `files.example.com` | Use the name, not the IP |
| Share / path | `[FILL IN: share and folder path for scans]` | A dedicated share, not a departmental one |
| Account | `[FILL IN: service account name]` | A dedicated service account. **Never a staff account** |
| Domain | `example.com` | |
| Filename pattern | `[FILL IN: e.g. scan-YYYYMMDD-HHMMSS]` | Include a timestamp or scans overwrite each other |
| Default format | PDF, `[FILL IN: dpi]` | Searchable PDF if the device supports OCR |

Set up on the Synology side (see
`Hardware & Backup/synology-nas-administration-guide.md`):

- A **dedicated share** for scans, not a subfolder of a departmental share.
  Scans arriving into a live working folder means a misconfigured MFP writes
  into member files.
- The service account gets **write access to that share and nothing else**. It
  does not need read access to the rest of the NAS, and should not have it.
- **Snapshots on the scan share**, so a scan deleted by accident is recoverable
  — the same capability that makes the NAS worth having.
- `[FILL IN: retention on the scan share — how long do scans stay before being
  filed or cleared?]` Proposed: scans older than 90 days are reviewed and
  cleared, because a scan share becomes a shadow document repository otherwise
  `[CONFIRM: proposed default]`.

**The security point that this section exists for:** the scan-to-folder service
account is one of the most commonly overlooked credentials in any organisation.
It is set once, typed into a printer's web interface, and then forgotten —
which means:

- It is **excluded from password rotation** because nobody remembers it exists.
- It is often given far more access than it needs, because "make it work" is
  faster than scoping a share.
- Its password is **stored on the printer**, retrievable by anyone who reaches
  the device's admin page or exports its configuration.
- It frequently ends up being a **staff member's own account** — so when that
  person changes their password or leaves, scanning breaks organisation-wide,
  and until then the printer holds a real user's credentials.
- It is rarely in the vault, so the one person who knows it is the one person
  who set it up.

Handling, per `Security & Hardening/secrets-management-guide.md`:

- Record it in the password manager, with the device it is configured on named
  in the entry, so rotation knows what it will break.
- Add it to the service-account inventory in
  `SysAdmin Procedures/Inventory_Asset_Reference.txt` §7.
- Scope it to the scan share only.
- Rotate it on the same cadence as other service accounts, and **update every
  MFP in the same change window** — this is the bit that gets missed, and it is
  why scanning breaks a week after a password change. See Troubleshooting.
- Proposed rotation: annually, and immediately on any suspicion of exposure
  `[CONFIRM: proposed default]`.
- `[FILL IN: what account do the MFPs currently use for scan-to-folder, and is
  it in the vault?]`

### Scan-to-email

The device submits to an SMTP relay and the relay delivers.

| Setting | Value | Note |
|---------|-------|------|
| SMTP server | `[FILL IN: mail.example.com or a dedicated relay]` | See `Mail & Messaging/zimbra-mail-administration-guide.md` |
| Port | 587 with STARTTLS, or 25 for an internal relay | `[FILL IN]` |
| Authentication | `[FILL IN: authenticated submission, or IP-allow-listed relay?]` | |
| From address | A dedicated address, e.g. `[FILL IN: scanner@example.com]` | Never a person's address |
| TLS | Required | |
| Max attachment size | `[FILL IN: what the relay accepts]` | Set the device limit below the relay's, or large scans fail silently |

Two design points:

- **Allow-listing the printer's IP on an internal relay** avoids storing another
  credential on the device. That is the better answer where it is available — one
  less secret on a device whose filesystem is reachable. It requires the printer
  VLAN rule above to be narrow, because an allow-listed IP is an open relay for
  anything that can spoof it on that segment.
- **Scan-to-email defeats every DLP control** by design — it takes a paper
  document and emails it anywhere. If the devices can send externally, that is a
  deliberate decision worth recording. Restricting scan-to-email to internal
  recipients, or to the sender's own address, is the common compromise
  `[CONFIRM: proposed default — internal recipients only]`.

Sender authentication matters here: mail from a printer through the relay must
still pass SPF and DKIM or it lands in spam. See
`Mail & Messaging/email-authentication-spf-dkim-dmarc.md`.

### Address book and LDAP lookup

MFPs can query AD by LDAP so users pick recipients rather than typing them.
If configured, it needs a bind account — **another commonly forgotten
credential**, subject to everything said above. Use LDAPS, a read-only account
scoped to the directory tree that holds staff, and record it in the vault. See
`Active Directory/ldap-connection-reference.md` and
`Security & Hardening/LDAPs certificate replacement procedure.txt` — an expired
LDAPS certificate breaks MFP address lookup along with everything else that
binds.

## Fax and e-Fax

Example Org has two fax paths and neither runs through a fax modem on a printer:

- **e-Fax** — the staff-facing path. Procedure is documented in
  the internal e-fax tip sheet. That
  tip sheet is the user-facing reference; do not duplicate it here.
- **DMS fax** — sending a fax from within the document management system,
  documented in `Dynamics GP & DMS/DMS Manual/Sending a fax through DMS.pdf`
  (and the `.doc` original). See `Dynamics GP & DMS/dynamics-gp-2018-admin-guide.md`
  for the surrounding system.

Administrative points not covered by either tip sheet:

- `[FILL IN: who is the e-fax provider, what is the account/number, and where
  are the portal credentials held?]`
- `[FILL IN: what fax numbers does Example Org hold, and where do inbound faxes land —
  a mailbox, a shared folder, or the DMS?]`
- **Analogue lines.** If any MFP still has a phone line connected for fax,
  record it — an analogue line is a path into the building that bypasses the
  network entirely, and it is usually forgotten until it appears on a phone
  bill. `[FILL IN: are any MFP fax lines still physically connected? If a device
  has a fax board and no line, note that too so nobody troubleshoots a fax that
  was never going to work.]`
- If an MFP does send faxes, its stored fax logs and any received-fax memory
  are part of what must be cleared at disposal (see below).

## Consumables and Maintenance

### Toner and supplies

Track supply levels rather than discovering them. Every device exposes them via
SNMP using the standard Printer MIB, which works across vendors:

```
snmpwalk -v2c -c [FILL IN: community string] <printer-ip> 1.3.6.1.2.1.43.11.1.1.6   # supply descriptions
snmpwalk -v2c -c [FILL IN: community string] <printer-ip> 1.3.6.1.2.1.43.11.1.1.8   # supply maximum capacity
snmpwalk -v2c -c [FILL IN: community string] <printer-ip> 1.3.6.1.2.1.43.11.1.1.9   # current level — compare against max for a percentage
snmpwalk -v2c -c [FILL IN: community string] <printer-ip> 1.3.6.1.2.1.43.10.2.1.4   # lifetime page count
```

The page count is worth recording monthly. It is what tells you a maintenance
kit is coming due before the device tells you, and it is the number a service
vendor will ask for.

Proposed thresholds `[CONFIRM: proposed defaults]`:

| Item | Threshold | Action |
|------|-----------|--------|
| Toner | Reorder at **20%** remaining | Below 20% a heavy print job can run a cartridge out mid-week |
| Toner | Minimum on-hand stock: **one spare cartridge per device per colour** | A device down for want of toner is an avoidable outage |
| Maintenance kit | Order at **90%** of rated life | Kits are not stocked locally by most vendors; allow lead time |
| Waste toner / staples (MFPs) | Order at **80%** | |
| Page count | Record **monthly** | Feeds both kit planning and the service contract meter read |

If a managed print contract is in place, supplies usually arrive automatically
on meter reads and these thresholds become a check that the vendor is doing
that, rather than a task. `[FILL IN: which applies at the organization?]`

Keep spare toner in a known place, recorded in the asset inventory, and **check
the part numbers against the inventory table** — the single most common
consumables mistake is buying the cartridge for the wrong model in a similar
series.

### Maintenance kits

A maintenance kit (fuser, rollers, transfer parts) is rated for a fixed page
count. After fitting one, the counter has to be reset by hand or the device
keeps reporting the old kit as worn out.

The existing library note, carried forward verbatim:

> **How To Reset the Maintenance Count on the M604 M605 M606 series**
>
> 1. Press the Home button on the printer's control panel.
> 2. Open the "Administration" menu.
> 3. Select "Manage Supplies".
> 4. Select "Reset Supplies".
> 5. Select "New Maintenance Kit".
> 6. Select "Yes" to reset the maintenance kit counter.

(Source: `Hardware & Backup/hp maintenance kit reset.txt`, content last modified
2026-07-06, UNVERIFIED.)

Points that note assumes but does not say:

- **Reset the counter only after the kit is physically fitted.** Resetting to
  silence a warning without fitting the kit means the next person has no idea
  how worn the fuser is, and fusers fail in ways that jam paper and,
  occasionally, smell like burning.
- Record the date, the page count at replacement, and who fitted it, in the
  Notes column of the inventory table. That history is what makes the next kit
  predictable.
- The menu path differs on other models and other vendors.
  `[FILL IN: the equivalent reset path for any non-M60x devices in the fleet]`

### Routine maintenance

Proposed schedule `[CONFIRM: proposed defaults]`:

| Cadence | Task |
|---------|------|
| Monthly | Record page counts; check supply levels; check the device error log in the web interface |
| Quarterly | Clean the ADF glass and rollers on MFPs (the cause of almost every "there's a line down my scans" ticket); check for firmware updates |
| Annually | Review the inventory table against reality; rotate the scan-to-folder service account; verify device admin passwords are not defaults |
| On replacement | Fit the kit, reset the counter, record it |

## Firmware Updates

Printer firmware is patched less than anything else on a network, and printers
have had real, exploited vulnerabilities: remote code execution via crafted
print jobs, authentication bypass on embedded web servers, credential
disclosure from configuration exports, and persistence surviving a factory
reset. A compromised MFP is a device inside the network that sees every
document printed and scanned through it.

Why it gets skipped, and the answer to each:

| Reason | Reality |
|--------|---------|
| "It's just a printer" | It is a networked Linux host with stored credentials and a copy of everything scanned |
| "It might brick it" | Real, but rare, and mitigated by updating one device at a time outside business hours |
| "Nobody asked for it" | Nobody asks for patching. That is why it needs a cadence, not a trigger |
| "There's no notification" | Correct — printers rarely notify. That is what the quarterly check is for |

Procedure:

1. Check the vendor's support page for the exact model and current firmware
   version (read the running version from the device's web interface or control
   panel — record both in the inventory Notes).
2. Read the release notes for the versions being skipped, not just the newest —
   security fixes are frequently described only in an intermediate release.
3. Schedule outside business hours. A firmware update takes a device out of
   service for 10–30 minutes and **must not be interrupted**.
4. Confirm the device is idle — no queued jobs, no held secure-print jobs. Held
   jobs are usually lost.
5. Apply through the device web interface, or via the vendor's fleet tool if
   the contract provides one.
6. **Do not power-cycle during the update.** This is how printers actually get
   bricked.
7. After: confirm the version, print a test page, send a test scan-to-folder and
   a test scan-to-email, and confirm the print queues still work from both
   platforms. Firmware updates have been known to reset network settings or
   disable SMB2.
8. Record the version and date in the inventory table.

Proposed cadence: check quarterly, apply security-relevant updates within 30
days, apply others at the next quarterly window `[CONFIRM: proposed default]`.
See `Security & Hardening/vulnerability-management-process.md` and
`SysAdmin Procedures/Patch_Management_Workflow.txt` — printers should appear in
both, and currently do not.

Firmware files should be downloaded by an administrator and pushed to the
device, not fetched by the device itself. That is why the printer VLAN denies
internet egress.

## Secure Print and End-of-Life Data

### Secure print / pull printing

Secure print holds a job at the device until the user authenticates at the
panel with a PIN or badge. Without it, a printed document sits in the output
tray from the moment it is sent until someone collects it — which for a shared
printer on a shared floor means member files, HR letters and payroll sitting
face-up in a corridor.

- **Where it matters most**: any device that prints member data, HR or finance
  documents. `[FILL IN: which devices are in shared or public-adjacent areas?]`
- Devices support it natively via a PIN entered in the driver at print time;
  the queue must be configured to expose the option, which driverless queues
  sometimes do not — this is one of the legitimate reasons to use a vendor
  driver.
- Held jobs expire. `[FILL IN: what is the hold timeout on each device?]`
  Proposed: hold for 12 hours, then delete `[CONFIRM: proposed default]`.
- Proposed: enable secure print on devices in shared areas, and make it the
  default queue for anything printing personnel or member records
  `[CONFIRM: proposed default]`.

### The MFP hard drive

**This is the part of printer administration with the highest consequence and
the least attention.** Most MFPs contain a hard drive or SSD, and it holds:

- Spooled copies of print jobs, often retained after printing
- Scanned documents, including anything scanned to folder or email
- Received faxes
- Address books, with staff names and email addresses
- Stored credentials — the scan-to-folder service account, the SMTP account, the
  LDAP bind account
- Job logs: who printed what, when

For Example Org that means an MFP disk can hold **member files, grievance
documentation, and personnel records**. A device sent back to a leasing company,
traded in, or put out as e-waste with that disk intact is a data breach, and it
is one that has happened to other organisations repeatedly and publicly.

**During service life:**

- Enable disk encryption if the device supports it. `[FILL IN: which devices
  support it, and is it enabled?]`
- Enable automatic job-data overwrite after each job — sometimes called "Image
  Overwrite" or "Secure Erase". This is the setting that stops the disk
  accumulating years of documents. `[FILL IN: enabled on which devices?]`
- Set job log retention to the minimum that is operationally useful.

**At end of life — mandatory sequence:**

1. **Confirm what the device holds.** Check the specification for a hard drive
   or SSD, and for a fax memory module. Do not assume a small device has none.
2. **Export nothing.** Do not take a configuration backup "just in case" — it
   contains the stored credentials in a portable file.
3. **Run the device's own secure erase / sanitisation** function — vendor names
   vary ("Secure Storage Erase", "Sanitize Disk", "Full Disk Overwrite"). A
   multi-pass overwrite takes hours; schedule it.
4. **Clear the address book, stored credentials, and job logs separately.** On
   many devices these live in NVRAM and are *not* covered by the disk erase.
5. **Factory reset** afterwards, to clear network settings and the admin
   password.
6. **Physically remove and destroy the drive** where the device is leaving the organization's
   control and the data warrants it. Proposed standard: any MFP that has handled
   member or personnel documents has its drive removed and destroyed rather than
   erased, because an erase is an assertion and a destroyed drive is a fact
   `[CONFIRM: proposed default]`.
7. **Rotate every credential that was on the device** — scan-to-folder, SMTP,
   LDAP bind — regardless of the erase. Per
   `Security & Hardening/secrets-management-guide.md`, a credential that has
   been on a device leaving your control is an exposed credential.
8. **Record the disposal**: device, serial, date, method, who performed it, and
   who witnessed it. `[FILL IN: does Example Org have a disposal record / certificate of
   destruction requirement? Ask whoever owns privacy compliance.]`

**Leased devices are the highest-risk case**, because the device leaves on a
schedule set by someone else and the leasing company collects it. Establish
before the end of the lease, in writing, who sanitises the drive and whether
Example Org may retain it. Many contracts allow drive retention for a fee, and it is
worth paying. `[FILL IN: are any Example Org MFPs leased, and what does the contract
say about the drive?]`

## Troubleshooting

### Symptom: print queue is stuck — jobs sit and do not print

**macOS:**

```
lpstat -o                                        # jobs waiting, across all queues
lpstat -p                                        # queue state — look for "disabled since"
lpq -P "[FILL IN: queue name]"                    # jobs on one queue
cancel -a "[FILL IN: queue name]"                 # cancel every job on that queue
cupsenable "[FILL IN: queue name]"                # re-enable a queue CUPS disabled after an error
tail -50 /var/log/cups/error_log                  # the actual reason it stopped
```

CUPS disables a queue after a failure and does not re-enable it. A "stuck"
queue is very often just a disabled one, and `cupsenable` fixes it in a second.
If jobs return after clearing, reset the spool:

```
sudo launchctl stop org.cups.cupsd                # stop the scheduler
sudo rm -rf /private/var/spool/cups/*             # clear the spool
sudo launchctl start org.cups.cupsd               # start it again
```

**Windows:**

```powershell
Get-PrintJob -PrinterName "[FILL IN: queue name]"           # what is queued
Remove-PrintJob -PrinterName "[FILL IN: queue name]" -ID [FILL IN: job id]
Get-Service -Name Spooler | Format-List Status,StartType     # is the spooler even running
```

```
net stop spooler                                             # stop the spooler
del /Q /F %systemroot%\System32\spool\PRINTERS\*.*           # clear the spool directory
net start spooler                                            # start it again
```

The spooler-clear sequence is the standard fix for a Windows queue that will
not drain, and it discards every queued job on the machine — tell the user
before running it.

If the same job jams the queue repeatedly on multiple machines, the job is
malformed (usually a bad PDF or an embedded font the device chokes on), not the
queue. Print it to PDF and re-print, or print as image.

### Symptom: cannot print from one specific application

- Likely causes, in order:
  1. The application uses its own print path. Adobe and some Java applications
     generate PostScript that a driver handles differently.
  2. The application is emulated on Arm64 (Parallels VMs, Windows on Arm) and
     the driver is not — see the runbook §11.
  3. The document contains a font or transparency the driver renders badly.
  4. The application has its own printer selection cached, pointing at a queue
     that no longer exists.
- Check: print a plain text document from the same application; print the same
  document from a different application. That pair of tests isolates
  application-vs-document immediately.
- Fixes, in escalating order:
  ```
  lpr -P "[FILL IN: queue name]" -o raw [FILL IN: file]     # macOS: bypass the driver entirely
  ```
  - Print to PDF, then print the PDF. Solves most cases, and is the fastest
    thing to tell a user.
  - In Adobe: "Print as Image" — slow, but works when nothing else does.
  - Switch the queue between driverless (`-m everywhere`) and the vendor
    driver. Applications that use advanced features often work with one and not
    the other.
  - On Arm64: use the IPP class driver, or route through the Parallels printer
    bridge to the Mac queue.

### Symptom: scan-to-folder stopped working after a password change

**This is the most predictable failure in this document.** The service account
password was rotated — during offboarding, a routine rotation, or a security
response — and the MFPs still hold the old one. Scanning breaks everywhere at
once, usually a day or two later when someone first tries to scan.

- Confirm it is this and not something else:
  ```
  nc -zv files.example.com 445                                    # is SMB reachable from the network at all
  smbclient -L //files.example.com -U "[FILL IN: service account]" # does the account authenticate with the password you hold
  ```
  Check the Synology log — DSM → Log Center — for failed authentication attempts
  from the printer IPs. Repeated failures from a printer address confirm it
  immediately.
- **Also check the account is not locked out.** Several MFPs retrying a stale
  password every few minutes will trip the AD lockout policy, which means fixing
  the password alone is not enough — the account has to be unlocked as well, and
  will re-lock if one MFP is missed.
  ```
  Search-ADAccount -LockedOut | Format-Table Name,LastLogonDate
  Unlock-ADAccount -Identity "[FILL IN: service account]"
  ```
- Fix: update the stored password **on every MFP**, in one pass, then unlock the
  account. Missing one device re-locks the account and it looks like the fix
  did not work.
- Prevent: the vault entry for the service account must list every device it is
  configured on, so the rotation procedure includes updating them. Add the MFPs
  to the offboarding checklist's "shared resources owned by this account" review
  — and if the scan account *is* a person's account, fix that first, because
  offboarding that person takes scanning down organisation-wide.

Related failure with the same symptom: **SMB version**. A DSM update that
disables SMB1 breaks an old MFP that only speaks SMB1. The Synology log shows
a protocol negotiation failure rather than an authentication failure — that is
how to tell the two apart. Do not re-enable SMB1 on `files.example.com`; the answer
is a firmware update on the MFP, or replacing it.

### Symptom: device unreachable

```
ping [FILL IN: printer hostname]                   # name resolves and host answers?
nslookup [FILL IN: printer hostname]                # is DNS pointing where you expect
nc -zv [FILL IN: printer ip] 9100                   # raw print port
nc -zv [FILL IN: printer ip] 443                    # embedded web server
arp -a | grep [FILL IN: printer ip]                 # is the MAC the one you expect, or has something else taken the address
```

Work outward:

1. **Is it powered on and showing a ready screen?** Genuinely the first check.
   A device in an error state stops answering on some ports and not others.
2. **Link light on the switch port.** A printer moved by cleaners or facilities
   is a recurring cause.
3. **Address conflict** — the `arp -a` check. A static printer inside a DHCP
   range is the classic version; see
   `Networking Guide/ipam-vlan-topology-reference.md` for the diagnosis.
4. **Lease expiry** — a device set to DHCP with no reservation gets a different
   address, and every queue built against an IP breaks. This is the argument for
   building queues against DNS names.
5. **VLAN or firewall** — if it answers from one VLAN and not another, it is a
   rule, not the printer. Check the printer VLAN policy above against the live
   ruleset.
6. **The device's own firewall / access list.** Many MFPs have an IP
   allow-list in their network settings. A firmware update can reset or enable
   it.

### Symptom: print spooler problems

**Windows** — the spooler crashing repeatedly is almost always a bad driver:

```powershell
Get-Service Spooler | Format-List Status,StartType
Restart-Service -Name Spooler -Force
Get-WinEvent -FilterHashtable @{LogName='System';ProviderName='Microsoft-Windows-PrintService'} -MaxEvents 20
Get-PrinterDriver | Format-Table Name,MajorVersion,PrinterEnvironment      # which drivers are installed
```

```
pnputil /enum-drivers                                  # find the oemNN.inf for the suspect driver
pnputil /delete-driver oem[FILL IN: NN].inf /uninstall /force   # remove it, then re-add a known-good one
```

Isolate by removing queues one at a time and restarting the spooler after each.
The queue whose removal stops the crashes identifies the driver. Prefer a v4 or
IPP class driver as the replacement rather than reinstalling the same one.

**macOS** — CUPS is more robust, but:

```
lpstat -s                                           # scheduler running? queues listed?
sudo launchctl kickstart -k system/org.cups.cupsd   # restart the scheduler
tail -100 /var/log/cups/error_log                   # errors from the scheduler itself
cupsctl --debug-logging                             # turn on verbose logging while reproducing
cupsctl --no-debug-logging                          # turn it back off — it is noisy
```

Last resort on macOS is System Settings → Printers & Scanners → right-click in
the printer list → **Reset printing system**, which removes every queue on the
machine. On a Mac whose queues come from Munki that is survivable — the next
check-in rebuilds them. On a Mac with hand-built queues it means rebuilding them
by hand, so establish which it is first.

### Symptom: scans have a line down every page

Not a network problem — it is dirt on the ADF glass strip, a narrow band
separate from the main platen. Clean it with a lint-free cloth and glass
cleaner (on the cloth, never sprayed into the device). If the line is present
from the ADF but not from the flatbed, this is certainly it. Included here
because it is a frequent ticket with a thirty-second fix that gets escalated as
a hardware fault.

## Security

- **Exposure**: printers should be internal only, on their own VLAN, with no
  internet egress and no inbound access from Guest. `[FILL IN: confirm current
  placement against the live pfSense ruleset]`
- **Admin passwords**: every device's embedded web server has an administrator
  account. Change it from the default, make each one unique, and store them in
  the password manager. A default printer password is a working credential on a
  device that holds other credentials. `[FILL IN: have all device admin
  passwords been changed from defaults?]`
- **Disable unused protocols** on each device: Telnet, FTP, raw FTP print, SNMP
  v1/v2c where v3 is available, IPv6 if unused, and any cloud print service that
  is not in use. Every one of those is a listener.
- **SNMP**: community strings are credentials. If SNMP is used for supply
  monitoring, restrict it to the monitoring host by IP and prefer v3.
  `[FILL IN: SNMP version and community string handling on the printer fleet]`
- **Stored credentials on devices**: scan-to-folder, SMTP, and LDAP bind
  accounts all live on the printers. Each must be dedicated, minimally scoped,
  recorded in the vault with the devices it is on, and rotated on schedule. See
  `Security & Hardening/secrets-management-guide.md`.
- **HTTPS on the embedded web server**: enable it and disable plain HTTP.
  Devices ship with self-signed certificates; a certificate from the internal CA
  is better where the device supports it. See
  `Security & Hardening/certificate-pki-lifecycle-guide.md`.
- **Print job confidentiality**: IPPS (`ipps://`) encrypts the job in transit;
  raw port 9100 does not. Prefer IPPS wherever the device supports it.
- **Physical security**: the output tray is an access control boundary. Secure
  print is the mitigation — see above.
- **Firmware** is a security control, not a feature update. See above.
- **Device data at disposal** — see End-of-Life Data. This is the highest-impact
  item in this section.

## Monitoring & Alerting

- Worth monitoring `[CONFIRM: proposed defaults]`:
  - Device reachability — a printer that stops answering ICMP or port 9100.
    Cheap to check and catches most faults before a user reports them.
  - Supply levels via the SNMP OIDs above; alert at the 20% toner and 90% kit
    thresholds.
  - Page counts, recorded monthly for planning and meter reads.
  - Queue depth on any central print server, if one exists.
  - Device error state — the Printer MIB exposes `hrPrinterDetectedErrorState`
    (`1.3.6.1.2.1.25.3.5.1.2`), which covers jams, covers open, and out-of-paper.
  - Repeated authentication failures from a printer IP against `files.example.com` —
    the early warning for the scan-to-folder password problem above, before
    users notice.
- `[FILL IN: is there an SNMP monitoring system that could poll printers? See
  SysAdmin Procedures/monitoring-alerting-guide.md]`
- What normal looks like: every device reachable, supplies above threshold, page
  counts increasing at a steady rate. A page count that stops increasing means
  a device nobody is using — worth knowing before renewing its service contract.

## Disaster Recovery

- **Losing a printer** is an inconvenience, not an outage, provided users can be
  pointed at another queue. That is only true if the inventory table exists and
  says where the other devices are. Proposed: every floor or area has a
  documented fallback device `[CONFIRM: proposed default]`.
- **Losing scan-to-folder** (the NAS or the share) stops scanning across the
  organisation. See `Hardware & Backup/synology-nas-administration-guide.md`.
  Fallback is scan-to-email.
- **Device configuration backup**: most MFPs can export their configuration.
  This is genuinely useful for rebuilding a replaced device — and it contains
  **stored credentials**, so the export is a secret. Store it in the vault or an
  access-controlled location, never on a general share, and delete it when the
  device is retired. `[FILL IN: are device configuration exports taken, and
  where are they kept?]`
- **Rebuild a replaced device**: address and DNS record → admin password →
  disable unused protocols → HTTPS → scan-to-folder → scan-to-email → firmware
  current → queues re-pointed (or DNS repointed, if queues use names — which is
  why they should) → test print, test scan-to-folder, test scan-to-email.
- **What is not recoverable**: anything held on the device only. Held secure
  print jobs, unforwarded received faxes, and the local address book are all
  lost when a device dies. Nothing should be designed to depend on them.
- Escalation if the owner is unavailable: `[FILL IN: name and contact]`; service
  vendor contact `[FILL IN: vendor, contract number, and support phone number]`

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why / Ticket |
|------------|-------------------|--------------|
| 2026-09-11 | Guide created. Absorbed the HP maintenance-kit reset note verbatim; the note file remains in place | Printing had no documentation beyond a six-line counter-reset note |
| `[FILL IN]` | `[FILL IN: when were the current devices installed, and what did they replace?]` | `[FILL IN]` |
| `[FILL IN]` | `[FILL IN: decision on print server vs direct IP]` | `[FILL IN]` |
| `[FILL IN]` | `[FILL IN: decision on managed print services contract]` | `[FILL IN]` |
| `[FILL IN]` | `[FILL IN: record any device disposal, with the sanitisation method used]` | `[FILL IN]` |

## References

- `Hardware & Backup/hp maintenance kit reset.txt` — the original M604/M605/M606
  counter-reset note, carried forward verbatim above
- `Hardware & Backup/synology-nas-administration-guide.md` — `files.example.com`,
  shares, permissions, snapshots; the scan-to-folder target
- `Networking Guide/ipam-vlan-topology-reference.md` — the Printers/IoT VLAN row
  and the inter-VLAN policy matrix; populate the addressing there, not here
- `Networking Guide/dns-dhcp-administration-guide.md` — reservations and DNS
  records for printers
- `Networking Guide/Network-Administration-Guide.md` — switch ports, VLAN
  membership, layered troubleshooting
- `Support Procedures/macos-desktop-support-runbook.txt` §15 — macOS printing
  troubleshooting, CUPS
- `Windows & Mac Workstations/windows-11-desktop-runbook.md` §11–12 — Arm64
  driver availability and the Parallels printer bridge
- `Windows & Mac Workstations/munki-server-administration-guide.md` — deploying
  drivers and queues to the Mac fleet
- `Security & Hardening/secrets-management-guide.md` — scan-to-folder, SMTP and
  LDAP bind credentials
- `Security & Hardening/vulnerability-management-process.md` and
  `SysAdmin Procedures/Patch_Management_Workflow.txt` — where printer firmware
  belongs
- `Security & Hardening/certificate-pki-lifecycle-guide.md` — device web server
  certificates
- `Active Directory/ldap-connection-reference.md` — MFP address book lookup
- `Mail & Messaging/email-authentication-spf-dkim-dmarc.md` — making
  scan-to-email deliverable
- the internal e-fax tip sheet —
  user-facing e-fax procedure
- `Dynamics GP & DMS/DMS Manual/Sending a fax through DMS.pdf` — faxing from the
  DMS
- `SysAdmin Procedures/Inventory_Asset_Reference.txt` — where device and service
  account records belong

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

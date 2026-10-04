> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Server Build Standard

## Overview

This is the standard for building a new server in this environment, from an empty Proxmox guest to a host that can be declared production. It exists because nothing in this library currently describes a build baseline. Two documents come close and neither is one: `Security & Hardening/Linux Setup and Hardening (Security Checklist) – Systems Knowledge Base.pdf` is a hardening checklist for a box that already exists, and `Security & Hardening/fresh-box-hardening-cheatsheet.txt` is an audit workflow (Lynis, ssh-audit, Trivy) run against a box that already exists. Both are used *inside* this standard, at the step where they belong. Neither tells you how to decide between a VM and a container, what to name the host, where to register it, what must be true before it carries production traffic, or how to retire it afterwards.

The document also covers decommissioning, deliberately in the same file. A build standard that stops at handover produces exactly the debris the organization already has: DNS records pointing at nothing, firewall rules for hosts that no longer exist, backup jobs for absent machines, monitoring that alerts on a box somebody turned off two years ago. Retirement is the mirror image of the build and belongs next to it.

Audience: the sysadmin building the host. Tier 3 depth — read the whole thing the first time, then use the condensed checklist at the end.

## Quick Facts

| Field            | Value                                                                                                                       |
|------------------|-----------------------------------------------------------------------------------------------------------------------------|
| Environment      | Applies to prod, staging and lab builds; the hand-off gate applies only to prod                                              |
| Location         | Default target: Proxmox VE cluster, server room. Windows Server builds: [FILL IN: are Windows Servers virtualised on Proxmox, on separate hardware, or both?] |
| Access           | Proxmox web UI at `https://[FILL IN: PVE node FQDN]:8006`; SSH to the built host as `orgadmin` with a key                     |
| Dependencies     | Proxmox cluster and its storage; Active Directory DNS (example.com) for name registration; NTP; the apt mirror / internet egress for patching; Graylog (graylog01.example.com) for log shipping; Retrospect for backup; internal CA or Let's Encrypt for TLS |
| Dependents       | Every service subsequently deployed on the host; the accuracy of the host inventory and of the firewall ruleset             |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                                                                                      |

## Scope

**Default target: an Ubuntu LTS guest on the Proxmox cluster**, VM or LXC, following the 24.04 LTS baseline referenced across the rest of this library. Everything in the numbered build sequence applies to that case unless noted.

**Docker host.** A Docker host is an Ubuntu guest built to this same standard, plus a container runtime and the operational differences that come with it:
- It is normally a **VM, not an LXC** (see the decision section below).
- Docker manipulates the host's packet filter directly. A Docker published port can bypass UFW rules, because Docker inserts its own rules ahead of them. Treat "the port is not in UFW" as no evidence at all that the port is closed — verify from another host with `nc -vz`, and bind containers to `127.0.0.1` where the service is only consumed locally or through a reverse proxy.
- Patching has two surfaces: the host OS, and the images. Host patching follows `SysAdmin Procedures/Patch_Management_Workflow.txt`; image patching is a rebuild-and-redeploy, scanned with Trivy per `Security & Hardening/fresh-box-hardening-cheatsheet.txt`.
- Backup covers volumes and compose files, not the containers. Record where the compose files live and make sure that path is in the backup set. See `Linux & Servers/docker-migration-procedure.md` for moving an existing stack.
- Add Docker Bench (CIS Docker benchmark) to the audit step — the command is in the hardening cheat sheet.

**Windows Server.** The base sequence still applies conceptually — name, address, DNS, time, patch, accounts, firewall, monitoring, backup, certificates, document — but the mechanics differ at every step, and the AD join is mandatory rather than optional. Time sync must come from the domain hierarchy, not from an independent NTP source, or Kerberos will break. Use `Windows & Mac Workstations/windows-server-2019-runbook.md` for the platform detail and this document for the sequence and the hand-off gate. A Windows Server build is a **Normal** change with a scheduled window, not an ad hoc task.

**Out of scope:** workstation builds (see `Windows & Mac Workstations/`), application installation and configuration (each service gets its own Tier 2/3 doc), and the Proxmox cluster itself (see `Linux & Servers/proxmox-cluster-administration-guide.md`).

---

## VM, LXC, or Container?

Decide this before provisioning; changing it later is a rebuild.

| Choose | When | Notes |
|---|---|---|
| **VM (KVM)** | Anything running Docker; anything needing its own kernel, kernel modules, or custom sysctls; anything running a database with specific I/O requirements; anything that must be migrated or snapshotted independently of the host kernel; anything internet-facing where the isolation boundary matters | The default when uncertain. Costs more memory; buys a real isolation boundary |
| **LXC container** | Lightweight, long-lived internal Linux services sharing the host kernel — internal web apps, small utility services, single-purpose daemons | Cheaper and faster to provision. Backup/restore is already documented in `Linux & Servers/lxc_backup_restore_proxmox91.txt`. Cannot run its own kernel; nesting Docker inside is possible but is a known source of subtle breakage and is not the standard here |
| **Docker container on an existing Docker host** | The workload is a packaged application that does not need its own OS lifecycle, and an appropriate Docker host already exists | Not a "server build" at all — no new host, no new inventory row, but it **does** need its own service documentation and it inherits the host's firewall and backup posture. Check that inheritance explicitly rather than assuming it |

Two rules of thumb. If the workload needs its own kernel behaviour, it is a VM. If the workload is internet-facing, it is a VM. Everything else can start as an LXC, and an LXC that outgrows those constraints is rebuilt as a VM rather than patched around.

---

## Naming and Addressing

**Hostnames.** The organization's existing servers use short, single-word, lowercase names in `example.com` — db01, db02, sign, appdb2, appdb1, erpdb, prod01, graylog01, files, forums, auth2, auth4, mail. Two patterns are visibly in use: evocative single words, and role-plus-number. Either is fine; mixing conventions inside one role is not.

[CONFIRM: proposed default naming convention — role-based names with a two-digit sequence (`graylog01`, `auth2`, `prod01`) for anything where a second instance is plausible, and single-word names only for singleton services. Names are lowercase, alphanumeric, no underscores, and must not encode the OS version, the hypervisor node, or the physical location, because all three change while the name does not.]

Names to avoid: anything containing `new`, `temp`, `test` (for a host that will become production), the year, or a person's name.

**Addressing.** [FILL IN: the server VLAN and its subnet — take the values from `Networking Guide/ipam-vlan-topology-reference.md`, not from memory]. Servers take **static addresses**, configured in netplan on the guest, not DHCP reservations — [CONFIRM: proposed default, matching the netplan examples in the existing Linux hardening checklist]. Record the allocation before you configure it, so two builds cannot claim the same address.

**Where a new host gets registered.** All of these, at build time, not later:

| Register in | What | Reference |
|---|---|---|
| Active Directory DNS (internal zone) | A record, and PTR in the reverse zone | `Networking Guide/dns-dhcp-administration-guide.md` |
| Public DNS zone | Only if the service is internet-facing — and only the public name | same |
| IPAM reference | The address allocation, so it is not reused | `Networking Guide/ipam-vlan-topology-reference.md` |
| Host inventory | The authoritative row for this host | `SysAdmin Procedures/Inventory_Asset_Reference.txt` |
| Proxmox | VMID, node, storage, and a description field naming the service and owner | `Linux & Servers/proxmox-cluster-administration-guide.md` |
| Retrospect | Backup client and media set membership | `Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md` |
| Graylog | Appears as a log source once the sidecar is running | `SysAdmin Procedures/monitoring-alerting-guide.md` |
| Certificate inventory | If it serves TLS | `Security & Hardening/certificate-pki-lifecycle-guide.md` |

Note that `SysAdmin Procedures/Inventory_Asset_Reference.txt` is currently unfilled boilerplate. It is nonetheless the intended home for the inventory row, and a build that adds a row to it is making it slightly less boilerplate. Do not invent a second inventory somewhere else.

---

## Base Build Sequence

Work in order. Steps 1–6 get the host to a state where it is safe to leave running; 7–13 are what make it supportable. A host that stops at step 6 is not built, it is merely booted.

### 1. Provision

- Decide VM or LXC (above). Record VMID, node, and the reason for the sizing.
- Size it: [CONFIRM: proposed defaults for a general-purpose Ubuntu guest — 2 vCPU, 4 GB RAM, 40 GB root disk, grown later rather than over-allocated now.] Data volumes are separate disks, not a larger root disk — that keeps the root filesystem small enough to snapshot and restore quickly, and it keeps `df` readable.
- Attach it to the server VLAN.
- Set the Proxmox description field: service, owner, build date, change ID.
- Set the guest to start on boot if it is production.

```
qm create <vmid> --name <hostname> --memory 4096 --cores 2   # VM; see PVE guide for full flags
pct create <vmid> <template> --hostname <hostname>            # LXC
```

### 2. OS install / template

- Ubuntu LTS. [FILL IN: confirm the current standard release — 24.04 LTS is referenced throughout this library; state whether new builds should now use a later LTS.]
- Prefer a maintained golden template over a fresh ISO install, so that steps 3–9 are already partly done and are identical across hosts. [FILL IN: does a current Ubuntu template exist in Proxmox, and what is its name/VMID? If not, building one is the highest-leverage improvement to this procedure.]
- If installing from ISO: minimal install, no desktop, OpenSSH server only. Do not install extras "in case".
- If cloning a template: regenerate the machine ID and SSH host keys, or every clone will share them.

```
sudo truncate -s 0 /etc/machine-id                   # clear cloned machine-id
sudo rm -f /etc/ssh/ssh_host_*                       # remove cloned host keys
sudo dpkg-reconfigure openssh-server                 # regenerate host keys
```

### 3. Hostname, hosts file, and DNS

```
sudo hostnamectl set-hostname <hostname>             # short name; FQDN comes from DNS + search domain
```

Add the host's own address to `/etc/hosts` alongside localhost, matching the existing convention in the Linux hardening checklist (loopback entry, then the host's own address with FQDN and short name). This keeps `sudo` and local tooling fast when DNS is unavailable.

Configure networking in netplan (`/etc/netplan/`): static address, default route, internal nameservers, and the search domains. The existing checklist records the search list as `example.com` and `example.org` — [CONFIRM: that both search domains still apply to new builds] — and records internal nameserver addresses that should be taken from `Networking Guide/dns-dhcp-administration-guide.md` rather than copied from an old sample file.

```
sudo netplan try                                      # validates and auto-reverts if you lose access
sudo netplan apply                                    # apply once 'try' is accepted
```

`netplan try` is the right command here for the same reason `sshd -t` is used later: it reverts automatically if the change costs you your session.

Then create the DNS records (forward and reverse) and confirm resolution works in both directions before continuing:

```
dig +short <hostname>.example.com                          # forward
dig +short -x <address>                               # reverse
```

### 4. Time sync

Clock skew breaks Kerberos, makes Graylog searches lie, and invalidates certificate checks. Fix it at build time, not when something breaks.

```
timedatectl                                           # confirm NTP is active and the timezone is right
sudo timedatectl set-timezone <Region/City>        # [CONFIRM: standard timezone for servers]
```

[FILL IN: the internal NTP source(s) servers should use — a domain controller, pfSense, or an external pool.] For a domain-joined host, time must come from the AD hierarchy; for a standalone Linux host, an internal source is preferred over the public pool so that the whole estate agrees with itself.

### 5. Patch to current

```
sudo apt update && sudo apt full-upgrade -y           # bring the box fully current before anything else
sudo apt autoremove -y                                # drop orphaned packages
[ -f /var/run/reboot-required ] && sudo reboot        # reboot now, not after the service is live
```

Then enable unattended security upgrades. The existing Linux hardening checklist records the exact `unattended-upgrades` configuration in use — allowed origins limited to the release and `-security` pocket, unused kernels and dependencies removed automatically, automatic reboot enabled at 02:00. Apply that configuration rather than re-deriving it, and note that **automatic reboot at 02:00 is a real behaviour with real consequences** for any service that does not come back cleanly on its own. [CONFIRM: whether automatic reboot should remain enabled for all new builds, or be disabled for hosts with an ordered start-up dependency — e.g. anything that needs a database up first.]

Ongoing patching then follows `SysAdmin Procedures/Patch_Management_Workflow.txt`. Record in the inventory which patch window the new host belongs to; a host in no window never gets patched.

### 6. Admin accounts and SSH keys

The organization's convention is a sudo-capable `orgadmin` account.

```
sudo adduser orgadmin                                 # create the admin account
sudo adduser orgadmin sudo                            # grant sudo
sudo groupadd sshusers                                # group the hardened sshd config allows
sudo adduser orgadmin sshusers                        # membership is required to log in after hardening
```

Install the authorised key **before** hardening sshd, and confirm key login works, or the next step locks you out:

```
ssh-copy-id -i ~/.ssh/<key>.pub orgadmin@<host>       # from the admin workstation
ssh orgadmin@<host> 'id'                              # prove key auth works before disabling passwords
```

Rules: no shared interactive accounts beyond `orgadmin` unless there is a reason recorded in the host's doc; no password authentication once keys are in place; root login disabled; service accounts are non-login (`--shell /usr/sbin/nologin`) and own only what they need. Key management and where private keys live is covered by `Security & Hardening/secrets-management-guide.md` — note the credential-exposure action item in `INDEX.md`, which is exactly what happens when this step is done casually.

### 7. sshd hardening

Do not hand-write an sshd policy per host. Use the existing drop-in: `Security & Hardening/sshd hardening conf.txt` holds the organization's `/etc/ssh/sshd_config.d/99-hardening.conf` — root login off, password auth off, pubkey only, `MaxAuthTries 3`, `AllowGroups sshusers`, X11/agent/tunnel forwarding off, verbose logging, and a restricted set of KEX algorithms, ciphers, MACs and host key algorithms.

Two things in that file need a decision per build rather than blind copying:

- **`Port 2222`.** If the standard is a non-default SSH port, every firewall rule, monitoring check, backup client and runbook has to agree with it. [CONFIRM: whether 2222 is the current standard for all servers, or whether that file reflects one host's configuration. Whichever it is, it must be consistent across the estate and reflected in the UFW rules in step 8.]
- **`AllowGroups sshusers`.** The group must exist and contain your admin account before the config is loaded (step 6 above).

Apply it safely — this is the one step in the build that can lock you out, and the method is already written down in `Security & Hardening/fresh-box-hardening-cheatsheet.txt`:

```
sudo cp <drop-in> /etc/ssh/sshd_config.d/99-hardening.conf   # place the drop-in, not an edit to sshd_config
sudo sshd -t                                                  # validate BEFORE applying; aborts on syntax error
sudo systemctl reload ssh                                     # reload, never restart — keeps sessions alive
```

Keep your first session open, then confirm login from a **second, brand-new** session before closing it. Then grade the result:

```
ssh-audit -l warn <host>                                      # expect no warn/fail lines
```

### 8. Host firewall

UFW on, default deny inbound, SSH limited to the networks that should reach it, then only the ports the service needs. The existing checklist records the pattern: `ufw limit` for SSH scoped to the server subnet and the VPN subnet, then explicit allows.

```
sudo apt install -y ufw                                       # if not already present
sudo ufw default deny incoming                                # deny by default
sudo ufw default allow outgoing                               # outbound open unless the service says otherwise
sudo ufw limit from <server-subnet> to any port <ssh-port>    # [FILL IN: server subnet from the IPAM reference]
sudo ufw limit from <vpn-subnet> to any port <ssh-port>       # [FILL IN: WireGuard subnet]
sudo ufw allow <service-port>/tcp                             # one line per port the service genuinely needs
sudo ufw enable                                               # turn it on
sudo ufw status verbose                                       # record this output in the host's doc
```

`limit` rather than `allow` for SSH gives basic rate limiting against brute force. If the sshd port is non-default (step 7), the UFW rules must use that port — a mismatch here is the most common way a new build becomes unreachable.

Two cautions. On a **Docker host**, UFW does not tell the whole story (see Scope). And a host firewall is not a substitute for the edge firewall: any rule needed on pfSense for this host goes through `Networking Guide/firewall-change-procedure.md`, with the host's purpose and expiry recorded in the rule description. Requesting those rules is part of the build, not a follow-up.

Consider fail2ban if the host exposes an authenticating service beyond SSH — the existing jails and filters are documented in `Security & Hardening/fail2ban filters UNBAN – Unjail yourself.pdf`, currently deployed on mail.example.com.

### 9. AD join (if applicable)

Applies to all Windows Servers and to any Linux host that needs domain authentication.

- Confirm DNS resolves the domain and its SRV records **before** attempting the join; nearly every failed join is a DNS problem.
- Confirm time is within tolerance of the domain — Kerberos will not tolerate meaningful skew.
- Place the computer object in the correct OU rather than the default Computers container, so that Group Policy applies as intended. [FILL IN: the OU new servers belong in]
- See `Active Directory/AD-Admin-Security-Guide.md` and, for LDAP binds from an application, `Active Directory/ldap-connection-reference.md`.
- For a Linux host that needs LDAP or Kerberos but not a full domain membership, prefer the narrower integration — a bind account with minimum rights — over joining the domain.

### 10. Monitoring and log shipping to Graylog

Every host ships logs to Graylog. A host that does not is invisible during an incident.

The mechanism is Graylog Sidecar managing Filebeat, with the collector configuration held centrally in Graylog and assigned by tag — the full procedure, including the sidecar install, the API token, the Beats input and the pipeline rule, is in `Security & Hardening/fresh-box-hardening-cheatsheet.txt` under "Shipping audit output to Graylog". Reuse it; do not invent a per-host syslog forwarder.

```
sudo dpkg -i graylog-sidecar_*.deb                    # install the sidecar
sudo nano /etc/graylog/sidecar/sidecar.yml            # server_url, API token, tags
sudo graylog-sidecar -service install                 # register as a service
sudo systemctl enable --now graylog-sidecar           # start it
```

[FILL IN: the Graylog server URL and port new hosts should use — existing runbooks reference both `graylog01.example.com` and `graylog.example.com:7555`; confirm which is current.] [FILL IN: the standard sidecar tag(s) for a general server build.]

Confirm the host appears under System → Sidecars as Active, and confirm messages are actually arriving — an Active sidecar with a broken file path ships nothing.

Beyond logs: `SysAdmin Procedures/monitoring-alerting-guide.md` is clear that the organization has centralised logging rather than centralised monitoring. Until that changes, the realistic minimum for a new host is [CONFIRM: proposed default — an uptime/port check for the service it provides, plus disk-space alerting; the existing Disk Monitoring Script referenced in the Linux hardening checklist is the current mechanism for the latter. Record which checks were configured in the host's Quick Facts.]

### 11. Backup enrolment (Retrospect)

A host that is not in a backup set is not a production host.

- Add the machine as a Retrospect client and confirm it appears in the console.
- Add it to both media sets — the onsite set and the offsite set — following the existing naming (`Server Backups 2026` onsite, `Server Backups Offsite 2026`). Restores come from the onsite set; the offsite set may require a disk from the offsite safe.
- Backups run daily at 02:00, so a day's files land in the **next** day's backup. That offset matters when you later restore.
- Decide and record **what** is backed up: the OS, or the OS plus data volumes, or data only with the OS rebuilt from this standard. For a guest on Proxmox there are two independent mechanisms — Retrospect at the file level and Proxmox/vzdump at the guest level (`Linux & Servers/lxc_backup_restore_proxmox91.txt`). Say explicitly which one is the restore path for this host; assuming both is how neither gets tested.
- [CONFIRM: proposed default — every new production host is added to the Retrospect onsite and offsite sets at build time, and a **test restore of one file** is performed before hand-off. An untested restore is a rumour.]
- Record the RPO the schedule actually delivers (daily at 02:00 means up to 24 hours of loss) in the host's Quick Facts, so nobody assumes better later.

See `Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md` for the restore mechanics.

### 12. Certificates (if it serves TLS)

- Determine which name(s) the certificate must cover, internal and external.
- Determine the issuer: Let's Encrypt for internet-facing names, internal CA for internal-only names. Check the CAA record on the public zone before assuming a public CA can issue.
- Issue and install per `Linux & Servers/reverse-proxy-and-tls-automation-guide.md`.
- Confirm automatic renewal **and** the reload hook. The failure mode that guide exists to catch is a certificate that renews on disk and is never reloaded into the running service.
- Add the certificate to the inventory table in `Security & Hardening/certificate-pki-lifecycle-guide.md` with its expiry date. No document in this library currently records expiry dates for anything; every new build is an opportunity to make that less true.
- Grade the result:

```
testssl.sh https://<host>.example.com                      # expect a clean grade; see the hardening cheat sheet
```

### 13. Audit before hand-off

Run the audit pass from `Security & Hardening/fresh-box-hardening-cheatsheet.txt` and fix what it finds, while the host is still empty and nothing depends on it. This is the cheapest moment in the host's life to fix hardening.

```
sudo lynis audit system --quick                       # host baseline; record the hardening index
ssh-audit -l warn localhost                           # SSH algorithm grade
sudo trivy rootfs --scanners vuln /                   # OS package CVEs
sudo lynis audit system --quick                       # re-run after fixes; the index should rise
```

Record the resulting Lynis hardening index in the host's documentation as the build-time baseline. A later drop in that number is a meaningful signal.

For a Docker host, add `trivy image` per service image and the Docker Bench CIS run. For a Windows Server, the equivalent baseline is [FILL IN: which Windows baseline/benchmark the organization applies — CIS, Microsoft security baseline via GPO, or neither].

---

## Hand-off: What Must Be True Before "Production"

A host is not production because it is running. It is production when all five of these are true and demonstrable. Anyone should be able to check them without asking the person who built it.

**1. Documented.**
- [ ] A Quick Facts entry exists for the host, in the doc for the service it runs (`Templates/tech-doc-template.md`).
- [ ] An inventory row exists in `SysAdmin Procedures/Inventory_Asset_Reference.txt`.
- [ ] DNS records exist, forward and reverse, and resolve.
- [ ] The Proxmox description field names the service and owner.
- [ ] Any pfSense rules created for this host reference a change ID and carry a purpose and an expiry (or "PERMANENT" with a reason).

**2. Backed up.**
- [ ] Enrolled in Retrospect, in both the onsite and offsite media sets.
- [ ] At least one successful backup has run and is visible in the console.
- [ ] A test restore of one file has been performed and is recorded with its date.
- [ ] The restore path (Retrospect file-level vs. Proxmox guest-level) is written down.

**3. Monitored.**
- [ ] Logs are arriving in Graylog — confirmed by finding a message from this host, not by the sidecar showing green.
- [ ] Disk-space monitoring is in place.
- [ ] A service check exists for whatever this host provides, and somebody receives its output.

**4. Patched.**
- [ ] Fully current as of the build date.
- [ ] `unattended-upgrades` configured, with a decision recorded about automatic reboot.
- [ ] Assigned to a patch window in `SysAdmin Procedures/Patch_Management_Workflow.txt`.

**5. Access-controlled.**
- [ ] Root login disabled; password authentication disabled; key-based access only.
- [ ] The hardened sshd drop-in is in place and `ssh-audit` is clean.
- [ ] UFW is enabled with a default-deny inbound policy, and `ufw status verbose` has been recorded.
- [ ] Only the intended people can log in, and each has their own key.
- [ ] Any secrets the service needs are stored per `Security & Hardening/secrets-management-guide.md` — not in this document, not in a text file on the host, not in the build notes.

The build is not finished until every box is ticked. [CONFIRM: proposed default — the sysadmin records the hand-off checklist result in the change record for the build, and the build is logged in `SysAdmin Procedures/Change_Management_Log.txt` as a Normal change.]

---

## The Documentation Obligation

Two artefacts, every new host, no exceptions.

**A Quick Facts entry.** Using the table from `Templates/tech-doc-template.md`: Owner, Environment, Location (host, VMID, URL, address), Access, Dependencies, Dependents, Last reviewed. If the host runs a service that warrants its own document, the Quick Facts table lives at the top of that document. If it does not, the host still needs the table — put it in the inventory entry.

The two fields that are always skipped and always matter are **Dependencies** and **Dependents**. Dependencies is what you check when this host breaks. Dependents is what you check before you touch it — and it is the field that makes decommissioning possible. A host whose Dependents field is honestly filled in can be retired safely; one whose field is empty can only be retired by turning it off and waiting to see who shouts.

**An inventory row.** In `SysAdmin Procedures/Inventory_Asset_Reference.txt`: hostname, FQDN, address, VM/LXC and VMID, Proxmox node, OS and version, purpose, owner, build date, patch window, backup set, monitoring status, and — critically — the decommission date if one is known.

That file is still generic boilerplate today. It remains the intended single source of truth for host facts, and the fastest route to resolving a large share of the `[FILL IN]` markers across this library is to fill it in. Adding each new host as it is built is the cheapest way to get there.

---

## Decommissioning

The mirror image of the build. Work in the reverse order of the sequence above, because the dependencies unwind in that order. **Do not start by turning the host off** — an unreachable host cannot tell you what still depended on it.

### 1. Establish what depends on it
- Read the Dependents field in the host's Quick Facts.
- Search the firewall ruleset and aliases for its address: `pfctl -sr | grep <address>`.
- Search DNS for records pointing at its address, including CNAMEs pointing at its name.
- Check for monitoring checks, backup jobs, cron jobs on *other* hosts that reach into this one, scripts referencing its hostname, application config referencing it, and anything mounting a share from it.
- Check whether any certificate covers its name.
- [CONFIRM: proposed default — a two-week notice period to affected users and service owners before any production host is retired.]

### 2. Quiesce, don't delete
- Stop the services. Leave the host running and reachable.
- [CONFIRM: proposed default — a 30-day quiet period with the services stopped but the host intact. Most missed dependencies surface within days, and recovery during the quiet period costs one `systemctl start` instead of a restore.]
- Watch the logs during the quiet period for connection attempts — that is the real dependency list, as opposed to the one on paper.

### 3. Take a final backup and keep it deliberately
- Run a final full backup and confirm it completed.
- [CONFIRM: proposed default — retain the final backup for 12 months, recorded in the inventory with the retention expiry date, then let it age out. A final backup with no recorded retention is either deleted too early or kept forever.]
- For a Proxmox guest, a final `vzdump` of the guest is the simplest complete artefact.

### 4. Unwind the registrations — in this order
1. **Remove firewall rules** on pfSense that exist for this host, via `Networking Guide/firewall-change-procedure.md`. Disable first, delete after the quiet period. **This is the step that, when skipped, produces the orphaned rules that the firewall procedure's periodic review exists to clean up.**
2. **Remove NAT entries** and any alias membership referencing the host.
3. **Remove DNS records**, forward and reverse, internal and public. Check for CNAMEs pointing at the name.
4. **Remove monitoring checks** so nothing alerts on an intentionally dead host — the fastest way to teach people to ignore alerts.
5. **Remove the Graylog sidecar registration** / stop expecting logs from it.
6. **Remove from Retrospect** — the client and the media set membership — *after* the final backup is secured.
7. **Remove the AD computer object**, if joined.
8. **Revoke certificates** issued to it, and remove its row from the certificate inventory.
9. **Revoke credentials**: service accounts it used, API tokens it held, database grants scoped to its address, SSH keys that existed only for it. Rotate anything shared that it knew — see `Security & Hardening/secrets-management-guide.md`.
10. **Release the IP address** in the IPAM reference, and note the date it was released. Do not immediately reissue it; a recycled address inherits every stale reference to the old host. [CONFIRM: proposed default — a 90-day cool-down before an address is reissued.]

### 5. Destroy
- Only after all of the above: stop and destroy the guest in Proxmox, and remove its storage.
- For physical hardware, follow media sanitisation before disposal. [FILL IN: the organization's media disposal/sanitisation process and who performs it.]

### 6. Close the record
- Mark the inventory row decommissioned, with the date, the change ID, and the final-backup retention expiry. **Do not delete the row** — a deleted row means that in six months nobody can tell whether the host ever existed, which is exactly the question that gets asked when a stale reference turns up.
- Log the decommission in `SysAdmin Procedures/Change_Management_Log.txt`.
- Update any document that referenced the host.

---

## Troubleshooting — Common Build Failures

### Symptom: locked out of SSH immediately after hardening
- Likely cause: key not installed before password auth was disabled; the admin user is not in `sshusers`; or UFW allows the old port while sshd now listens on a new one.
- Check: from the Proxmox console (which does not depend on sshd), `sudo sshd -T | grep -Ei 'port|passwordauth|allowgroups'` and `sudo ufw status verbose`.
- Fix: correct via console, `sudo sshd -t`, then `sudo systemctl reload ssh`.
- Prevention: the three-session method in `Security & Hardening/fresh-box-hardening-cheatsheet.txt` — keep session 1 open, validate, reload, verify from a fresh session.

### Symptom: host has no network after applying netplan
- Likely cause: YAML indentation, wrong interface name, or a gateway on the wrong subnet.
- Check: from the console, `ip addr`, `ip route`, `sudo netplan get`.
- Fix: correct the YAML and `sudo netplan try` — which reverts automatically if it costs you access.
- Prevention: always `netplan try`, never `netplan apply`, on a remote change.

### Symptom: hostname resolves internally but not externally, or vice versa
- Likely cause: example.com is split-horizon — internal and external views are separate zones with separate records.
- Check: `dig @<internal-resolver> <name>` against `dig @<external-resolver> <name>`.
- Fix: create the record in the view that is missing it. See `Networking Guide/dns-dhcp-administration-guide.md`.

### Symptom: AD join fails
- Likely cause: DNS (SRV records not resolvable from the host) or clock skew.
- Check: resolve `_ldap._tcp.example.com`; compare `timedatectl` against a domain controller.
- Fix: point the host at the internal resolvers and fix time sync, then retry. Do not work around it by joining with an IP address.

### Symptom: service unreachable from another host, but the port is listening
- Likely cause: UFW, the pfSense rule was never requested, or — on a Docker host — a container bound only to loopback.
- Check: `ss -tlnp` on the host; `sudo ufw status verbose`; `nc -vz <host> <port>` from the client; then the pfSense firewall log.
- Fix: work outward — host firewall first, then edge firewall via the firewall change procedure.

### Symptom: logs are not arriving in Graylog
- Likely cause: sidecar shows Active but Filebeat cannot read the log files, or the path glob is wrong, or the Beats input is not listening.
- Check: `journalctl -u graylog-sidecar`; confirm the sidecar user can read the target paths; confirm the Beats input is RUNNING in Graylog.
- Fix: grant read access (add to `adm` or set ACLs), correct the paths in the collector config, which is central — so the fix applies to every tagged host.

### Symptom: cloned guests behave strangely — duplicate SSH host key warnings, DHCP collisions, identical machine IDs
- Likely cause: the template was cloned without regenerating machine-id and SSH host keys.
- Fix: step 2 above, then reboot.
- Prevention: build the regeneration into the template's first-boot process.

### Symptom: disk fills shortly after go-live
- Likely cause: the root disk was sized for the OS and is now holding application data or logs.
- Check: `df -h`, then `du -xh / | sort -h | tail -20`.
- Fix: move data onto a separate disk (`Linux & Servers/Formatting and Mounting Disks in Linux – Systems Knowledge Base.pdf`) rather than growing the root disk. Confirm journald and application log rotation are bounded.

### Symptom: the host rebooted overnight without explanation
- Likely cause: `unattended-upgrades` with `Automatic-Reboot "true"` at 02:00 — a configured, documented behaviour, not a fault.
- Check: `journalctl -u unattended-upgrades --since yesterday` and `/var/log/unattended-upgrades/`.
- Fix: if this host must not auto-reboot, disable it for this host and record why (step 5).

### Symptom: backup shows the host but restores nothing useful
- Likely cause: only the OS is in the backup set, and the data lives on a volume that was never added.
- Check: the media set contents in the Retrospect console.
- Fix: add the volume and re-run. Then do the test restore that should have happened at hand-off.

---

## Security

- **Exposure:** default posture is internal-only. An internet-facing host is an explicit decision requiring an edge firewall rule, TLS, and ideally the Barracuda WAF in front of it.
- **Auth:** key-based SSH only; individual keys per person; `orgadmin` for administration; service accounts non-login and least-privileged.
- **Secrets:** never in this document, never in a build note, never in a file on the host that is backed up in the clear. See `Security & Hardening/secrets-management-guide.md` and the credential-exposure action item in `INDEX.md`.
- **Baseline evidence:** the Lynis hardening index at build time, recorded, so drift is detectable.
- **Attack surface:** install nothing speculatively. The existing checklist's list of packages to remove is a reasonable starting point for a minimal install — remove what the build does not need, before the service is live.
- **Patching** is a security control, not a maintenance chore; a host outside a patch window is an unmanaged host.

## Monitoring & Alerting

- Minimum per host: logs to Graylog, disk-space monitoring, and one service check.
- Baseline to record at hand-off: normal CPU/memory, normal disk usage, Lynis hardening index, and what a healthy log stream looks like — so that "abnormal" has a definition.
- The wider gap, and the fact that the organization currently has logging rather than monitoring, is documented in `SysAdmin Procedures/monitoring-alerting-guide.md`. Do not let a new build quietly assume monitoring that does not exist.

## Disaster Recovery

- **The build standard is the DR procedure.** A host rebuilt by following this document and then restoring data is the fallback when a guest is unrecoverable — which is why steps 1–13 must be reproducible rather than remembered.
- **RTO/RPO:** [CONFIRM: proposed defaults for a general-purpose internal server — RPO 24 hours, matching the 02:00 daily Retrospect cycle; RTO 8 hours for a full rebuild-and-restore. Record any host that needs better, and what mechanism delivers it, in that host's own documentation — the default is not a promise.]
- **Restore paths:** Proxmox guest-level restore (fastest, whole-guest) and Retrospect file-level restore (selective). Know which applies before you need it.
- **Escalation:** [FILL IN: escalation contact when the sysadmin is unavailable.]
- See `SysAdmin Procedures/Backup_DR_Runbook.txt` and `Hardware & Backup/power-outage-shutdown-runbook.md`.

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why |
|------------|-------------------|-----|
| 2026-09-11 | Standard created | No build baseline existed; the hardening checklist and cheat sheet are both post-build |
| 2026-09-11 | VM is the default; LXC by exception | Isolation and Docker compatibility matter more than the memory saving |
| 2026-09-11 | Five-point hand-off gate (documented, backed up, monitored, patched, access-controlled) | "Running" is not "production"; the gate is checkable by someone other than the builder |
| 2026-09-11 | Decommissioning included in the same document | Stale firewall rules and dead DNS records come from hosts that were switched off rather than retired |
| [FILL IN] | [FILL IN: record each material change to this standard] | [FILL IN] |

## References

- `Security & Hardening/Linux Setup and Hardening (Security Checklist) – Systems Knowledge Base.pdf` — the existing hardening checklist; the source for `orgadmin`, unattended-upgrades, UFW and netplan conventions
- `Security & Hardening/fresh-box-hardening-cheatsheet.txt` — Lynis / ssh-audit / Trivy audit workflow, safe sshd reload method, Graylog Sidecar setup
- `Security & Hardening/sshd hardening conf.txt` — the sshd hardening drop-in to deploy
- `Security & Hardening/secrets-management-guide.md` — key and credential handling
- `Security & Hardening/certificate-pki-lifecycle-guide.md` — certificate inventory and lifecycle
- `Linux & Servers/proxmox-cluster-administration-guide.md` — the hypervisor platform
- `Linux & Servers/lxc_backup_restore_proxmox91.txt` — LXC backup/restore and migration
- `Linux & Servers/docker-migration-procedure.md` — moving an existing Docker stack
- `Linux & Servers/reverse-proxy-and-tls-automation-guide.md` — TLS issuance, renewal and the reload hook
- `Linux & Servers/Formatting and Mounting Disks in Linux – Systems Knowledge Base.pdf` — adding data volumes
- `Networking Guide/firewall-change-procedure.md` — requesting edge firewall rules, and removing them at decommission
- `Networking Guide/ipam-vlan-topology-reference.md` — VLANs and address allocation
- `Networking Guide/dns-dhcp-administration-guide.md` — DNS registration, split-horizon behaviour
- `Active Directory/AD-Admin-Security-Guide.md`, `Active Directory/ldap-connection-reference.md` — domain join and LDAP integration
- `Windows & Mac Workstations/windows-server-2019-runbook.md` — Windows Server platform detail
- `Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md` — backup media sets and restore mechanics
- `SysAdmin Procedures/Patch_Management_Workflow.txt` — patch windows and procedure
- `SysAdmin Procedures/Change_Management_Log.txt` — change record template (currently unfilled boilerplate)
- `SysAdmin Procedures/Inventory_Asset_Reference.txt` — intended host inventory (currently unfilled boilerplate)
- `SysAdmin Procedures/monitoring-alerting-guide.md` — what is and is not monitored
- `Templates/tech-doc-template.md` — the Quick Facts table every host needs

---

## Condensed Build Checklist

Copy this into the build's change record and work down it. Bracketed values come from the sections above — do not guess them.

```
# ---- 0. Before you start -------------------------------------------------
# [ ] Purpose, owner, environment agreed and written down
# [ ] VM / LXC / container decision made and recorded
# [ ] Hostname chosen; address allocated from the IPAM reference
# [ ] Change record opened in Change_Management_Log.txt

# ---- 1-2. Provision ------------------------------------------------------
# [ ] Guest created on Proxmox; VMID, node, sizing recorded
# [ ] Proxmox description set: service | owner | build date | change ID
# [ ] Ubuntu LTS installed or template cloned
sudo truncate -s 0 /etc/machine-id                 # cloned guests only
sudo rm -f /etc/ssh/ssh_host_* && sudo dpkg-reconfigure openssh-server

# ---- 3. Identity and network --------------------------------------------
sudo hostnamectl set-hostname <hostname>           # short name
# [ ] /etc/hosts updated with own address + FQDN + short name
# [ ] netplan: static address, gateway, internal nameservers, search domains
sudo netplan try                                    # auto-reverts if you lose access
# [ ] Forward and reverse DNS records created
dig +short <hostname>.example.com                        # verify forward
dig +short -x <address>                             # verify reverse

# ---- 4. Time -------------------------------------------------------------
sudo timedatectl set-timezone <Region/City>      # [CONFIRM standard]
timedatectl                                         # NTP active, no skew

# ---- 5. Patch ------------------------------------------------------------
sudo apt update && sudo apt full-upgrade -y
sudo apt autoremove -y
[ -f /var/run/reboot-required ] && sudo reboot
# [ ] unattended-upgrades configured per the hardening checklist
# [ ] Auto-reboot decision recorded
# [ ] Patch window assigned

# ---- 6. Accounts and keys ------------------------------------------------
sudo adduser orgadmin && sudo adduser orgadmin sudo
sudo groupadd sshusers && sudo adduser orgadmin sshusers
# from the admin workstation:
ssh-copy-id -i ~/.ssh/<key>.pub orgadmin@<host>
ssh orgadmin@<host> 'id'                            # key auth MUST work before step 7

# ---- 7. sshd hardening ---------------------------------------------------
# keep this session open for the rest of this step
sudo cp <drop-in> /etc/ssh/sshd_config.d/99-hardening.conf
sudo sshd -t                                        # validate before applying
sudo systemctl reload ssh                           # reload, never restart
# [ ] Verified login from a NEW session before closing the old one
ssh-audit -l warn <host>                            # expect clean

# ---- 8. Host firewall ----------------------------------------------------
sudo apt install -y ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit from <server-subnet> to any port <ssh-port>
sudo ufw limit from <vpn-subnet> to any port <ssh-port>
sudo ufw allow <service-port>/tcp
sudo ufw enable && sudo ufw status verbose          # record this output
# [ ] Any required pfSense rules requested via firewall-change-procedure.md

# ---- 9. AD join (if applicable) -----------------------------------------
# [ ] DNS SRV records resolve; clock within tolerance; correct OU

# ---- 10. Logging and monitoring -----------------------------------------
sudo dpkg -i graylog-sidecar_*.deb
sudo nano /etc/graylog/sidecar/sidecar.yml          # server_url, token, tags
sudo graylog-sidecar -service install
sudo systemctl enable --now graylog-sidecar
# [ ] Host visible in Graylog AND a real message found from it
# [ ] Disk-space monitoring in place
# [ ] Service check configured, output goes to a person

# ---- 11. Backup ----------------------------------------------------------
# [ ] Retrospect client added; onsite AND offsite media sets
# [ ] One successful backup observed
# [ ] Test restore of one file performed and dated
# [ ] Restore path (Retrospect vs Proxmox) written down

# ---- 12. Certificates (if it serves TLS) --------------------------------
# [ ] Issued, installed, auto-renewal AND reload hook verified
# [ ] Added to the certificate inventory with its expiry date
testssl.sh https://<hostname>.example.com

# ---- 13. Audit -----------------------------------------------------------
sudo lynis audit system --quick                     # record the hardening index
sudo trivy rootfs --scanners vuln /
# Docker hosts: trivy image <each image>; docker-bench-security

# ---- 14. Hand-off gate ---------------------------------------------------
# [ ] DOCUMENTED   — Quick Facts entry + inventory row + DNS + Proxmox notes
# [ ] BACKED UP    — enrolled, ran, test-restored
# [ ] MONITORED    — logs arriving, disk watched, service checked
# [ ] PATCHED      — current, unattended-upgrades on, in a patch window
# [ ] ACCESS-CTRL  — keys only, root off, UFW deny-by-default, ssh-audit clean
# [ ] Change record closed
```

---
*Tier 3 document. Review annually or after any material change to the build platform.*

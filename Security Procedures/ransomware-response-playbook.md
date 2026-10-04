> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Ransomware Response Playbook

## Overview

This playbook covers the part of a ransomware incident that no other document in
this library covers: **deciding what to do while systems are encrypted.**
`Security Procedures/IR_Security_Scenarios_Guide.md` scenario 6 covers detection
and the first commands on a single host. `Security & Hardening/endpoint-security-operations-guide.md`
covers isolating an endpoint and preserving its evidence.
`SysAdmin Procedures/Backup_DR_Runbook.txt` covers restoring a system that failed
for ordinary reasons. None of them answers the questions that actually get asked
in hour one of a ransomware event: do I pull the plug or cut the network, which
backups are still trustworthy, how far back do I have to go, who decides whether
we pay, and in what order do systems come back.

The audience is whoever is standing in front of the problem — most likely one
person, at night, with users calling. It is written to be read under pressure:
the sequence matters more than the explanation, and the explanation is there only
where a wrong choice is irreversible.

Two things about the organization shape everything below. First, `erpdb.example.com` holds payroll
and member personal data, which makes this a privacy incident as well as an
availability incident from the first minute. Second, the organization's recovery capability is
real but layered and manual — Retrospect, Synology Snapshot Replication, Proxmox
guest backups, MySQL dumps plus binlogs — and modern ransomware attacks all four
of those deliberately. **Protecting the backups is the first priority after
isolation, ahead of investigation, ahead of restoring anything.**

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Owner            | the sysadmin (it@example.com) — incident commander by default until leadership names one `[CONFIRM: proposed default]` |
| Environment      | prod — whole estate. There is no non-production version of this event |
| Location         | This file, plus a printed copy kept offline: `[FILL IN: where the printed copy of this playbook and the first-hour card are kept — a playbook that only exists on an encrypted file share is not a playbook]` |
| Access           | Assume the vault, MDM, AV/EDR console, Graylog and mail may all be unavailable or untrustworthy. Break-glass credentials and out-of-band comms are prerequisites — see `Security & Hardening/secrets-management-guide.md` |
| Dependencies     | Retrospect (Mac + Linux clients); Synology Snapshot Replication on files.example.com; Proxmox LXC/VM backups; MySQL dumps + binlog copies; Graylog graylog01.example.com for timeline; pfSense/switching for isolation; AD (auth2.example.com, auth4.example.com) for everything else |
| Dependents       | Dynamics GP and DMS (erpdb.example.com — payroll and member data), Zimbra (mail.example.com), files.example.com, sign.example.com, prod01.example.com, app02.example.com, app03.example.com, data.example.com, db02.example.com, forums.example.com, and the whole macOS/Windows 11 endpoint fleet |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                               |

## How It Works

### What a ransomware incident actually is

Encryption is the last stage, not the incident. By the time files stop opening,
an operator has usually been inside for days or weeks. The shape is consistent
enough to plan against:

```
Initial access      # phished credential, exposed service, or a malicious document
  -> Execution      # payload runs on one endpoint — patient zero
  -> Persistence    # LaunchAgent/LaunchDaemon, scheduled task, service
  -> Credential theft   # local creds, cached domain creds, anything typed on that host
  -> Privilege escalation to domain admin
  -> Discovery      # AD enumeration, file shares, backup infrastructure
  -> Backup destruction     # THIS happens before encryption, not after
  -> Exfiltration   # data stolen for the second extortion lever
  -> Encryption     # mass file modification, the part users notice
```

Three consequences follow, and they drive the rest of this document:

1. **The dwell time is the problem.** A restore point taken the day before
   encryption almost certainly contains the attacker's persistence. Clean means
   "predates the intrusion", not "predates the encryption".
2. **The backups are a target, not a safety net.** Deleting snapshots, stopping
   backup jobs, and encrypting the backup repository are standard steps in the
   attack, done with the domain admin credentials stolen earlier.
3. **Exfiltration means this is a privacy incident even if recovery goes
   perfectly.** Restoring from backup in four hours does not undo a copy of the
   DMS member database leaving the building.

### Where the organization is exposed, and where the organization is strong

| Layer | What exists | What ransomware does to it |
|-------|-------------|----------------------------|
| Synology Snapshot Replication (files.example.com) | Btrfs snapshots, near-instant restore | Snapshots survive host-side encryption — this is the organization's best asset. They do **not** survive a DSM admin who deletes them, which is why snapshot locking matters (`Hardware & Backup/synology-nas-administration-guide.md` records this as an open `[FILL IN]`) |
| Retrospect | Daily 2:00 AM, onsite "Server Backups 2026" + offsite "Server Backups Offsite 2026", offsite disks in the offsite safe | The onsite set is reachable over the network and can be encrypted or overwritten with encrypted data. The offsite disk in the safe is air-gapped and is the strongest restore point the organization holds |
| Proxmox LXC/VM backups | vzdump to `[FILL IN: vzdump target storage — local, NAS-backed, or PBS?]` | If the target is NAS-backed or otherwise network-reachable from a compromised host, treat it as compromised until proven otherwise |
| MySQL | Nightly dumps + binary log copies (`Databases/mysql-administration-guide.md`) | Dumps on the host filesystem encrypt with everything else. Binlogs enable point-in-time recovery **only** if a clean dump and the intervening binlogs both survive |
| AD (auth2, auth4) | System State backups per `SysAdmin Procedures/Backup_DR_Runbook.txt` §2.1 | Domain compromise means AD is rebuilt or restored, and `krbtgt` is reset twice regardless |

### The three questions this playbook exists to answer

```
1. Is the damage still spreading?        -> Immediate Actions
2. Do we have a clean restore point?     -> Protect the Backups, then Verify a Restore Point
3. What are we obliged to tell whom?     -> Notification and Regulatory Obligations
```

Everything else — strain identification, forensic depth, the ransom note, the
attacker's portal — is secondary to those three and can wait.

---

# PHASE 1 — RECOGNITION

## What distinguishes ransomware from other incidents

Most incidents are quiet. This one has a loud signature, and the signature is
**mass file modification by a small number of processes**. A single compromised
host beaconing to C2 touches almost no files. Ransomware rewrites thousands of
them per minute.

Confirm ransomware rather than assuming it, because the response is drastic:

| Signal | Ransomware | Something else |
|--------|-----------|----------------|
| Many files renamed with a common new extension | Yes | A bulk rename script, a sync client gone wrong |
| A ransom note dropped in every affected directory | Decisive | Nothing else does this |
| File *contents* are high-entropy, not just renamed | Yes | A rename-only "attack" is a hoax or a test |
| Mass modification with no corresponding user activity | Yes | A scheduled migration, a Retrospect restore writing over a share |
| Backup jobs and shadow copies failing at the same time | Yes — deliberate | A coincidence worth doubting |

A ransom note plus unreadable files is decisive. Renamed-but-readable files are
not — check one file's contents before triggering a full response, because a
botched sync or a mass rename looks identical from the helpdesk queue.

```
file /path/to/suspect-file                              # a real encrypted file reports "data", not its original type
head -c 64 /path/to/suspect-file | xxd | head -4        # look for a known header vs random bytes
```

## Early signals, in the order they usually arrive

**User reports.** Almost always first. "My files have weird names", "Word says
the file is corrupted", "there's a text file in every folder". Two or more of
these from different people inside an hour is the trigger, not a coincidence.
Ask one question immediately: *are the files on your Mac/PC, or on the
S: drive?* Local-only is one host; share files mean it is already spreading
through files.example.com.

**Backup job failures.** Retrospect scripts failing, or completing with a
suspicious jump in the amount of data backed up. A backup that suddenly gets much
larger because every file "changed" is a mass-encryption event being faithfully
backed up over the clean copy. Check the Retrospect console's Activities and the
script logs on the Retrospect (Archive) machine.

**Snapshot Replication anomalies on files.example.com.** Snapshot space consumption
jumping far above baseline is the same signal from the storage side — copy-on-write
snapshots grow in proportion to how much data has changed. Also watch for
snapshot jobs failing, snapshots missing, or the retention list suddenly shorter
than it should be. A shrinking snapshot list is not a housekeeping event; it is
someone deleting them.

```
# DSM -> Snapshot Replication -> Snapshots -> [share]   : count and timestamps against the retention policy
# DSM -> Storage Manager -> Volume                      : snapshot space consumed vs the usual figure
# DSM -> Log Center                                     : snapshot deletions, admin logins, SMB session counts
```

**Graylog patterns.** `[FILL IN: Graylog stream and saved searches for each of the patterns below — monitoring-alerting-guide.md records the stream list as [FILL IN] as well, and this is the point at which that gap costs real time]`

Patterns worth having as saved searches before the day you need them:

- Authentication spikes and failures against auth2.example.com / auth4.example.com, and
  successful logons for one account from many hosts in a short window.
- Any break-glass account authentication (the highest-value single alert in the
  environment — `Security & Hardening/secrets-management-guide.md`).
- Service accounts authenticating from hosts that have never used them.
- Snort alerts from pfSense on internal-to-internal SMB, and lateral movement
  tooling signatures.
- Mass SMB session churn against files.example.com from a single client.
- Windows event IDs for shadow-copy deletion and backup catalogue deletion
  `[FILL IN: are Windows event logs forwarded to Graylog? endpoint-security-operations-guide.md records this as unconfirmed]`.

**Endpoint signals.** AV/EDR "ransomware behaviour" detection is Critical by
definition (`Security & Hardening/endpoint-security-operations-guide.md`). So is
an EDR agent found disabled — that is frequently the step immediately before
encryption, not an agent fault.

**Host-level confirmation**, once you have a host in hand:

```
sudo lsof | grep -iE "\.encrypted|\.locked|\.crypt" | head -20     # what is holding the renamed files open
ps aux | sort -k3 -rn | head -10                                   # sustained high CPU is the encryption itself
sudo fs_usage -w -f filesys | head -50                             # macOS: live file activity, find the writer
find / -name "*README*RECOVER*" -o -name "HOW_TO_DECRYPT*" 2>/dev/null | head   # locate the ransom note
```

Do not open the ransom note in an application. Read it with `cat` or `less`, and
photograph it if you need to show someone.

---

# PHASE 2 — IMMEDIATE ACTIONS (FIRST 15 MINUTES)

Do these in order. Do not stop to investigate between them. Everything here is
reversible except step 4, and step 4 is the one that has to be deliberate.

**1. Note the time.** Wall-clock, written down, along with what you saw and who
reported it. Every timeline in the next month is anchored to this.

**2. Declare it.** Say the word "ransomware" out loud to `[FILL IN: IT lead / senior management contact]`
by phone, not email `[CONFIRM: proposed default — phone/SMS, because mail.example.com may itself be affected or untrustworthy]`.
Do not wait for confirmation of scope. A false alarm costs an apology; a delayed
declaration costs restore points.

**3. Stop the bleeding at the network layer — isolate, do not power off.**
Cut the affected hosts' network reachability while leaving them powered on. The
decision rule and the reasoning are in the next section; the default is
**isolate**.

```
# Preferred — network-level, works even against a hostile host, needs no access to it:
#   move the port/MAC to the quarantine VLAN, or reject its 802.1x authentication
#   [FILL IN: quarantine VLAN ID and the exact steps — same gap as endpoint-security-operations-guide.md]

# macOS, hands on the machine:
sudo ifconfig en0 down                             # wired interface down, machine stays powered on
networksetup -setairportpower en1 off              # Wi-Fi radio off — check for a second interface

# Windows:
Get-NetAdapter | Where-Object Status -eq 'Up'      # find every live adapter first
Disable-NetAdapter -Name "<AdapterName>" -Confirm:$false   # down, machine stays powered on
```

Pulling the cable and switching Wi-Fi off is equally valid and faster. Check for
a second interface — a dock, a USB adapter, a tethered phone, a VPN client
reconnecting.

**4. Protect the file share before protecting anything else.** If files on
files.example.com are being encrypted and you cannot identify and cut the client
within about two minutes `[CONFIRM: proposed default]`, cut the **NAS** off the
network instead of hunting the client. Disabling SMB on the NAS stops every
encrypting client at once and keeps DSM, its snapshots and its logs intact.

```
# DSM -> Control Panel -> File Services -> SMB -> disable        # stops all SMB clients at once; DSM stays up
# Do NOT power the NAS off — snapshots, logs and DSM management all stay available if it is running
```

Protecting the data store beats protecting one endpoint. You can always find
patient zero afterwards; you cannot un-encrypt a share.

**5. Halt every backup job.** Before anything else in the backup estate — see
Phase 3. A running Retrospect script is at this moment backing up encrypted files
over your clean set.

**6. Do not log in anywhere with a privileged account.** Assume domain admin is
compromised. Every privileged credential typed on a suspect host from this point
forward is handed to the attacker. Work from a known-clean machine, with an
account scoped to what you are doing.

**7. Do not reboot, do not "just try" a restore, do not run a cleaner.** Rebooting
destroys the process list, open connections and memory. Restoring on top of a
live infection re-encrypts the restored data and burns the restore point. Cleaning
tools destroy evidence and do not reliably remove persistence.

**8. Do not touch the ransom note, the attacker's portal, or any contact address.**
Not to "see what they want", not from a work machine, not at all. Contact with the
operator is a decision made later, by named people, on advice — see Phase 6.

**9. Start a written log.** A plain text file on a clean machine, or paper.
Timestamp every action and every observation. `[FILL IN: where the incident log
is kept — it must not be on an affected system or on files.example.com]`

**10. Preserve, then scope.** Only once the spread has stopped do you move to
evidence (Phase 5) and scoping (Phase 4). Both are useless if encryption is still
running while you do them.

## Isolate vs. shut down — the tradeoff, and the rule

This is the decision people get wrong under pressure, in both directions.

**Powering off destroys volatile memory.** What is lost with it:

- Encryption keys that some families hold in RAM, and which have occasionally
  made free decryption possible. Rare, but you cannot get the chance back.
- The malware's configuration, injected code, and unpacked payload — often the
  only place the strain and its capabilities are legible.
- The list of network connections and C2 destinations that tells you where else
  it went.
- Credentials in memory that reveal which accounts the attacker actually used.
- On a FileVault Mac, powering off also drops the volume key, which makes later
  disk-level forensics substantially harder.

**Staying online lets the damage continue.** Encryption keeps running, and — more
dangerously — a human operator who notices you responding can react: accelerate
encryption, delete backups, wipe logs, or trigger destruction.

**Isolation is the resolution of the tradeoff**, which is why it is the default:
it takes the operator's hands off the machine and stops lateral spread while the
memory stays intact. It does not stop encryption of local disk on that host, which
is the accepted cost.

**The decision rule:**

| Situation | Action |
|-----------|--------|
| Default, any host | **Isolate.** Network off, power on |
| Host is actively encrypting, network isolation is achievable in the next couple of minutes | Isolate. Then, if encryption continues locally and the host holds data that matters, kill the encrypting process by PID before considering power |
| Host is actively encrypting a **share**, and you cannot cut its network fast enough | Cut the **NAS** off SMB instead (step 4 above). Protect the many, not the one |
| Host is a Proxmox **VM** | `qm suspend <vmid> --todisk` — freezes execution *and* writes memory to disk. This is the best of both and is the only place the organization gets a free memory capture |
| Host is a Proxmox **LXC container** | No equivalent memory snapshot. Isolate at the network/bridge level; `pct stop` is a hard stop and loses memory |
| Host is a **domain controller** (auth2, auth4) or **erpdb.example.com** | Do not act alone. Isolating a DC or the payroll/member database has estate-wide consequences — get a second person on the phone first `[CONFIRM: proposed default]` |
| Encryption is demonstrably **finished** on that host — note dropped, no active writer | Isolate only. There is nothing left to stop, and memory is now pure upside |
| Physical destruction risk: fire, water, thermal, or a hardware fault | Power off. Safety and hardware outrank evidence, always |

**Only pull power when the alternative is worse.** "It felt safer" is not a
reason. If you do power off a host, write down why, and the time — the
investigation will need to know that the memory was deliberately sacrificed.

---

# PHASE 3 — PROTECT THE BACKUPS (FIRST PRIORITY AFTER ISOLATION)

Do this before scoping, before evidence collection, before answering anyone's
questions. Modern ransomware targets backups on purpose and with stolen domain
admin credentials, and the window in which a clean restore point still exists is
the window you are in right now.

The objective is not to restore anything. It is to make the surviving restore
points **unreachable and unmodifiable** until you know which of them are clean.

## 3.1 Halt the backup jobs

A backup that runs now overwrites good data with encrypted data, or rotates a
clean set out of retention. Stop every one of them.

```
# Retrospect (Archive) machine — console:
#   Activities : stop any running script immediately
#   Scripts    : disable every scheduled script, do not merely pause one
#   Media sets : note which sets are currently members of a running job
#   Grooming/recycle policies: disable — grooming can remove exactly the old backups you need
```

```
# Proxmox, on each node — stop scheduled guest backups writing over good ones:
#   Datacenter -> Backup : disable every backup job
pvesm status                                     # which storages are active, and are any NAS-backed?
```

```
# MySQL hosts (data.example.com, db02.example.com, and any other) — suspend the nightly dump:
crontab -l                                       # confirm what is scheduled before changing it
crontab -l > ~/crontab-backup-$(date +%F).txt    # keep a copy so it can be restored exactly
# comment out the nightly dump line, e.g. /home/orgadmin/MySQLNightlyBackup.sh
```

**Preserve what is already on disk.** Copy the most recent known-good MySQL dumps
and the binary logs off the host to offline media before anything else touches
them — they are the difference between losing a day and losing an hour on
erpdb/data.

## 3.2 Break the network path to the Synology

Snapshots on files.example.com are the organization's fastest recovery path and the thing most
worth defending. They survive client-side encryption. They do **not** survive an
attacker with DSM admin credentials.

In order of preference:

1. **Disable SMB** (DSM → Control Panel → File Services → SMB). Stops every client
   instantly, leaves DSM, snapshots and logs available to you.
2. **Move the NAS's switch port to an isolated management VLAN** so only your
   admin workstation can reach it. `[FILL IN: quarantine/management VLAN ID and steps]`
3. **Pull the network cable** — effective, but you lose DSM management too.
4. **Power it off** — last resort only. You lose your own visibility, and an
   unclean shutdown risks the volume (`Hardware & Backup/synology-nas-administration-guide.md`).

Then, on the NAS itself:

```
# DSM -> Log Center            : admin logins, their source IPs, and any snapshot/share deletions
# DSM -> Control Panel -> User : any account created or privilege-changed recently
# DSM -> Snapshot Replication  : confirm the snapshot list still matches the retention policy
```

If a DSM admin login appears from an unexpected source, treat every DSM
credential as compromised and assume the attacker could have deleted snapshots —
check the count against what retention should be holding.

## 3.3 Make the snapshots immutable

This is the step that converts "we have snapshots" into "we will still have
snapshots tomorrow".

```
# DSM -> Snapshot Replication -> Snapshots -> select share -> select snapshot -> Lock
#   A locked snapshot cannot be deleted, by anyone, until it is unlocked or its retention lock expires.
#   Lock every snapshot that plausibly predates the intrusion — see Phase 4 for how far back that is.
# DSM -> Snapshot Replication -> [share] -> Settings -> Retention : suspend automatic deletion
#   while the incident is open, so retention does not quietly remove your restore points.
```

`[FILL IN: confirm snapshot locking / immutable snapshots are available on the DSM version in use on files.example.com — the NAS guide records both the DSM version and whether locking is enabled as unknown. If locking is unavailable, the only protection is that nobody with DSM admin reaches the NAS, which is why 3.2 comes first.]`

## 3.4 Secure the offline and offsite copies

- **The offsite Retrospect disk in the offsite safe is the strongest restore point
  the organization holds**, precisely because it has been air-gapped since it was last written.
  Leave it in the safe. Do not connect it to anything, and specifically do not
  connect it to the Retrospect machine, until the recovery environment is known
  clean and you know which restore point you want.
- Record which disk covers which dates before anyone handles them
  (`Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md` — media
  sets **Server Backups 2026** onsite and **Server Backups Offsite 2026** offsite).
- Treat the onsite media set as suspect until verified: it is network-reachable,
  which is the whole problem.
- `[FILL IN: is there any cloud or S3/B2 off-site copy, and does it have object
  lock / immutability enabled? Backup_DR_Runbook.txt §5 shows S3 and B2 sync as an
  example rather than a confirmed configuration — establish which it is.]`

## 3.5 Take the Retrospect machine off the shared network

The backup server is a primary target. If it is reachable from a compromised
host, it is at risk; if it has been compromised, everything it wrote recently is
suspect.

```
# Isolate the Retrospect (Archive) machine to the same management-only path as the NAS
# Then, on it:
#   check for unexpected local accounts, new admin accounts, and recent logins
#   check the Retrospect catalog files exist and are the expected size
#   check for a ransom note anywhere on its volumes
```

If the Retrospect machine itself is compromised, the offsite disks in the safe
become the only trusted path, and the whole recovery timeline changes. Establish
this early rather than discovering it during a restore.

## 3.6 Inventory what survived, before trusting any of it

Write this table down. It is the input to every decision in Phases 6 and 7.

| Restore point | System covered | Date/time | Reachable from a compromised host? | Verified clean? |
|---------------|----------------|-----------|-----------------------------------|-----------------|
| `[FILL IN]`   | `[FILL IN]`    | `[FILL IN]` | `[FILL IN]`                     | Not yet         |

Rules for filling it in:

- "Reachable from a compromised host" includes anything mounted by, backed up by,
  or authenticated to with credentials that existed on a compromised host.
- Nothing is marked verified clean until it has been through Phase 7's
  verification. **Untested restore points are rumours** — the same rule as
  `SysAdmin Procedures/Backup_DR_Runbook.txt` §3, and this is the day it matters.
- Include the Synology snapshots with their exact timestamps, the Retrospect
  onsite and offsite sets with their coverage dates, the Proxmox vzdump archives,
  the MySQL dumps and the binlog range they cover, and the AD System State
  backups.

---

# PHASE 4 — SCOPING

Scoping runs in parallel with protecting backups if two people are available, and
after it if only one is. The goal is four answers: where it started, when
encryption started, how far it spread, and which credentials are compromised.

## 4.1 Patient zero and the encryption start time

The encryption start time bounds the damage. The **intrusion** start time bounds
which restore points are safe, and it is always earlier.

```
# On an affected host — the oldest encrypted file usually marks local encryption start:
find /path/to/affected -name "*.<ransom-extension>" -type f -exec stat -f "%m %N" {} \; 2>/dev/null | sort -n | head -5   # macOS
find /path/to/affected -name "*.<ransom-extension>" -type f -printf "%T@ %p\n" 2>/dev/null | sort -n | head -5             # Linux
```

```
# On files.example.com, snapshots give you a bracket for free:
#   DSM -> Snapshot Replication -> browse the newest CLEAN snapshot and the oldest DIRTY one
#   Encryption of that share began between those two timestamps.
```

Then work backwards in Graylog from the encryption start time for the intrusion:
first appearance of the malicious binary, first anomalous authentication, first
lateral movement, first backup-job interference.
`[FILL IN: Graylog saved search for authentication and process events by host and account]`

**Record two timestamps, clearly labelled:**

- `Encryption started: [FILL IN]` — what the damage assessment uses.
- `Earliest known attacker activity: [FILL IN]` — what the restore point choice
  uses. When in doubt, push this earlier, not later.

## 4.2 Blast radius

Enumerate deliberately rather than by impression. Something that looks unaffected
because nobody has opened a file on it today is not unaffected.

| Area | What to check | Status |
|------|---------------|--------|
| macOS fleet | Munki check-in recency; AV/EDR detections; user reports; spot-check files on a sample | `[FILL IN]` |
| Windows 11 workstations | Defender status, event logs, user reports | `[FILL IN]` |
| Windows under Parallels | Both the guest **and** the Mac host — the guest can encrypt shared folders that reach the host and the NAS (`Windows & Mac Workstations/parallels-shared-folders-and-drive-mapping.md`) | `[FILL IN]` |
| files.example.com | Which shares, which directories, which snapshot is the last clean one per share | `[FILL IN]` |
| Dynamics GP / DMS (erpdb.example.com) | Payroll and member data — database files, backups, the application host. **Highest-priority determination in this table** | `[FILL IN]` |
| Zimbra (mail.example.com) | Mail store integrity; whether mail is still flowing | `[FILL IN]` |
| MySQL (data.example.com, db02.example.com) | Datadir, dump directory, binlogs; whether replication is still running and whether it replicated damage | `[FILL IN]` |
| Proxmox nodes and guests | Node filesystems, guest disks, and the vzdump target storage | `[FILL IN]` |
| Other application hosts | prod01, sign, app02, app03, forums | `[FILL IN]` |
| Domain controllers | auth2.example.com, auth4.example.com — AD database, SYSVOL, and whether either was logged into by the attacker | `[FILL IN]` |
| Network and security appliances | pfSense, Barracuda WAF, switches, wireless — config changes, new accounts, new rules | `[FILL IN]` |
| Backup infrastructure | Retrospect machine, NAS, backup storage | `[FILL IN]` |

Two traps:

- **Replication propagates damage.** MySQL replication from data.example.com to
  db02.example.com will faithfully replicate destructive statements. Check replication
  state and stop the replica if damage is replicating
  (`Databases/mysql-administration-guide.md`).
- **Synology Snapshot Replication to a second device**, if configured, replicates
  encrypted data to the replica too. The snapshots on both sides are still the
  defence; the live replica is not.
  `[FILL IN: is replication to a second Synology configured? The NAS guide records this as unknown]`

## 4.3 Assume domain admin is compromised

Until proven otherwise, treat these as in the attacker's hands:

- Every account that logged into any affected host, especially any privileged
  account used during the incident before step 6 of Phase 2 was applied.
- **Every Domain Admin account.** Ransomware that reaches file shares at scale
  usually has domain privilege; assuming otherwise is how organisations get
  re-encrypted mid-recovery.
- The `krbtgt` account — assume the attacker can forge Kerberos tickets, which
  means AD authentication cannot be trusted until it is reset (twice, with a
  replication cycle between — Phase 7).
- Service accounts (`svc-backup`, `svc-monitoring`, `svc-deploy`, the
  `servicesadmin` LDAP bind account) — these are the ones with reach.
- Local administrator accounts on every affected host.
- DSM admin on files.example.com, the Retrospect machine's accounts, pfSense and
  Barracuda admin, switch enable passwords.
- Anything stored in a browser, keychain, or credential manager on an affected
  host.

Proving otherwise means finding the credential in Graylog and showing it was not
used from an unexpected source in the relevant window — which is worth doing for
the few that would be expensive to rotate, and not worth doing for the rest.

Rotation itself happens in Phase 7, before systems reconnect, following the
rotation procedure in `Security & Hardening/secrets-management-guide.md`. Do not
start rotating now: rotating credentials while the attacker still has access
tells them you are responding and achieves nothing durable.

---

# PHASE 5 — EVIDENCE PRESERVATION (BEFORE ANY REBUILD)

## Why this comes before rebuilding

Rebuilding is the single most tempting thing to do, because it feels like
progress. It also destroys the only copy of the answer to how this happened.

Without evidence you cannot determine the initial access vector, which means you
cannot close it — and re-infection through the same door is the common ending to
a badly run ransomware recovery. You also cannot determine **whether data was
exfiltrated**, which is the question that drives the entire privacy and
notification obligation in Phase 8. "We do not know if member data left" is both
the worst answer to give a regulator and the answer you are guaranteed to give if
you wipe the hosts first.

Evidence also tells you which restore point is clean, by establishing when the
intrusion began. Wiping a host removes the timestamps you need to make that call,
and the fallback is to go much further back than necessary.

## What to capture, most volatile first

Keep at least one affected host **intact and powered on, isolated, untouched**,
as the reference specimen — ideally patient zero if it can be identified.
`[CONFIRM: proposed default — one representative host per platform is preserved
whole until the investigation closes]`

**1. Memory**, where the host is still powered on and the case warrants it
(it does — this is ransomware). `Security Procedures/IR_Security_Scenarios_Guide.md`
scenario 11 covers acquisition and analysis.

```
qm suspend <vmid> --todisk                        # Proxmox VM: freezes it AND writes memory to disk
#   The resulting state file is a memory image. Copy it off before doing anything else with that guest.
# macOS/Windows physical: osxpmem / WinPmem per scenario 11; if unavailable, capture live state below
```

**2. Live state**, before anything is killed or rebooted:

```
ps aux > ~/evidence/processes.txt                              # full process list with command lines
sudo lsof -i -n -P > ~/evidence/connections.txt                # who it is talking to right now
sudo lsof > ~/evidence/open-files.txt                          # which files the encryptor holds open
last | head -40 > ~/evidence/logins.txt                        # recent logins, including the attacker's
```

**3. Persistence and filesystem artefacts** — follow
`Security Procedures/modules/macos-host-triage.txt` phases 1–3 rather than
improvising; it covers LaunchAgents/Daemons, `sfltool dumpbtm`, login items, cron,
kernel and system extensions, and binary verification in the right order.

**4. The malicious artefacts themselves** — the encryptor binary, the ransom
note, a small sample of encrypted files, and one matching unencrypted original if
one exists anywhere (a snapshot, a backup, an email attachment). A plaintext/
ciphertext pair is occasionally what makes a strain identifiable or a decryptor
applicable.

```
shasum -a 256 <binary> <ransom-note> > ~/evidence/hashes.txt   # hash everything on collection
```

Do not upload files to public services. Hash lookups only — the triage toolkit's
rule (`Security Procedures/WORKFLOW.txt`, rules of engagement 5) applies with more
force here, because these files are the organization's own data.

**5. Logs, before they roll:**

```
log collect --last 7d --output ~/evidence/system-logs.logarchive   # macOS unified log archive
journalctl --since "14 days ago" > ~/evidence/journal.txt          # Linux
# Windows: export Security, System and Application event logs via Event Viewer -> Save All Events As
```

Export from Graylog as well, into a file held outside the environment — Graylog
itself is a target, and an attacker who reaches graylog01 can delete the timeline.
`[FILL IN: Graylog export procedure and where the export is stored]`

**6. Firewall, WAF and appliance logs** — pfSense/Snort and Barracuda WAF records
covering the intrusion window. These are frequently the only place outbound
exfiltration is visible, which makes them central to Phase 8.

**7. Disk images**, if the case warrants forensic depth.
`[FILL IN: is full-disk imaging in scope for the organization, and with what tooling? — the
same open question as endpoint-security-operations-guide.md]`

## Handling rules

- Copy evidence **off** each host to `[FILL IN: evidence storage location — must
  be offline or on isolated storage, never files.example.com and never a synced folder]`.
- Hash on collection; record who collected what, when, and from which machine.
- Retain for `[FILL IN: evidence retention period]` `[CONFIRM: proposed default —
  retain until the incident is formally closed and any legal, insurance or
  regulatory matter is concluded, whichever is later]`.
- Insurance and legal both routinely require evidence preservation. Destroying it
  before they are consulted can affect a claim.
  `[FILL IN: confirm with legal counsel and with the insurer what preservation is
  required and for how long]`
- **Stop before reimaging if member data may be involved.** Escalate first.
  `[FILL IN: who to notify when an incident may have legal or privacy implications]`

---

# PHASE 6 — THE RANSOM DECISION

## This is not IT's decision

It is not the sysadmin's decision, and it must not be made alone or under time
pressure created by the attacker's countdown. IT's job here is to supply facts —
what is encrypted, what is recoverable, from when, how long recovery will take,
and what evidence exists about exfiltration — and to implement whatever is
decided.

**Who must be involved before any decision is made, and before any contact with
the operator:**

| Party | Role in the decision | Contact |
|-------|---------------------|---------|
| Senior leadership / executive | The decision itself. Only they can weigh organisational impact and accept the consequences | `[FILL IN: named decision-maker and out-of-hours contact]` |
| Legal counsel | Legal exposure of paying and of not paying; notification obligations; privilege over the investigation | `[FILL IN: legal counsel name and contact]` |
| Cyber insurer / broker | Most policies require notification before any action, and many provide the incident response resources and the negotiation function. Acting first can void coverage | `[FILL IN: insurer, policy number, 24-hour claims line — confirm whether a cyber policy exists at all]` |
| Law enforcement | Reporting; may hold intelligence on the strain or the operator | `[FILL IN: law enforcement reporting contact and whether reporting is expected or required — confirm with legal counsel]` |
| Privacy / regulatory advice | Whether and when the privacy regulator and affected individuals must be notified — see Phase 8 | `[FILL IN: confirm with legal counsel]` |
| Specialist incident response firm | If retained, they normally run any communication with the operator. Individuals should not | `[FILL IN: retained IR firm, or the insurer's panel firm]` |

**Notify the insurer early**, before deciding anything and before engaging
anyone. `[FILL IN: confirm with the broker what the policy requires and in what
timeframe]`

## The considerations, stated evenhandedly

This section deliberately does not recommend a course of action.

**Arguments that are raised for paying:**

- It may be the fastest path back if no clean restore point exists for a critical
  system.
- Where data has been exfiltrated, payment is sometimes sought in exchange for a
  promise of deletion.
- Prolonged downtime has its own costs — operational, financial and reputational
  — and for some organisations those exceed the demand.

**Arguments that are raised against paying:**

- **Payment does not guarantee recovery.** Decryptors are frequently slow,
  partial, or buggy; some files do not come back; some organisations receive
  nothing. This is a documented and common outcome, not a rare one.
- A promise to delete exfiltrated data is unverifiable. The data has already been
  copied; there is no mechanism by which its deletion can be confirmed.
- Paying marks an organisation as one that pays, which is associated with repeat
  targeting.
- Payment funds the operation that did this, and the next one.
- **Legal exposure.** Payment may carry legal risk in your jurisdiction, including but not
  limited to sanctions considerations where the recipient cannot be identified.
  This document does not state what the law requires.
  `[FILL IN: confirm with legal counsel — the legality, the reporting
  obligations, and any sanctions screening required before a payment is
  considered]`
- Recovery work still has to happen either way: a decryptor does not remove the
  attacker's persistence, does not close the access vector, and does not make the
  hosts trustworthy. Rebuilding and credential rotation are required regardless.

**The fact that changes the conversation:** a **verified clean restore point**.
Its existence removes the pressure entirely — the decision becomes "how long will
restoring take", which is a planning question, not an extortion question. This is
the whole reason Phase 3 comes before everything else, and why the verification in
Phase 7 is not optional. Establish the restore position **before** the decision
meeting, and bring it to that meeting as a written statement of what can be
recovered, from when, and by when.

**If exfiltration is confirmed or suspected**, the availability question and the
disclosure question separate. Restoring perfectly from backup resolves the first
and does nothing about the second. The privacy obligations in Phase 8 apply
regardless of whether anything is paid, and regardless of how well the restore
goes.

## What IT does in the meantime

Do not wait for the decision. While it is being made:

- Continue the recovery track. Preparing a clean rebuild is not a commitment
  either way, and it is the work that makes "do not pay" viable.
- Do not contact the operator, do not visit their portal, and do not let anyone
  else do so from organization infrastructure. Set that expectation explicitly with
  anyone who has seen the note.
- Preserve the ransom note and any identifiers in it as evidence.
- Identify the strain from the note and a sample file where possible — a public
  decryptor exists for some families, and this is worth ten minutes:
  `https://www.nomoreransom.org` and `https://id-ransomware.malwarehunterteam.com`
  (hash and note text only; do not upload organization data files).

---

# PHASE 7 — RECOVERY SEQUENCE

Do not start recovery until: the spread has stopped, the backups are protected,
evidence is captured, and you know the earliest attacker activity time. Starting
early is how organisations get encrypted a second time, mid-recovery, with their
restore points already spent.

## 7.1 Rebuild from clean, or restore in place?

| | Rebuild from clean | Restore in place |
|---|---|---|
| What it means | New OS install on wiped storage, then restore data only | Restore files onto the existing installation |
| Removes attacker persistence | Yes | **No** |
| Speed | Slower | Faster |
| When it is acceptable | Always | Only where the host is provably unaffected **and** no attacker credential ever touched it |

**The rule: rebuild, do not clean.** Any host that ran the ransomware, or that an
attacker authenticated to, is rebuilt from a known-good image and has data
restored onto it. Cleaning an infected host leaves you unable to say whether the
persistence is gone, and you will be asked. This is the same conclusion
`Security Procedures/modules/macos-host-triage.txt` phase 5 step 6 reaches for a
single host, applied across an estate.

Restore-in-place is defensible for a host that was isolated before it was touched
and where the evidence supports that — but the burden of proof sits with
restoring, not with rebuilding.

For guests, rebuilding means restoring the vzdump archive of a guest from **before
the intrusion**, not repairing the current one.

## 7.2 Build a clean room first

Recovery happens on an isolated network segment, not the production one.
`[FILL IN: which VLAN/segment is used as the recovery clean room, and how it is
isolated from the production network]`

Prerequisites before the first system comes up:

- Known-clean installation media and images, verified by hash.
- A clean administrative workstation, built fresh, never used on the compromised
  network with privileged credentials.
- New credentials generated in advance (Phase 7.4) — not the old ones.
- The verified restore point identified (Phase 7.3) and physically or logically
  available.
- Patching material for whatever the initial access vector turns out to be. If the
  vector is not yet known, that is a reason to look harder, not a reason to skip
  this.

## 7.3 Verify a restore point is actually clean

This is the step that is skipped and should not be. A restore point is clean only
if it predates the **intrusion**, not the encryption.

1. **Choose by the earliest-known-attacker-activity time, not the encryption
   time.** Select a restore point at least `[CONFIRM: proposed default — 7 days]`
   before the earliest confirmed attacker activity, and be prepared to go further
   back if the dwell time is unclear. Losing a week of data is recoverable;
   restoring the attacker's persistence is not.
2. **Restore it into the isolated clean room, never onto production.**
3. **Check for the known indicators** from Phase 5 — the encryptor's hashes,
   filenames, persistence paths, and any accounts the attacker created.

```
# Persistence sweep on a restored macOS system, before it is trusted:
sudo sfltool dumpbtm                                         # authoritative background-item list
ls -la /Library/LaunchDaemons /Library/LaunchAgents ~/Library/LaunchAgents   # compare against a known-good build
/Applications/KnockKnock.app/Contents/MacOS/KnockKnock -json | jq '.[] | select(.signature.status != "Apple")'   # all non-Apple persistence

# Restored Linux/LXC:
systemctl list-unit-files --state=enabled                    # unexpected enabled units
crontab -l; ls -la /etc/cron.d/                              # scheduled persistence
grep -rn "" /root/.ssh/authorized_keys /home/*/.ssh/authorized_keys 2>/dev/null   # added keys

# Restored Windows: scheduled tasks, services, Run keys, and local account list
```

4. **Scan the restored data** with current definitions before it leaves the clean
   room.
5. **Confirm the data is intact, not just present.** Open real files. Query real
   tables. A restored database that starts is not the same as a restored database
   with the right row counts:

```
mysql -e "SELECT COUNT(*) FROM <table>;"                     # row counts against a known figure
zcat /path/to/dump.sql.gz | head -30                         # dump is readable and is the database you think
pg_restore --list /path/to/dump > /dev/null && echo "dump OK"
```

6. **Record the verification** — what was checked, by whom, with what result.
   That record is what allows the restore point to be trusted in the meeting where
   the ransom question is asked.

If a restore point fails verification, go back further. Do not "clean" a restore
point.

## 7.4 Rotate every credential before anything reconnects

Rotation happens **after** the attacker's access is cut and **before** rebuilt
systems rejoin the network. Rotating too early warns them; rotating too late
hands them the rebuilt estate.

Follow `Security & Hardening/secrets-management-guide.md` — its rotation procedure
(find every consumer first) applies unchanged, at estate scale.

The set, at minimum:

- **Every Domain Admin and privileged AD account.**
- **`krbtgt`, reset twice**, with at least one full AD replication cycle between
  the two resets. One reset does not invalidate forged tickets; two do. Resetting
  twice in quick succession breaks Kerberos across the domain — wait for
  replication between auth2.example.com and auth4.example.com to complete.
- **DSRM passwords** on both domain controllers.
- **Every user account** `[CONFIRM: proposed default — force a password reset for
  all staff accounts, and force MFA re-registration]`.
- **Service accounts** — `svc-backup`, `svc-monitoring`, `svc-deploy`, the
  `servicesadmin` LDAP bind account, MySQL application and admin accounts, the
  MySQL replication account. Each has a consumer list that must be updated in the
  same window or authentication breaks.
- **Local administrator accounts** on every host, rebuilt or not.
- **Appliance and network credentials** — pfSense, Barracuda WAF, switch enable,
  wireless controller, DSM admin on files.example.com, the Retrospect machine,
  Graylog admin, MDM and AV/EDR consoles.
- **API keys and tokens**, including DNS provider, off-site backup, and anything
  a compromised host held.
- **SSH keys** — remove old public keys from every `authorized_keys` and issue new
  keypairs.
- **Certificate private keys**, if a key could have been read from a compromised
  host — reissue rather than reuse (`Security & Hardening/certificate-pki-lifecycle-guide.md`).
- **Break-glass accounts**, and re-seal them.

## 7.5 The order systems come back

The dependency order is the same one `Hardware & Backup/power-outage-shutdown-runbook.md`
uses for bring-up, with verification gates that are about trust rather than
hardware. **Do not start a tier until the tier below it is verified.**

| # | Tier | Systems | Gate before the next tier |
|---|------|---------|---------------------------|
| 0 | Network and perimeter | pfSense, switches, quarantine/clean-room segmentation | Segmentation enforced; the recovery segment cannot reach anything unrebuilt; rules reviewed for attacker changes |
| 1 | **Identity and DNS** | auth2.example.com, auth4.example.com; internal DNS | DCs rebuilt or restored clean; `krbtgt` reset twice; replication healthy; DNS resolving; a test domain login works |
| 2 | Storage | files.example.com — volumes, shares, permissions, snapshot schedules | Volumes Healthy; shares reachable from a clean test client; snapshot schedules running again; locked snapshots retained |
| 3 | Hypervisors | Proxmox nodes | Nodes up, cluster quorate, all storages active, clocks correct |
| 4 | **Databases** | erpdb.example.com, data.example.com, db02.example.com | Engines started cleanly; startup logs reviewed for crash recovery; row counts verified; replication rebuilt from a verified source, never resumed blindly |
| 5 | Core applications | Dynamics GP / DMS, Zimbra (mail.example.com) | Application connects to its database; a real transaction and a real mail send/receive both work |
| 6 | Remaining applications and web | prod01, sign, app02, app03, data, db02, forums | Sites serve over HTTPS with valid certificates; a real user journey works |
| 7 | Endpoints | macOS fleet, Windows 11 workstations, Parallels guests | Each machine rebuilt or verified clean, re-enrolled in Munki/MDM, new credentials, before it touches the restored shares |

Why this order:

- **AD and DNS first** because everything else authenticates and resolves through
  them, and because a domain that still trusts forged tickets contaminates
  everything you bring up after it.
- **Storage before databases** because databases and applications mount it.
- **Databases before applications** because an application started against a
  missing database produces retry loops, confusing errors, and a second restart
  later.
- **Endpoints last** because an endpoint that was not properly rebuilt, connecting
  to a freshly restored share, restarts the incident.

Additional rules during bring-up:

- **Bring systems up onto the isolated segment first**, verify, then move to
  production. Not the other way round.
- **Monitor intensively** as each tier comes up. Re-encryption during recovery is
  a known pattern. Graylog, the AV/EDR console, and file-modification rates on
  files.example.com all get watched actively, not passively.
- **Restore data, not system state**, wherever possible. The point of rebuilding
  is to not carry the old installation forward.
- **Do not reconnect the offsite Retrospect disk** until the recovery environment
  is verified clean. It is the last copy.
- Take a fresh backup of each restored system once it is verified and before it
  takes production load, so the recovery itself has a rollback point.

---

# PHASE 8 — NOTIFICATION AND REGULATORY OBLIGATIONS

**Every specific in this section must be confirmed with legal counsel before it is
acted on. This document establishes the shape of the obligation, not its content.**

## The applicable frame

The organization holds personal information about members and
employees. The relevant privacy legislation is **`[FILL IN: applicable privacy
statute]`**, overseen by `[FILL IN: privacy regulator]`.
`[FILL IN: confirm with legal counsel that this is the applicable statute for
the organization's holdings, and whether any other statute or obligation also applies — for
example in respect of employee records or any health-related information the organization
holds]`

## What triggers an obligation

A ransomware incident can trigger privacy obligations **whether or not data was
exfiltrated**: unauthorised access alone may be sufficient, and loss of access to
personal information can itself be a form of incident.

- `[FILL IN: confirm with legal counsel — what threshold triggers a reporting
  obligation to the privacy regulator, and how it is assessed]`
- `[FILL IN: confirm with legal counsel — the timeframe within which a report must
  be made]`
- `[FILL IN: confirm with legal counsel — when affected individuals must be
  notified, by whom, and what the notification must contain]`
- `[FILL IN: confirm with legal counsel — any record-keeping obligation for
  incidents that do not meet the reporting threshold]`
- `[FILL IN: confirm with legal counsel — obligations arising from payroll data
  specifically, and any obligation owed to employers, benefit providers or other
  third parties whose data the organization holds]`

## What IT must establish to support that assessment

The assessment is legal, but it cannot be made without facts only IT can provide.
Produce these as a written statement:

1. **What categories of personal information were involved.** Member records,
   employment and payroll data (Dynamics GP / DMS), mail contents, file shares.
2. **How many individuals are potentially affected**, and whether they are
   members, employees, or both.
3. **Whether data was exfiltrated, or whether it cannot be excluded.** Firewall,
   Snort, WAF and proxy logs covering the intrusion window are the primary
   evidence. Say plainly which it is — "no evidence of exfiltration" and "we can
   confirm no exfiltration" are very different statements, and only the first is
   usually true.
4. **The window of unauthorised access** — first attacker activity to containment.
5. **What has been done** to contain, recover and prevent recurrence.

## Internal and external communication

| Audience | When | Who sends it | Notes |
|----------|------|--------------|-------|
| Senior leadership | Immediately on declaration | IT | Phone, not email |
| Legal counsel | Immediately | `[FILL IN: who engages counsel]` | Engage before external communication of any kind |
| Cyber insurer | Immediately, before decisions or engagements | `[FILL IN]` | Policy may require it before any action |
| Staff | Once containment is underway | `[FILL IN: who communicates to staff]` | Say what is down, what not to do (do not switch machines on, do not reconnect drives, do not open the note), and when the next update comes |
| Members | Only per legal advice | `[FILL IN: who communicates to members]` | Never ahead of counsel |
| Regulator | Per legal advice | `[FILL IN: confirm with legal counsel]` | |
| Law enforcement | Per legal advice | `[FILL IN: confirm with legal counsel]` | |
| Media | Only via `[FILL IN: designated spokesperson]` | | Nobody else comments, including on personal accounts |

**Out-of-band communication is a prerequisite.** If Zimbra is down or untrusted,
internal email does not work, and neither does anything that authenticates against
AD. `[FILL IN: the out-of-band method — an SMS list, a phone tree, personal mobile
numbers held offline, or an external status page. This is the same gap recorded in
Hardware & Backup/power-outage-shutdown-runbook.md and it needs to be closed once,
for both documents.]`

Assume the attacker may be reading organization mail. Conduct incident coordination out of
band until identity and mail are rebuilt and credentials rotated.

---

# PHASE 9 — POST-INCIDENT

## Root cause

The incident is not closed until the initial access vector is named and closed.
"Probably phishing" is not a root cause. Establish, from the evidence collected in
Phase 5: how access was obtained, how privilege was escalated, how lateral
movement happened, how backups were reached, and what detection existed that
should have fired earlier but did not.

If the vector genuinely cannot be determined, record that explicitly — it changes
the risk posture and it justifies broader hardening rather than a targeted fix.

## The honest gap review

Do it while it is fresh, with the goal of fixing things rather than allocating
blame. The questions worth asking:

- Which of the early signals in Phase 1 were present in the data but not noticed,
  and what would have had to exist for them to be noticed?
- How long was the dwell time, and what would have caught it sooner?
- Were the backups adequate? Which restore point was actually used, how far back
  did it put us, and how long did recovery take against the RTO/RPO targets in
  `SysAdmin Procedures/Backup_DR_Runbook.txt`?
- Were snapshots locked, or did we get lucky?
- Did the offsite copy matter? Would it have been enough on its own?
- What in this playbook was wrong, missing, or unusable under pressure?
- Which `[FILL IN]` markers in this document cost real time on the day? Those are
  the highest-priority follow-ups in the entire library.

## What to change

Candidate actions, to be selected on evidence rather than adopted wholesale:

- Enable and enforce **snapshot locking** on files.example.com, and confirm retention
  covers a realistic dwell time.
- Establish a genuinely **offline or immutable** backup copy for every critical
  system, not only for those covered by the offsite disks.
- Reduce the blast radius of a single compromised credential — tiered
  administration, separate admin accounts, no daily-driver domain admin.
- Close the **Windows workstation telemetry gap** recorded in
  `Security & Hardening/endpoint-security-operations-guide.md`; without it, scoping
  is guesswork.
- Build the Graylog alerts and saved searches Phase 1 depends on, and test that
  they fire (`Security Procedures/modules/detection-validation.md`).
- Turn the incident's indicators into detections
  (`Security Procedures/modules/detection-engineering.txt`).
- **Exercise a restore.** Not a file restore — a full system restore, timed.
- Re-run this playbook as a tabletop `[CONFIRM: proposed default — annually, and
  after any material infrastructure change]`.

Write the post-incident record into Decisions & History below, and into
`SysAdmin Procedures/Change_Management_Log.txt`.

---

## Operations (Day-2)

Preparedness work that makes this playbook usable. A playbook that is first read
during an incident is a document, not a capability.

### Monthly

- [ ] Verify Retrospect scripts ran and the media sets are as expected.
- [ ] Verify snapshots exist on every snapshotted share, at the expected count.
- [ ] Verify the offsite disk rotation actually happened and the safe holds what
      the record says it holds.
- [ ] Confirm the printed copy of the first-hour card is where it should be, and
      is current.

### Quarterly

- [ ] Test a restore from each layer — a Synology snapshot, a Retrospect onsite
      restore, a Proxmox guest, a MySQL dump plus binlogs — and record the date
      and result. `[CONFIRM: proposed default]`
- [ ] Confirm break-glass credentials work and are sealed
      (`Security & Hardening/secrets-management-guide.md`).
- [ ] Verify the out-of-band contact list is current.
- [ ] Review who holds domain admin, and whether they still need it.

### Annually

- [ ] Tabletop this playbook end to end with everyone who might have to run it.
      Time the decision points.
- [ ] Full system restore test for at least one critical system, to a clean
      environment. `[CONFIRM: proposed default]`
- [ ] Review cyber insurance coverage and the notification requirements in the
      policy. `[FILL IN: confirm whether a cyber policy exists, its coverage, and
      its notification requirements]`
- [ ] Confirm with legal counsel that the Phase 8 obligations are still correctly
      stated.

### After any infrastructure change

- [ ] Does the new system have a backup, and is that backup reachable from a
      compromised host? If yes, it is not a backup for this scenario.
- [ ] Add it to the Phase 4 blast radius table and the Phase 7 bring-up order.

## Troubleshooting

### Symptom: files are renamed but still open normally

- Likely cause: a hoax, a test, a failed encryptor, or a mass-rename script.
- Check: `file <path>` reports the original type, and the contents read normally.
- Fix: do not stand down the response until you have checked several files across
  several shares. A partial encryptor renames everything and encrypts some.

### Symptom: encryption continues after the host is isolated

- Likely cause: a second infected host you have not found, or the encryptor is
  running on the NAS-facing client you did not identify.
- Check: which client sessions are open on files.example.com (DSM → Resource Monitor →
  Connections); which hosts show mass SMB activity in Graylog.
- Fix: disable SMB on the NAS (Phase 2 step 4) rather than continuing to hunt
  host by host. Find the client afterwards.

### Symptom: the restore is encrypted too

- Likely cause: the restore point postdates the intrusion, or the restore target
  is still infected, or the backup itself was encrypted before it was written.
- Check: the restore point's timestamp against the earliest-known-attacker-activity
  time; whether the restore target was rebuilt or merely cleaned.
- Fix: go further back, restore into the clean room, and rebuild the target. Do
  not retry onto the same host.

### Symptom: Retrospect will not restore the date you need

- Likely cause: the date is not in the recent list and needs retrieving from the
  media set catalog, or the required disk is the offsite one in the offsite safe.
- Check: `Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md`
  step 4 — "More Backups…" → Retrieve.
- Fix: remember the **+1 day rule** — backups run at 2:00 AM, so files from the
  8th are in the 9th's or later backup. And check **Restore to a new folder**;
  skipping it overwrites the destination.

### Symptom: snapshots are missing or fewer than retention should hold

- Likely cause: someone with DSM admin deleted them — the attacker.
- Check: DSM → Log Center for deletion events and the admin logins around them.
- Fix: treat DSM admin as compromised, isolate the NAS to management-only access,
  lock everything that remains, and move the recovery plan onto Retrospect and the
  offsite set. Record the deletion as evidence — it is material to both the
  investigation and any insurance claim.

### Symptom: after rebuilding AD, clients cannot authenticate

- Likely cause: the two `krbtgt` resets were done without a replication cycle
  between them, or DNS is pointing at a host that is not yet up.
- Check: replication health between auth2.example.com and auth4.example.com; DNS resolution
  from a client; clock skew (`timedatectl` / `w32tm /query /status`) — Kerberos
  tolerates only minutes of drift.
- Fix: allow replication to complete, correct clocks, then retest. Do not reset
  `krbtgt` a third time to "fix" it.

### Symptom: recovery is underway and new encryption appears

- This is the worst case and it is a known pattern. Stop bringing systems up
  immediately.
- Check: which system was compromised, and whether it was rebuilt or restored in
  place; whether credentials were rotated before it reconnected.
- Fix: re-isolate, go back to Phase 2, and treat every system brought up since the
  last verified-clean point as suspect. The usual cause is a host that was cleaned
  rather than rebuilt, or an unrotated credential.

## Security

- **This playbook contains no credentials and must never contain any.** It tells
  you where they live, not what they are.
- Assume during an incident that the attacker can read anything on the organization
  infrastructure, including mail, tickets and file shares. Coordinate out of band.
- Break-glass credentials are the assumed access path when AD or the vault is
  unavailable. They must be sealed, offline, and tested —
  `Security & Hardening/secrets-management-guide.md`.
- The highest-value hardening against this scenario, in rough order: immutable or
  offline backups; restricted and tiered domain admin; endpoint telemetry that is
  actually watched; and segmentation that limits what one compromised credential
  can reach.
- The Munki repo, the MDM console and the AV/EDR console can each execute code on
  every endpoint. In a ransomware incident they are both a recovery tool and a
  distribution mechanism — verify their integrity before using them to rebuild.
- `[FILL IN: confirm whether files.example.com, and any management interface, is
  reachable from the internet. An internet-exposed NAS or hypervisor is the single
  most common initial access vector for this scenario.]`

## Monitoring & Alerting

Detections that would shorten this incident, and which should exist before it
happens:

- **Mass file modification rate** on files.example.com above baseline — the single most
  direct ransomware signal available.
- **Snapshot deletion** or snapshot count dropping on the NAS.
- **Backup job failure** on Retrospect, Proxmox, or the MySQL dump — alert on
  absence, not only on error.
- **Backup job volume anomaly** — a sudden large increase in data backed up.
- **Break-glass and service account authentication** from unexpected sources.
- **AV/EDR ransomware-behaviour detections** and **agent disabled/tampered**, both
  of which page out of hours per
  `Security & Hardening/endpoint-security-operations-guide.md`.
- **Shadow copy / backup catalogue deletion** events on Windows hosts.
- **Snort alerts on internal-to-internal SMB** and lateral movement tooling.
- Volume of outbound data from internal hosts, which is the exfiltration signal.

`[FILL IN: Graylog streams, alert conditions and destinations for each of the
above — monitoring-alerting-guide.md records the stream list as unknown, and every
detection here depends on it]`

`[FILL IN: out-of-hours paging method — an alert nobody sees at 2am is not a
control]`

Baseline first: `[FILL IN: normal daily file-modification volume on files.example.com,
normal Retrospect backup size per media set, normal snapshot space consumption.
Without these, "anomalous" is not actionable.]`

## Disaster Recovery

- **RTO/RPO in a ransomware event are not the RTO/RPO in
  `SysAdmin Procedures/Backup_DR_Runbook.txt`.** Those targets (AD 4h, databases
  2h, file servers 8h) assume a single system failed and the rest of the estate is
  trustworthy. Here, every system is rebuilt, credentials are rotated
  estate-wide, and the restore point is days older than the failure.
  `[CONFIRM: proposed default — plan on days to weeks for full recovery, and set
  leadership's expectations accordingly on day one]`
- Recovery priority order, highest first
  `[CONFIRM: proposed default — confirm with leadership, because this is a business
  decision, not a technical one]`:
  1. Identity and DNS (auth2, auth4) — nothing else works without them.
  2. Dynamics GP / DMS (erpdb.example.com) — payroll obligations have dates that do
     not move.
  3. files.example.com — most staff cannot work without it.
  4. Zimbra (mail.example.com) — communication, internal and external.
  5. Everything else.
- **Minimum viable service**: `[FILL IN: what does the organization need running to function at
  a basic level — payroll, member services, mail? Agree this before an incident,
  because agreeing it during one wastes the hours that matter most.]`
- Escalation if the owner is unavailable: `[FILL IN: second person with the
  knowledge and access to run this playbook]`. A single point of failure here is
  the same finding recorded in `Security & Hardening/secrets-management-guide.md`,
  and this scenario is where it hurts most.
- External incident response capability: `[FILL IN: retained IR firm or the
  insurer's panel — establish this in advance; finding one mid-incident costs a
  day]`

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why / Ticket |
|------------|-------------------|--------------|
| 2026-09-11 | Created this playbook alongside the existing IR scenarios guide | The IR guide covers detection and single-host response; nothing covered recovery decision-making while systems are encrypted — backup protection, restore point selection, the ransom decision, and the rebuild order |
| 2026-09-11 | Default containment stance recorded as **isolate, not power off**, with a written decision rule for the exceptions | The choice is irreversible in one direction and was previously undocumented |
| 2026-09-11 | Backup protection placed ahead of scoping and evidence in the response order | Modern ransomware destroys backups before encrypting; the window to protect them is measured in minutes |
| `[FILL IN]` | `[FILL IN: record incidents, tabletop exercises, restore tests and playbook corrections here]` | `[FILL IN]` |

## References

- `Security Procedures/IR_Security_Scenarios_Guide.md` — scenario 6 (first commands on a host), 1, 2, 5, 10, 11
- `Security Procedures/modules/macos-host-triage.txt` — five-phase host triage; isolate-or-observe decision; persistence sweep
- `Security Procedures/START-HERE.txt`, `Security Procedures/WORKFLOW.txt` — email/document triage; step 4 identifies the affected host, step 14 the escalation triggers
- `Security Procedures/modules/detection-engineering.txt`, `Security Procedures/modules/detection-validation.md` — turning this incident's indicators into detections that fire
- `Security Procedures/modules/file-recovery.txt` — recovering a deleted file still held open by a process, during live response
- `Security & Hardening/endpoint-security-operations-guide.md` — endpoint isolation, evidence preservation, alert severity, exclusions
- `Security & Hardening/secrets-management-guide.md` — rotation procedure, break-glass accounts, exposure response
- `Security & Hardening/certificate-pki-lifecycle-guide.md` — certificate key reissue after compromise
- `Hardware & Backup/synology-nas-administration-guide.md` — snapshots, locking, shares, degraded arrays, DSM administration
- `Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md` — media sets, the +1 day rule, "Restore to a new folder"
- `Hardware & Backup/power-outage-shutdown-runbook.md` — bring-up order and gates, which this playbook's recovery order follows
- `SysAdmin Procedures/Backup_DR_Runbook.txt` — RTO/RPO targets, AD System State restore, database restore, verification
- `SysAdmin Procedures/monitoring-alerting-guide.md` — Graylog streams, alerts and index sets
- `SysAdmin Procedures/Change_Management_Log.txt` — record the recovery changes here
- `Databases/mysql-administration-guide.md` — dumps, binary logs, point-in-time recovery, replication
- `Linux & Servers/proxmox-cluster-administration-guide.md`, `Linux & Servers/lxc_backup_restore_proxmox91.txt` — guest backup and restore
- `Active Directory/AD-Admin-Security-Guide.md` — AD attack paths, privileged account hygiene
- `Dynamics GP & DMS/dynamics-gp-2018-admin-guide.md` — the payroll/member system whose data drives the privacy obligation
- `Mail & Messaging/zimbra-mail-administration-guide.md` — mail recovery
- `Windows & Mac Workstations/parallels-shared-folders-and-drive-mapping.md` — why a Windows guest can encrypt files on the Mac host and the NAS
- External: `https://www.nomoreransom.org` (free decryptors, strain identification), `https://id-ransomware.malwarehunterteam.com` (strain identification from a note — do not upload organization data files)

---

# FIRST-HOUR CARD — PRINT AND KEEP OFFLINE

```
================================================================
  RANSOMWARE — FIRST HOUR              ORG IT — rev 2026-09-11
  Full playbook: Security Procedures/ransomware-response-playbook.md
  Call: the sysadmin [FILL IN: phone]
        Escalation:    [FILL IN: name / phone]
        Legal:         [FILL IN]
        Insurer:       [FILL IN: 24h claims line]
================================================================

CONFIRM IT IS RANSOMWARE
  [ ] Ransom note present in affected directories
  [ ] File CONTENTS unreadable, not just renamed  (file <path> = "data")
  [ ] Mass file modification with no user activity behind it

FIRST 15 MINUTES — IN THIS ORDER
  1. [ ] Note the time. Start a written log on a CLEAN machine.
  2. [ ] Declare it by PHONE to [FILL IN: IT lead]. Do not wait for scope.
  3. [ ] ISOLATE affected hosts — network OFF, POWER ON.
  4. [ ] Share being encrypted and can't cut the client in ~2 min?
         DISABLE SMB ON THE NAS instead. (DSM -> File Services -> SMB)
         Do NOT power the NAS off.
  5. [ ] HALT ALL BACKUP JOBS — Retrospect scripts, Proxmox backup jobs,
         MySQL nightly dump. Disable grooming/recycle.
  6. [ ] NO privileged logins on any suspect host. Assume DA is stolen.
  7. [ ] Do NOT reboot. Do NOT restore yet. Do NOT run a cleaner.
  8. [ ] Do NOT touch the ransom note's portal or contact address.

ISOLATE vs POWER OFF
  DEFAULT = ISOLATE. Power off destroys memory: keys, config, C2, creds.
  Staying online lets encryption run and lets the operator react.
  Proxmox VM?  qm suspend <vmid> --todisk   (stops it AND saves memory)
  DC or erpdb? Get a second person on the phone before isolating.
  Power off ONLY for fire/water/thermal, or when nothing else can stop
  active destruction. Write down why and when.

PROTECT THE BACKUPS  (first priority after isolation)
  [ ] Backup jobs halted (above)
  [ ] NAS network path broken — SMB disabled or port to mgmt VLAN
  [ ] Snapshots LOCKED; retention auto-delete suspended
  [ ] DSM Log Center checked for snapshot deletions + admin logins
  [ ] Offsite Retrospect disk STAYS IN THE OFFSITE SAFE — do not connect it
  [ ] Retrospect machine isolated and checked for compromise
  [ ] Copy surviving MySQL dumps + binlogs to offline media
  [ ] Write down every restore point and its date

SCOPE  (assume domain admin is compromised)
  [ ] Encryption start time ______   Earliest attacker activity ______
  [ ] Which hosts, which shares, erpdb/GP status, mail status
  [ ] Check MySQL replication + Synology replication — they spread damage

PRESERVE BEFORE ANY REBUILD
  Rebuilding first destroys how-it-happened AND whether data left.
  [ ] Memory (qm suspend --todisk on VMs), ps/lsof/last, logs, note,
      sample encrypted file, the binary. Hash everything. Store OFFLINE.
  [ ] Keep one affected host intact and untouched.

RANSOM DECISION — NOT IT'S TO MAKE
  Leadership + legal counsel + insurer (notify BEFORE acting) +
  law enforcement per advice. Paying does not guarantee recovery and
  may carry legal exposure [confirm with counsel]. A VERIFIED CLEAN
  RESTORE POINT is what removes the pressure — establish it first.
  Nobody contacts the operator.

RECOVERY ORDER  (gate at each tier; rebuild, do not clean)
  0. Network / segmentation
  1. AD + DNS (auth2, auth4) — krbtgt reset TWICE, replication between
  2. Storage (files.example.com)
  3. Hypervisors (Proxmox)
  4. Databases (erpdb, data, db02)
  5. Core apps (GP/DMS, Zimbra)
  6. Remaining apps and web
  7. Endpoints — rebuilt, re-enrolled, new credentials
  Restore point must predate the INTRUSION, not the encryption.
  Rotate ALL credentials before anything reconnects.
  Verify restores in an isolated clean room first.

PRIVACY — erpdb holds payroll and member data
  [ ] Tell legal counsel EARLY. [applicable privacy statute] is the frame.
  [ ] Determine: what data, how many people, exfiltration yes/no/unknown
  [ ] All timing and notification specifics: confirm with counsel.

DO NOT
  - Do not reboot or power off by reflex
  - Do not restore onto an infected host
  - Do not reconnect the offsite disk until the clean room is verified
  - Do not rotate credentials while the attacker still has access
  - Do not email about the incident if mail may be compromised
================================================================
```

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

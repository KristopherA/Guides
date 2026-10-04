> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Backup Restore Testing Procedure

## Overview

This procedure covers **proving that the organization's backups can actually be
restored**. It is not a restore procedure — those already exist, one per
system, and they are listed in References below. This document is about
the separate discipline of deliberately restoring something nobody has
lost, on a schedule, to a place where it cannot do any harm, and writing
down what happened and how long it took.

The reason it exists is a gap found during the 2026-09-11 library review.
The library documents, in reasonable detail, **how to back up** (Retrospect
media sets, Proxmox `vzdump`, `mysqldump` plus binary logs, Synology
Snapshot Replication) and **how to restore when something has broken**
(`Retrospect Restore Procedure.md`, `lxc_backup_restore_proxmox91.txt`, the
recovery section of the MySQL guide). What it does not contain anywhere is
**evidence that a restore has been performed and verified**. There is no
restore-test log, no per-system "last tested" date, and no record of how
long any restore actually took.

That is the classic failure mode. Backup software reports success for
years while writing something unusable — an excluded directory nobody
noticed, a database dumped without its `mysql` schema, a media set whose
catalog has drifted, an offsite disk that has not been rotated. Every one
of those looks identical to a healthy backup until the day it is needed,
and the day it is needed is the worst possible day to find out.

Two related honesty notes, because they change how this document should be
read:

- **`SysAdmin Procedures/Backup_DR_Runbook.txt` is unfilled boilerplate.**
  Its section 3.3 is headed "TEST RESTORE (monthly — schedule this)" and
  its section 1 contains an RPO/RTO table. Neither describes the organization. The
  hosts are `web01`, `app01`, `fileserver`; the backup server is
  `10.0.1.50`; the domain is `example.com`. **The RPO and RTO
  figures in that table are template defaults, not Example Org commitments**, and
  nothing in the library shows them being agreed with anyone. Do not quote
  them to management and do not plan against them. Establishing real
  figures is a business decision — see
  `SysAdmin Procedures/business-continuity-plan.md`.
- **One restore is known to have been exercised.** `Retrospect Restore/
  Retrospect Restore Procedure.md` carries "Last reviewed: 2026-07-09
  (procedure tested successfully on this date)", and the transcript in
  `Retrospect Restore/Retrospect restore.txt` is that session: a restore
  of the `data.example.com` MySQL backup folders from the onsite media set to
  the Test drive. That is a real, verified file-level restore of one system
  and it is the only one in the library. It is the baseline this procedure
  builds on, not a substitute for the rest of it.

## Quick Facts

| Field            | Value                                                       |
|------------------|-------------------------------------------------------------|
| Owner            | IT lead (it@example.com)                          |
| Environment      | prod (tests are run **against** production backups, **into** an isolated destination) |
| Location         | Retrospect console on the Retrospect (Archive) machine; Proxmox web UI / PVE node shell; DSM at files.example.com; database hosts over SSH |
| Access           | Retrospect console; Proxmox root; MySQL admin; DSM admin; [FILL IN: Zimbra admin and Dynamics GP/DMS admin access — who holds each] |
| Dependencies     | Retrospect (Archive) machine and its media sets; the Test drive used as restore destination; Proxmox backup storage; Synology snapshots; spare capacity to restore into; `Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md` |
| Dependents       | Every RTO and RPO claim Example Org makes. The business continuity plan depends on the numbers this procedure produces |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                     |

## How It Works

### Why restore testing is separate from having a restore procedure

A restore procedure answers *"what buttons do I press?"*. A restore test
answers *"does the data on the media still turn back into a working
system, and how long does that take?"*. They fail independently, and a
correct procedure over a bad backup produces a confident, well-documented
failure.

Concretely, a documented restore procedure does **not** tell you:

| Question | Only a test answers it |
|---|---|
| Is the backup readable at all? | Media degrades, catalogs drift, archives truncate. `tar: Error is not recoverable` (see `lxc_backup_restore_proxmox91.txt`) is discovered on restore, never on backup |
| Is the *right data* in it? | An exclude rule, a renamed directory, a database added last year and never added to the dump script. The job still reports success |
| Does the restored thing **run**? | A database that imports without error can still be missing its stored routines, its `mysql` schema, or its replication metadata |
| Can *this person* do it? | A procedure only one person has ever executed is a single point of failure with a filename |
| **How long does it take?** | This is the number that converts an assumed RTO into a real one. Nobody can estimate it. It has to be measured |
| Does the offsite copy work? | The onsite set gets exercised by accident. The offsite set — "Server Backups Offsite 2026", whose disks live in the offsite safe — gets exercised only deliberately |

There is also a human reason. A restore performed for the first time
during an outage is performed under pressure, by someone who has not done
it before, on a day when getting it wrong makes things worse. The
transcript in `Retrospect Restore/Retrospect restore.txt` contains the
best statement of this in the whole library, from the person being shown
the procedure: they wanted to drive it themselves, because watching
someone else succeed does not build the muscle. That is the entire
argument for scheduled testing.

### The three test depths

Each depth proves something the one below it cannot. Running the cheap one
often is good practice; running *only* the cheap one produces false
confidence, because it exercises the media and nothing else.

| Depth | What you do | What it proves | What it does **not** prove |
|---|---|---|---|
| **1. File-level restore** | Pull specific files or folders out of the backup to an isolated destination and compare them against the source | The media is readable, the catalog resolves, the restore path works, and the operator can drive the tool | Nothing about whether the *system* those files belong to can be rebuilt. A perfect file restore of a database datadir may still be an unusable database |
| **2. Application-level restore** | Restore the application's data into a *running but isolated* instance of that application and use it — log in, run a query, open a record, send a test message | The data is internally consistent and the application accepts it. Catches missing schemas, missing stored routines, broken permissions, encoding damage, missing config | Nothing about rebuilding the host, the OS, the packages, the certificates, or the network identity |
| **3. Full bare-metal / VM rebuild** | Build the host from nothing — new VM or CT, OS, packages, config — and restore into it until the service works | The **whole** recovery path, including the steps nobody wrote down: the certificate that lives outside the backup, the DNS record, the firewall rule, the ODBC DSN, the AD join. This is the only depth that measures a true RTO | Nothing about the rest of the estate. A rebuilt app server still assumes its database exists |

The failures that actually hurt cluster at depths 2 and 3. A file-level
test would not have caught, for example, a MySQL dump taken without the
`mysql` schema — the dump file restores cleanly and the replica it came
from can never resume replication (see the "Backing up a replica" note in
`Databases/mysql-administration-guide.md`).

**Rule of thumb:** every system gets depth 1 regularly. Every
business-critical system gets depth 2 at least annually. Depth 3 is done
at least once for each *class* of system — one LXC container, one VM, one
database host, one Mac — so that the rebuild path is known, and then
repeated when the class changes materially.

---

# TEST SCHEDULE

## System tiers

Tiering exists so that test effort matches consequence. The organization is a member-based
organization: the consequence that matters is not lost revenue, it is a missed
arbitration deadline, a grievance file that cannot be produced, a payroll
run that does not happen, or member data that cannot be relied on during
negotiations.

| Tier | Definition | Systems |
|---|---|---|
| **Tier 1 — critical** | Loss stops payroll, member services, or a legally time-bound process (grievances, arbitration deadlines) | Dynamics GP; DMS / dms-db.example.com; files.example.com; AD (auth2.example.com, auth4.example.com); Zimbra (mail.example.com) |
| **Tier 2 — important** | Loss disrupts staff work substantially but has a workaround for days, not hours | data.example.com, db02.example.com (MySQL pair); prod01.example.com; sign.example.com; the Proxmox platform itself |
| **Tier 3 — supporting** | Loss is an inconvenience or affects a limited group | forums.example.com; app01.example.com; app02.example.com; graylog01.example.com; staff workstations |

[FILL IN: confirm this tiering with Example Org leadership. IT can propose it;
only the business can say whether, for example, forums.example.com being down
for a week is acceptable. The business impact analysis in
`SysAdmin Procedures/business-continuity-plan.md` is where that
conversation is recorded.]

## Proposed cadence

| Tier | File-level (depth 1) | Application-level (depth 2) | Full rebuild (depth 3) |
|---|---|---|---|
| Tier 1 | Monthly `[CONFIRM: proposed default]` | Semi-annually `[CONFIRM: proposed default]` | Annually `[CONFIRM: proposed default]` |
| Tier 2 | Quarterly `[CONFIRM: proposed default]` | Annually `[CONFIRM: proposed default]` | Once, then after any material platform change `[CONFIRM: proposed default]` |
| Tier 3 | Semi-annually `[CONFIRM: proposed default]` | Opportunistically — when the system is being rebuilt or migrated anyway `[CONFIRM: proposed default]` | Not scheduled |

Additional triggers that are not calendar-driven, and which matter more
than the calendar does:

- **After any change to a backup job, schedule, media set, or retention
  rule.** A changed job is an untested job.
- **After any major version upgrade** of the backed-up system (DSM, MySQL,
  Proxmox, Zimbra) or of Retrospect itself.
- **After adding a new host or a new database.** The commonest real-world
  data loss at a small site is a system that was never added to the backup
  at all. A depth-1 test is the cheapest way to find out.
- **After any restore performed in anger.** Record it in the log below —
  a real restore is a test that the universe scheduled for you, and its
  measured duration is the most trustworthy number you will ever get.
- **Offsite media specifically:** `[CONFIRM: proposed default — test a
  restore from the "Server Backups Offsite 2026" media set at least
  annually.]` The offsite set is the one that will be needed after a fire
  or a ransomware event, and it is the one that is never exercised by
  accident. It requires retrieving a disk from the offsite safe, which is
  itself a step worth rehearsing.

## Test matrix

This is the record of what has been tested. **Every cell in the results
columns is empty because no restore test has been recorded anywhere in the
library** — with the single exception noted in the Overview, the Retrospect
file-level restore of 2026-07-09.

Fill a row in each time a test is run, and keep the detail in the
restore-test log further down.

| System | Backup method | Last tested | Result | Tested by | Next due |
|---|---|---|---|---|---|
| Dynamics GP | [FILL IN: what backs GP up today — MS SQL backup job, Retrospect, or both?] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| DMS / dms-db.example.com | [FILL IN: database backup method and whether the DMS web tier is backed up separately] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Zimbra — mail.example.com | [FILL IN: Zimbra's own backup, filesystem/Retrospect, or both] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| AD — auth2.example.com | [FILL IN: system state backup method actually in use] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| AD — auth4.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| files.example.com (Synology) | Snapshot Replication (on-volume) + Retrospect (offsite/long-term) | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| data.example.com (MySQL primary) | Nightly `mysqldump` + binary logs + Retrospect | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| db02.example.com (MySQL replica) | [FILL IN: is the replica dumped too, and does the dump include the `mysql` schema?] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| prod01.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| sign.example.com | [FILL IN: Docker volumes + database; how are they captured?] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| forums.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| app01.example.com | `mysqldump` via cron + Retrospect | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| app02.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| graylog01.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Proxmox LXC guests | `vzdump` to [FILL IN: storage target] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Proxmox VM guests | `vzdump` to [FILL IN: storage target] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Proxmox host config (`/etc/pve`) | [FILL IN: is the cluster config itself backed up? This is what you need to rebuild a node] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Synology DSM configuration | DSM Configuration Backup — [FILL IN: is it taken, and where is it stored off the NAS?] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| macOS workstations | [FILL IN: Time Machine, Retrospect client, MDM re-image, or nothing?] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Windows 11 workstations | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Retrospect catalog files | [FILL IN: are the Retrospect catalogs themselves backed up? Losing a catalog means rebuilding it from media, which is slow] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |

Two rows in that table are easy to overlook and are worth stating plainly:

- **The Proxmox cluster configuration and the DSM configuration are not
  data, they are the ability to rebuild.** A restore of guest archives
  onto a node you cannot reconstruct is a long afternoon.
- **The Retrospect catalogs** are how Retrospect knows what is on the
  media. Media without a catalog is recoverable but slowly.

---

# THE TEST PROCEDURE

## Before you start

- [ ] **Schedule it.** A restore test is a change: it consumes storage, it
      consumes I/O on the backup system, and it can collide with the 2:00 AM
      Retrospect window. Log it in
      `SysAdmin Procedures/Change_Management_Log.txt`.
- [ ] **Pick the restore point deliberately.** Do not always take last
      night's. Testing a backup from [CONFIRM: proposed default — at least
      once a year, restore from a point at the far end of the retention
      period] proves the retention policy is real and that older media
      still reads.
- [ ] **Confirm free space at the destination** before starting, not
      halfway through. A test that fills a production volume has turned
      itself into an incident.
- [ ] **Know what "correct" looks like before you restore.** Decide the
      verification in advance — a row count, a specific file's checksum, a
      named record, a login that must succeed. Verifying after the fact
      invites grading your own homework.
- [ ] **Have the source available to compare against**, where the source
      still exists.

## The step that matters most: restore to an ISOLATED destination

**Never restore into production to test a restore.** The point of the
exercise is to gain confidence, not to risk the live copy of the thing you
are trying to protect. Every system below has a documented isolated
destination; use it.

| System | Isolated destination | The specific danger if you skip this |
|---|---|---|
| Retrospect | The **Test** drive on the Retrospect (Archive) machine, with **"Restore to a new folder" checked** | Without that checkbox Retrospect **overwrites the destination drive**. This is called out twice in `Retrospect Restore Procedure.md` and again in the session transcript. It is the single most dangerous step in this document |
| Proxmox LXC/VM | Restore to a **new, unused VMID**, network disconnected | `pct restore <same vmid> --force` destroys the running container. A restored guest that boots with the production IP and hostname causes an outage and, for a domain member, can disrupt AD |
| MySQL | A scratch instance or a differently-named database on a non-production host | Restoring over the live datadir is unrecoverable. Replaying binary logs into a production server applies real writes |
| Synology | Snapshot **Browse** (read-only), copying files out | Snapshot **Restore** rolls the *entire share* back for every user, not just the file you wanted |
| Zimbra | A scratch account, or a separate test server | Restoring a mailbox over a live one loses everything the user received since the backup |
| Dynamics GP / DMS | A restored copy of the database under a different name, pointed at by a test company/instance | Overwriting the payroll database is the worst outcome available in this environment |

Isolation means three things, and all three are needed:

1. **A different name or ID** than the production object.
2. **No network identity collision** — different IP, different hostname,
   and for anything domain-joined, do not let it register in AD.
3. **No shared writable storage** with production. A test guest that
   mounts the live NAS share read-write can modify live data.

```
# Proxmox: restore to a spare VMID, and detach the network BEFORE starting
pct restore 9101 /var/lib/vz/dump/<archive>.tar.zst --storage [FILL IN: storage]   # new VMID, not the original
pct set 9101 --net0 name=eth0,bridge=[FILL IN: isolated bridge],link_down=1        # bring it up with no network
pct start 9101                                                                      # now it is safe to boot
```

## Generic test procedure

Steps 1-9 apply to every system. The per-system notes that follow fill in
the specifics.

**1 — Record the start.** Wall-clock time, to the minute. Everything about
the timing measurement depends on this being an honest number.

**2 — Locate the restore point.** Note the media set, archive filename,
snapshot timestamp, or dump file you are restoring from — and note whether
finding it was itself slow. Retrieving a backup that is not in
Retrospect's recent list requires a **More Backups… → Retrieve** job that
can take several minutes before the restore even begins
(`Retrospect Restore Procedure.md`, step 4). That time counts.

**3 — Verify integrity before restoring, where the tool allows it.**

```
zcat /path/to/dump.sql.gz | head -40            # readable, and is the header what you expect?
sha256sum -c /path/to/backup.sha256             # if checksums are recorded alongside backups
zstd -t /var/lib/vz/dump/<archive>.tar.zst      # archive integrity without extracting
```

[FILL IN: are checksums recorded alongside backups today? If not, adding
them is cheap and makes this step meaningful.]

**4 — Restore to the isolated destination.** Per the table above. Do not
improvise the destination.

**5 — Record the restore-complete time.** This is the raw restore
duration. Keep it separate from the verification time that follows — they
answer different questions.

**6 — Verify at the depth you are testing.**

- Depth 1: compare against the source.

```
diff -r /restore/path/ /source/path/            # exact tree comparison where the source still exists
find /restore/path -type f | wc -l              # file count against the source's count
sha256sum /restore/path/<known-file>            # checksum of a file whose value you know
```

- Depth 2: start the application against the restored data and use it.
  Log in. Open a record. Run a report. Send a message.
- Depth 3: the service answers correctly from another host, over its
  normal protocol, with its normal certificate.

**7 — Record the service-usable time.** From the start of step 1 to the
moment the restored thing is actually usable. **This is the number that
becomes the RTO evidence.**

**8 — Write the log entry.** Use the template below. Write it now, while
the detail is fresh, including everything that surprised you.

**9 — Clean up deliberately.**

```
pct stop 9101 && pct destroy 9101               # remove the test container
mysql -e "DROP DATABASE restore_test;"          # remove the scratch database
# Retrospect: delete the restored files from the Test drive once verified
```

Cleanup is part of the procedure, not an afterthought. A forgotten test
restore on the Test drive fills it before the next test; a forgotten test
VM consumes storage and, worse, may eventually be mistaken for something
real.

---

# PER-SYSTEM TEST NOTES

## Retrospect

Authoritative procedure: `Hardware & Backup/Retrospect Restore/Retrospect
Restore Procedure.md`. Do not restate it — follow it. What matters for
*testing* specifically:

- **Backups run daily at 2:00 AM, so a given day's files land in the next
  day's backup.** To test recovery of a file as it existed on the 8th,
  restore from the 10th or 11th. Getting this wrong makes a healthy
  backup look empty.
- **Test the onsite set routinely; test the offsite set deliberately.**
  Onsite is "Server Backups 2026"; offsite is "Server Backups Offsite
  2026" and may prompt for a disk from the offsite safe. The offsite test is
  the one that matters after a fire or ransomware, and retrieving the disk
  is part of what you are rehearsing. Record how long the retrieval took.
- **Destination: the Test drive, with "Restore to a new folder" checked.**
  Skipping that checkbox overwrites the destination drive.
- **Deliver nothing during a test.** The "Restore for <user>"
  folder on the Systems Admin share is for real restores. A test ends at
  verification and cleanup.
- **Test a restore of an *older* backup, not just the most recent.** The
  Retrieve path (step 4 of the procedure) is a different code path from
  the recent-list path and is the one that will be needed under pressure.
- [FILL IN: which hosts have a working Retrospect client today? A host
  that silently dropped out of the backup set is invisible until someone
  looks. Cross-check the client list against the host inventory in
  `SysAdmin Procedures/Inventory_Asset_Reference.txt` once that file has
  been filled in.]
- [FILL IN: Retrospect server version and the client versions deployed.
  The MySQL guide records a pinned 2018-era Linux client; confirm what is
  actually installed.]

## Proxmox LXC and VM

Authoritative procedure: `Linux & Servers/lxc_backup_restore_proxmox91.txt`
and `Linux & Servers/proxmox-cluster-administration-guide.md`.

Proxmox is the easiest platform here to test properly, because restoring
to a new VMID is non-destructive by design and is already documented as
the safe side-by-side option.

```
pvesm list [FILL IN: backup storage id]         # what archives exist, and how old is the newest?
zstd -t /var/lib/vz/dump/<archive>.tar.zst      # archive integrity check before restoring
pct restore 9101 <archive> --storage [FILL IN]  # restore to a SPARE VMID — never the original
pct set 9101 --net0 name=eth0,bridge=[FILL IN: isolated bridge],link_down=1   # no network identity clash
pct start 9101
pct exec 9101 -- systemctl is-system-running    # 'running' or 'degraded' — read the degraded list
pct exec 9101 -- df -h                          # are all the expected mounts there?
pct stop 9101 && pct destroy 9101               # clean up
```

Test-specific points:

- **`qm restore` for VMs is the same shape** — restore to a spare VMID,
  and check the restored VM's network config before starting it.
- **Verify inside the guest, not just that it booted.** A container that
  starts and a container whose application works are different claims.
  Check the service, the data directory, and the config files the
  application actually reads.
- **Bind mounts and NAS-backed storage are the classic gap.** `vzdump`
  does not back up bind-mounted host directories by default. If a guest's
  real data lives on a mount point, the archive may contain an empty
  directory where the data should be. [FILL IN: which guests use bind
  mounts or NAS-backed storage, and is that data covered by another
  backup?] This is worth checking before the next test, not during it.
- **Restore a guest whose disks live on the NAS** at least once. It
  exercises a different storage path and a different failure mode.
- Note the archive's compression and size against the restore duration —
  that ratio is what lets you estimate a restore you have not run.

## MySQL

Authoritative procedure: the Backups and Recovery sections of
`Databases/mysql-administration-guide.md`. That guide already carries the
line **"Last tested restore: [FILL IN: date]. An untested restore is a
rumour."** — this procedure is how that blank gets filled.

A MySQL restore test that stops at "the dump imported without errors" is
the most common form of false confidence in this environment. The dump
importing proves the file is syntactically valid SQL. It proves nothing
about completeness, about point-in-time recovery, or about replication.

Test it in four parts:

**1. The dump restores and the data is complete.**

```
mysql -e "CREATE DATABASE restore_test;"                       # scratch target, never the live schema
mysql restore_test < /path/to/<db>dump_YYYY-MM-DD.sql          # import the nightly dump
mysql restore_test -e "SELECT COUNT(*) FROM <known_table>;"    # compare against production's count
mysql restore_test -e "SHOW TABLES;" | wc -l                   # table count matches production?
mysql restore_test -e "SELECT COUNT(*) FROM information_schema.routines WHERE routine_schema='restore_test';"
```

That last check catches a real and quiet failure: a dump taken without
`--routines` restores perfectly and silently lacks every stored procedure
and function. The documented nightly dump does use `--routines`;
confirming it on the restored copy is how you know the *running* job still
matches the documented one.

**2. Point-in-time recovery actually works.** This is the part that is
never tested and the part that justifies the whole binary log
infrastructure. Restoring last night's dump and stopping there means
accepting up to 24 hours of data loss — which is only acceptable if that
was a deliberate decision, and it has not been made.

```
head -40 /path/to/<db>dump_YYYY-MM-DD.sql | grep -i "CHANGE.*SOURCE\|CHANGE MASTER"
#   --source-data=2 writes the binlog file and position into the dump as a comment.
#   That comment is where the replay must start. If it is absent, PITR is not possible from this dump.

mysqlbinlog --start-position=<pos from above> --stop-datetime="<a chosen point>" \
  /path/to/binlogs/mysql-bin.0000NN > /tmp/replay.sql    # ALWAYS write to a file and read it first
mysql restore_test < /tmp/replay.sql                      # then apply to the SCRATCH database

mysql restore_test -e "SELECT COUNT(*) FROM <table>;"     # more rows than the dump alone had?
```

What this test must prove, and what a dump-only test cannot:

- The binary logs are **being copied off the database host at all**. The
  MySQL guide flags this explicitly: `[FILL IN: where binary logs are
  actually copied to today, and whether the copy is running.]` If the
  copy is not running, point-in-time recovery is imaginary and the real
  RPO is 24 hours, not minutes.
- The copied logs are **on different storage from the datadir**. Binary
  logs on the disk that died are not a recovery capability.
- The logs are **continuous** — no gap between the dump's recorded
  position and the first copied log, and no gap between log files.
- The replay applies cleanly and produces the expected row counts.

**3. Replication can be re-established after a restore.** A restored
primary that no replica can follow, or a restored replica that cannot
resume, is a half-recovery.

```
mysql -e "SHOW REPLICA STATUS\G" | egrep "Replica_IO_Running:|Replica_SQL_Running:|Seconds_Behind_Source:|Last_IO_Error:|Last_SQL_Error:"
mysql -e "SELECT COUNT(*) FROM mysql.slave_master_info;"   # replication metadata present in the restored copy?
```

The specific trap, straight from the MySQL guide: **MySQL 8 keeps
replication metadata in InnoDB tables in the `mysql` schema**, not in
`master.info`. A dump of application databases only does not contain it,
and a replica restored from such a dump cannot resume. Verify the dump
includes the `mysql` schema, or that there is a documented re-point
procedure that does not need it.

Also remember, per that guide: `Seconds_Behind_Source: 0` alone is not
proof of being caught up. Check `Relay_Log_Space` has stopped growing and
`Replica_SQL_Running_State` is idle.

**4. The consumers work.** For data.example.com / db02.example.com and especially
dms-db.example.com, the database existing is not the same as Dynamics GP or DMS
working against it. That is a depth-2 test and needs the ODBC path
exercised — see below.

[FILL IN: which host is available to restore into for MySQL testing? A
dedicated scratch instance — a small LXC container with MySQL 8 — would
make this routine instead of awkward, and is cheap on the existing
Proxmox platform.]

## Synology snapshots (files.example.com)

Authoritative procedure:
`Hardware & Backup/synology-nas-administration-guide.md`.

Snapshots are the fast recovery path and the one users notice. They are
also the one most likely to be quietly misconfigured, because a share with
snapshots disabled looks exactly like a share with snapshots enabled until
somebody needs one.

Test monthly, at depth 1:

```
# DSM: Snapshot Replication -> Snapshots -> [share] -> confirm last snapshot timestamp is recent
# DSM: Snapshot Replication -> Snapshots -> [share] -> select a snapshot -> BROWSE (never "Restore")
#      Copy one known file out, and compare it against the live copy.
# Windows client: right-click a file on the mapped drive -> Properties -> Previous Versions
#      This is the self-service path. If it is empty, "Make snapshot visible" is off for that share.
```

What the test must cover:

- **Every user-facing share, in rotation** — not just the convenient one.
  Snapshot settings are per shared folder. [FILL IN: list of shares with
  snapshots enabled, from DSM; a share missing from that list is a gap.]
- **The user's own route works**, not just the admin's. Previous Versions
  from a Windows client is the route that saves the most support time;
  test it from a client, not from DSM. Mac users have no Previous Versions
  — their route is the admin one, and that is a known limitation to
  confirm rather than rediscover.
- **Retention matches what is documented.** Restore from the *oldest*
  snapshot that retention claims to hold. If the policy says 30 days and
  the oldest available is 9, the policy and reality have diverged.
- **Use Browse, never Restore.** "Restore" rolls the whole share back for
  every user of it.

**Snapshots are not a backup and a snapshot test is not a backup test.**
They live on the same volume as the data. A snapshot test proves fast
user-error recovery works; it says nothing about surviving the loss of the
NAS. That case is a Retrospect test, and it is the one that needs
[FILL IN: is there a documented bare-metal NAS recovery path? The NAS
guide flags this as its biggest gap] resolved before it can be run
end-to-end.

## Zimbra mailstore (mail.example.com)

Authoritative reference: `Mail & Messaging/zimbra-mail-administration-guide.md`.

Mail is tier 1 because grievance and arbitration correspondence lives in
it, and because those deadlines are external and unforgiving.

[FILL IN: what backs Zimbra up today — Zimbra's own backup facility
(network edition), a scripted `zmmailbox`/`zmmboxmove` export, an LDAP +
mailstore filesystem backup via Retrospect, or a VM-level `vzdump` of the
whole guest? The test procedure depends entirely on this answer, and
nothing in the library records it.]

Whatever the mechanism, a Zimbra restore test must cover three separate
things, because they are backed up separately and fail separately:

| Component | Why it is separate | Test |
|---|---|---|
| **Mailbox content** | The messages themselves | Restore one mailbox to a scratch account and confirm message count, folder structure, and that a message body opens |
| **LDAP / account directory** | Accounts, aliases, distribution lists, COS. Restoring mail into an account that no longer exists is not a restore | Confirm the restored account, its aliases, and its group memberships exist |
| **Configuration** | Domains, DKIM keys, filters, shared mailbox delegation | Confirm a shared mailbox's delegation survives — `Mail & Messaging/Share Mailboxes Zimbra` covers the live configuration |

Test at depth 1 on the cadence above, and at depth 2 at least annually:
restore a mailbox into a **scratch account** (never over a live one), open
it in the web client, and read a message.

```
# On the Zimbra host, as the zimbra user:
su - zimbra
zmcontrol status                                     # all services running before you touch anything
zmmailbox -z -m [FILL IN: scratch account] getAllFolders   # folder structure of the restored mailbox
zmprov ga [FILL IN: scratch account] | head -30      # account exists with expected attributes
```

The specific danger: **restoring a mailbox over a live account** discards
everything received since the backup. Always restore to a scratch account
and move messages afterwards if a real recovery is needed.

Also confirm DKIM keys are part of whatever is backed up. A restored mail
server that fails DKIM starts failing DMARC at recipients — see
`Mail & Messaging/email-authentication-spf-dkim-dmarc.md`.

## Dynamics GP and DMS (dms-db.example.com)

These are the most business-critical systems Example Org runs — payroll and member
data — and they are the hardest to test, because a meaningful test needs a
second application environment, not just a second database.

[FILL IN: does a test or training company/instance of Dynamics GP exist,
and is there a non-production DMS web tier? Without one, depth-2 testing of
GP/DMS is not possible and that limitation should be recorded and
escalated rather than quietly tolerated.]

Minimum viable test until that exists:

- Depth 1, on the cadence above: restore the database backup to a scratch
  name on a non-production instance and confirm it mounts, plus row counts
  on [FILL IN: a stable, well-known table in the DMS/GP schema] against
  production.
- Confirm the **ODBC path** is part of the recovery plan, not an
  afterthought. The MySQL guide notes the connector version dependency
  explicitly: a 5.x connector cannot authenticate to a MySQL 8 server
  using default authentication, and the error does not say so. A restored
  database that the application cannot connect to is not a restored
  service.
- Confirm what else GP and DMS need that is *not* in the database:
  [FILL IN: report definitions, custom modifications, integration files,
  scheduled jobs, shares under `Dynamics GP & DMS/Shares for Dynamics`].
  These are exactly the things a database-only backup misses and a
  database-only test never reveals.

## Active Directory (auth2.example.com, auth4.example.com)

AD is the dependency under almost everything else — file shares, mail,
workstation login, the NAS domain join. It also has the most dangerous
restore in the estate, because a careless AD restore can reintroduce
deleted objects or cause replication divergence across the domain.

[FILL IN: how are the domain controllers backed up today? `Backup_DR_Runbook.txt`
describes `wbadmin` system state backups to a share, but that is template
text and its target is `10.0.1.50\ad-backups`, which is not an Example Org address.
Establish what actually runs.]

Testing rules specific to AD:

- **Never test an AD restore against the live domain.** A DSRM system
  state restore must be done on an **isolated network**, with no path to
  the production DCs. This is not a preference; a test restore that
  replicates back into production is an incident.
- Having two DCs (auth2 and auth4) means the realistic failure is losing
  one, and the realistic recovery is rebuilding it and letting replication
  repopulate it — not restoring from backup at all. **That path should be
  tested too**, and it is much safer: [CONFIRM: proposed default — test
  rebuilding a secondary DC from scratch and letting it replicate, rather
  than testing a system state restore, unless a forest-recovery scenario
  is specifically being rehearsed.]
- The scenario that *does* need a backup is losing both, or a logical
  corruption that replicated. That is a forest recovery, it is a
  significantly harder exercise, and it needs planning time.

---

# MEASURING AND RECORDING RESTORE TIME

This is the part that turns documentation into a commitment you can
defend. **Example Org currently has no measured restore time for anything.** The
figures in `Backup_DR_Runbook.txt` ("2-4 hours", "15 minutes to several
hours", "2-6 hours") are the template's illustrative estimates and were
never measured here.

Measure five intervals, separately. The split matters, because they are
improved by completely different actions:

| Interval | From → To | Why it is separate |
|---|---|---|
| **T1 — Locate** | Decision to restore → restore point identified and available | Includes Retrospect's Retrieve job, fetching a disk from the offsite safe, finding the right dump. Often the biggest surprise, and it is fixed by better indexing, not faster hardware |
| **T2 — Transfer** | Restore started → data fully written to the destination | The only interval people estimate, and usually the smallest. Scales with data volume |
| **T3 — Reconstruct** | Data present → application running against it | OS, packages, config, certificates, DSNs, domain join. For a depth-3 test this dominates everything else |
| **T4 — Verify** | Application running → confirmed correct | Checking row counts, opening records, testing a login |
| **T5 — Total to usable** | Decision to restore → service usable by staff | **This is the number to compare against a target RTO.** It is T1+T2+T3+T4 plus whatever went wrong |

Practical rules for making the number honest:

- **Record wall-clock timestamps, not durations.** Write down the actual
  time at each transition. Durations computed afterwards are guesses.
- **Include the waiting.** Time spent hunting for a password, waiting for
  a disk from the safe, or reading documentation counts. In a real
  recovery you will spend it too.
- **Do not subtract the mistakes.** If a step had to be redone, the time
  stays in. The first real recovery will have mistakes too.
- **Record the data volume alongside the time**, so future estimates can
  scale. "40 GB in 50 minutes" is reusable; "about an hour" is not.
- **A test run by the person who knows the system best is a best case.**
  Note who ran it. [CONFIRM: proposed default — at least annually, have a
  test run by someone who is *not* the system's owner, following only the
  written procedure. That measures the recovery Example Org would actually get if
  the owner were unavailable, and it is the only way to find out that the
  procedure has an undocumented step in it.]

Once T5 exists for a system, it becomes evidence in the business impact
analysis in `SysAdmin Procedures/business-continuity-plan.md`. Until then,
every RTO in this library is an aspiration.

---

# RESTORE TEST LOG

Append an entry per test. Keep the log in this file so that the evidence
and the procedure live together — a log kept somewhere else is a log
nobody finds.

Record failures in full. A failed test that is documented is the most
valuable entry in the log; a failed test that is quietly re-run until it
passes is worse than no test at all.

## Template

```
--------------------------------------------------------------------
RESTORE TEST RECORD
Date:                 YYYY-MM-DD
Tested by:
System:
Tier:                 1 / 2 / 3
Depth:                1 file-level / 2 application-level / 3 full rebuild
Backup source:        media set / storage / dump file
Restore point:        date and time of the backup being restored
Why this point:       most recent / oldest in retention / offsite set / other

Destination (ISOLATED):
Isolation confirmed:  [ ] different name or ID
                      [ ] no network identity collision
                      [ ] no shared writable storage with production

TIMINGS (wall clock)
  Decision / start                    __:__
  Restore point located and available __:__      T1 = ____
  Restore started                     __:__
  Restore complete                    __:__      T2 = ____
  Application running                 __:__      T3 = ____
  Verification complete               __:__      T4 = ____
  TOTAL TO USABLE                                T5 = ____
  Data volume restored:               ____ GB

VERIFICATION PERFORMED (what specifically was checked)
  -
  -

RESULT:   [ ] PASS   [ ] PASS WITH FINDINGS   [ ] FAIL

FINDINGS / SURPRISES (anything not in the written procedure)
  -

ACTIONS RAISED (with owner and due date)
  -

DOCUMENTATION UPDATED?   [ ] yes — which file(s):
                         [ ] no update needed
CLEANUP COMPLETED?       [ ] restored data removed  [ ] test guest destroyed
NEXT TEST DUE:           YYYY-MM-DD
--------------------------------------------------------------------
```

## Log entries

```
--------------------------------------------------------------------
Date:        2026-07-09
Tested by:   [FILL IN: who drove the restore — the transcript names two
             participants by chat handle only]
System:      data.example.com — MySQL nightly backup folders
Tier:        2
Depth:       1 (file-level)
Backup source: Retrospect, "Server Backups 2026" (onsite media set)
Restore point: 2026-05-11 backup (containing the 8th and 9th folders)
Destination: Test drive on the Retrospect (Archive) machine,
             "Restore to a new folder" checked
Timings:     [FILL IN: not recorded at the time. The transcript spans
             roughly 11:56 to 12:13, but that includes narration and
             teaching; it is not a measured restore time]
Result:      PASS — files restored and verified visually
Findings:    The target date was not in the recent list and had to be
             retrieved via More Backups… -> Retrieve, which ran as a
             separate job first. The +1 day rule (2:00 AM backups mean a
             day's files land in the next day's backup) caused an initial
             wrong selection.
Actions:     Machine renamed from `data` to `data.example.com` in Retrospect
             for clarity.
Documented:  Yes — `Retrospect Restore/Retrospect Restore Procedure.md`
Next due:    [FILL IN]
--------------------------------------------------------------------
```

[FILL IN: every subsequent entry. As of 2026-09-11 the entry above is the
only recorded restore test in the library.]

---

# WHEN A TEST FAILS

## A failed restore test is an incident

Treat it as one. Not a maintenance task, not a note for the next review —
an incident, because the finding is that **Example Org has been unable to recover
a system for an unknown length of time and did not know it.** The data
loss has not happened yet, which is the only good news, and which is
exactly the window in which it is cheap to fix.

Handle it under `Security Procedures/IR_Security_Scenarios_Guide.md`'s
general incident discipline: contain, understand, fix, record. The
specific steps:

**1. Do not delete the evidence.** Keep the failed restore, the logs, and
the error text. Do not re-run the test to "see if it works this time"
before capturing what failed. The second run may succeed for an irrelevant
reason and the finding will be lost.

**2. Establish the blast radius immediately.** The most important question
is not "why did this one fail" but "what else is affected":

| The failure | The question to answer today |
|---|---|
| One archive is unreadable | Is it one archive, or is the media/storage failing? Test a second and third from the same set |
| A dump is missing data | Since when? Check older dumps. The job may have been silently broken for months |
| A host is not in the backup at all | What else is not in it? Enumerate every host against the backup client list |
| The restore procedure is wrong | Every other system using the same tool is affected |
| Point-in-time recovery does not work | The real RPO is the dump interval, right now, for every database. Say so |

**3. Determine how long it has been broken.** This is the number the
business needs. "The nightly dump has not included the `mysql` schema
since the upgrade in March" is a different conversation from "last night's
job failed".

**4. Restore backup coverage before anything else.** If a system is
currently unprotected, an immediate manual backup — to whatever medium
works — comes before root-cause analysis. Protect first, understand
second.

**5. Notify.** [CONFIRM: proposed default — a failed tier 1 restore test
is reported to Example Org management the same working day, with plain language
about which business functions are currently unrecoverable and for how
long they have been. A tier 2 or 3 failure is reported in the next regular
update unless it indicates a systemic problem.] Backup failures are the
kind of risk that leadership is entitled to know about while it is still
theoretical.

**6. Fix, then re-test.** The fix is not complete when the configuration
changes. It is complete when a restore test passes. Log both attempts.

**7. Record it.** Both the failure and the fix go in the log above and in
Decisions & History below. A library that records only successful tests is
back to being a set of assumptions.

## Failures worth anticipating

These are the ones most likely to show up in this environment, based on
what the library already documents:

| Failure | Likely cause | Where to look |
|---|---|---|
| Archive will not extract | Media or storage corruption; truncated archive from a job that ran out of space | `lxc_backup_restore_proxmox91.txt` failure blocks |
| Backup exists but is empty or tiny | Source path wrong, exclude rule too broad, the job ran while the source was unavailable (e.g. an encrypted home directory not mounted — see the eCryptfs note in the MySQL guide) | The backup job's own log |
| Database imports but the app fails | Missing routines, missing `mysql` schema, character set damage (`utf8` vs `utf8mb4`), wrong ODBC connector version | `Databases/mysql-administration-guide.md` |
| Restored guest breaks production | Restored with the production IP/hostname and started on the production network | The isolation table above |
| Snapshot has nothing in it | Snapshots not enabled on that share, or the volume is ext4 (snapshots require Btrfs) | `synology-nas-administration-guide.md` |
| Offsite restore cannot proceed | Disk not in the safe, media set not rotated, nobody has safe access | `Retrospect Restore Procedure.md` |
| Restore works but takes far longer than assumed | Nobody had ever measured it | The timing section above. This is a finding, not a pass |

---

## Operations (Day-2)

### Quarterly — review the test matrix

- [ ] Every system in the matrix has a `Next due` date that has not passed.
- [ ] Any system with no recorded test at all is escalated, not deferred.
- [ ] New hosts added since the last review are in the matrix. A host
      missing from the matrix is usually a host missing from the backup.
- [ ] Decommissioned hosts are removed, so the matrix stays credible.

### Annually — review the procedure itself

- [ ] The cadences above are still right for the tiering, and the tiering
      still matches what the business says matters.
- [ ] Measured restore times are compared against the agreed RTOs in
      `SysAdmin Procedures/business-continuity-plan.md`. Where the measured
      time exceeds the target, that is a gap to be closed or a target to be
      renegotiated — not a number to quietly leave alone.
- [ ] At least one test was run by someone other than the system owner.
- [ ] At least one test was run from the offsite media set.

### After every restore performed in anger

- [ ] Log it here with real timings. A real recovery is the highest-quality
      data point available and it is routinely thrown away.

## Troubleshooting

### Symptom: the backup you want is not listed in Retrospect

- Likely cause: only a limited number of days appears in the recent list;
  older backups must be retrieved from the media set catalog.
- Check / fix: select any backup from the correct machine and onsite media
  set, click **More Backups…**, find the date (remember the +1 day rule),
  click **Retrieve**, and watch **Activities** — the running-jobs count
  goes up by one and back down. Re-run the search to refresh the list.
  Full detail in `Retrospect Restore Procedure.md` step 4.
- **Count this time as T1.** It is part of a real recovery.

### Symptom: Retrospect asks for a disk that is not available

- Likely cause: you selected the **Offsite** media set. Its disks live in
  the offsite safe.
- Fix: for a routine test, cancel and use the onsite set. For a deliberate
  offsite test, retrieve the disk — and record how long that took, because
  that delay is real in a disaster.
- [FILL IN: who has access to the offsite safe, and what is the process and
  lead time for retrieving a disk? This is a dependency of the offsite
  recovery path and it is not documented anywhere.]

### Symptom: `pct restore` fails with "already exists"

- Likely cause: you used the production VMID.
- Fix: **do not add `--force`.** Restore to a spare VMID instead.
  `--force` destroys and replaces the existing container, which during a
  test would turn an exercise into an outage.

### Symptom: restore fails part-way with a space error

- Likely cause: the destination did not have room; archives are compressed
  and restore to several times their stored size.
- Check: `df -h` at the destination, and `pvesm status` for Proxmox
  storages, **before** starting.
- Fix: clear previous test restores from the Test drive (cleanup is part
  of the procedure), or choose a destination with room. Note it as a
  finding — if a test restore does not fit, a real one will not either.

### Symptom: the dump imports but row counts do not match production

- Likely causes, in order: production has changed since the dump was taken
  (compare against the dump's timestamp, not against right now); the dump
  did not include all databases; the job is failing intermittently.
- Check: the dump's own `--dump-date` header, the list of databases in
  the dump, and the backup job log for the intervening nights.
- **Do not dismiss a small discrepancy.** Confirm it is explained by
  elapsed time. An unexplained difference is a finding.

### Symptom: the restored system works but nothing can reach it

- Likely cause: exactly what isolation is for — the restored copy has no
  network, or a deliberately different address. This is a **pass**, not a
  failure, provided verification was done from the host itself or through
  the isolated path.
- Watch for the opposite failure: a restored guest that *can* reach the
  production network and has taken a production IP or registered in AD.
  Stop it immediately and check for collisions.

### Symptom: the test passes but took far longer than expected

- This is a **finding, not a pass**. Record the measured time, compare it
  against the assumed RTO, and raise the difference. An unachievable RTO
  that nobody knows is unachievable is worse than an honest longer one.

## Security

- **Test restores contain live data.** A restored copy of dms-db.example.com is
  payroll and member data with production sensitivity and no production
  controls. It must be protected as production: access-restricted while it
  exists, and destroyed as soon as the test is verified.
- Cleanup is a security control, not tidiness. [CONFIRM: proposed default
  — test restores containing member or payroll data are destroyed within
  one working day of the test completing, and the log entry records the
  destruction.]
- Restore destinations must not be more exposed than the source. A scratch
  database on a host reachable from a wider network than the original is a
  downgrade in protection.
- Credentials used for testing come from 1Password, never from this
  document. See `Security & Hardening/secrets-management-guide.md`.
- **Ransomware changes the requirements.** Recovery from ransomware
  depends on backups the attacker could not reach: the offsite Retrospect
  media in the offsite safe, and locked/immutable Synology snapshots if they
  are enabled (`[FILL IN: is snapshot locking enabled? The NAS guide flags
  this as unconfirmed]`). An offsite restore test is therefore also a
  ransomware-readiness test, and is the strongest argument for running one.
- [FILL IN: is Retrospect's backup data encrypted at rest, particularly on
  the offsite disks that leave the building? If not, that is worth
  recording as a decision rather than an omission.]

## Monitoring & Alerting

Testing sits on top of monitoring and does not replace it. The current
state, per `SysAdmin Procedures/monitoring-alerting-guide.md`, is that
backup job success is not comprehensively alerted. The MySQL guide states
it directly: **"Nothing alerts when this job fails. A dump that has not run
for three weeks looks exactly like one that ran last night, until you need
it."**

The two layers, and what each is for:

| Layer | Answers | Cadence |
|---|---|---|
| Backup monitoring | "Did the job run, and did it report success?" | Daily, automated |
| Restore testing | "Would the output of that job actually save us?" | This document |

Monitoring without testing gives a green dashboard over unusable media.
Testing without monitoring means a job can be broken for weeks between
tests. Both are needed.

- [FILL IN: does Retrospect send job success/failure notifications, and to
  whom? Confirm the address is monitored.]
- [FILL IN: do Proxmox `vzdump` jobs have "Email on error" or "always"
  configured (Datacenter → Backup)? The LXC guide notes this requires a
  working SMTP relay.]
- [FILL IN: is anything alerting on the MySQL nightly dump, the binary log
  copy process, or Synology snapshot task failures?]
- [CONFIRM: proposed default — add a daily automated freshness check that
  fails loudly when any backup artefact is older than its expected
  interval, and route it to graylog01.example.com alongside the existing
  streams.] Age of the newest artefact is the single most useful backup
  metric and is cheap to check.

## Disaster Recovery

This procedure is the evidence base for disaster recovery, not a
substitute for it.

- The restore procedures themselves live in the per-system documents
  listed in References.
- Sequencing during a real recovery — what comes back first, and what has
  to be working before the next thing can start — is in
  `Hardware & Backup/power-outage-shutdown-runbook.md` (bring-up order)
  and `SysAdmin Procedures/business-continuity-plan.md` (recovery
  prioritisation).
- **RTO and RPO targets do not exist yet.** They are not IT's to set
  alone. The business impact analysis in the business continuity plan is
  where they get decided; the measured times from this procedure are what
  make the decision an informed one rather than a wish.
- Escalation if the owner is unavailable: [FILL IN: name and contact].
  Note that this is the case the "run a test with someone other than the
  owner" rule above is designed to prepare for.

## Decisions & History (ADR-lite)

| Date | Decision / Change | Why / Ticket |
|---|---|---|
| 2026-07-09 | Retrospect file-level restore performed and verified (`data.example.com` MySQL backup folders, onsite media set) | Walk-through and verification; recorded in `Retrospect Restore Procedure.md` |
| 2026-09-11 | This procedure created | Library review found restore procedures but no evidence of restore testing and no restore-test log anywhere |
| [FILL IN] | Test tiering and cadences confirmed with Example Org leadership | [FILL IN] |
| [FILL IN] | First full depth-3 rebuild test completed | [FILL IN] |

## References

Restore procedures this document tests (follow these; do not duplicate
them):

- `Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md` —
  authoritative Retrospect restore procedure; tested 2026-07-09
- `Hardware & Backup/Retrospect Restore/Retrospect restore.txt` — the
  session transcript behind that procedure
- `Linux & Servers/lxc_backup_restore_proxmox91.txt` — LXC backup and
  restore, including restore-to-new-VMID
- `Linux & Servers/proxmox-cluster-administration-guide.md`
- `Databases/mysql-administration-guide.md` — dumps, binary logs,
  point-in-time recovery, replication
- `Databases/mysql replication patching.txt` — replication verification
  detail used by the MySQL test
- `Hardware & Backup/synology-nas-administration-guide.md` — snapshots and
  the Browse-vs-Restore distinction
- `Mail & Messaging/zimbra-mail-administration-guide.md`
- `Mail & Messaging/Share Mailboxes Zimbra` — shared mailbox configuration
  that a Zimbra restore must preserve
- `Dynamics GP & DMS/dynamics-gp-2018-admin-guide.md`

Related:

- `SysAdmin Procedures/business-continuity-plan.md` — where RTO/RPO
  targets and recovery prioritisation are decided
- `Hardware & Backup/power-outage-shutdown-runbook.md` — bring-up order
  and dependency gates
- `SysAdmin Procedures/monitoring-alerting-guide.md` — backup job
  monitoring, the layer below this one
- `SysAdmin Procedures/Change_Management_Log.txt` — where scheduled tests
  are logged
- `Security Procedures/IR_Security_Scenarios_Guide.md` — incident
  discipline, used when a test fails
- `Security & Hardening/secrets-management-guide.md`
- `SysAdmin Procedures/Backup_DR_Runbook.txt` — **unfilled boilerplate.**
  Its structure is useful; its hosts, addresses, retention periods and
  RPO/RTO figures are template defaults and describe no part of the organization
- `Hardware & Backup/Retrospect_Mac_User_Guide-EN.pdf` — vendor manual

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Synology NAS Administration (files.example.com)

## Overview

files.example.com is the organization's primary file server: a Synology NAS running DSM. It
holds the departmental SMB shares used by staff on both the macOS and
Windows 11 fleet, and it provides the point-in-time recovery capability
that saves the most support time — Snapshot Replication, which lets a
user's deleted or mangled file be recovered in minutes without touching
the backup system. Retrospect backs the NAS up for longer-term and
off-site retention.

This guide covers the administration of that NAS: layout, permissions,
quotas, snapshots, DSM updates, disk replacement, and the failure modes
that actually occur.

## Quick Facts

| Field            | Value                                                       |
|------------------|-------------------------------------------------------------|
| Owner            | IT lead (it@example.com)                          |
| Environment      | prod                                                        |
| Location         | Server room, [FILL IN: rack position]; model [FILL IN: Synology model], DSM [FILL IN: version] |
| Access           | DSM web UI https://files.example.com:5001 ; SMB \\\\files.example.com ; SSH [FILL IN: enabled? if so, port] |
| Dependencies     | AD domain example.com (authentication), DNS, network switching, APC Symmetra LX/RM UPS |
| Dependents       | All staff file access; [FILL IN: any server-side mounts — does Proxmox use the NAS as a storage backend? Do any application hosts mount shares?] |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                     |

## Hardware and Storage Layout

Populate from DSM → Storage Manager. Do not guess volume or share names.

| Field | Value |
|-------|-------|
| Model | [FILL IN] |
| Serial number | [FILL IN] |
| DSM version | [FILL IN] |
| Drive bays / populated | [FILL IN] |
| Expansion unit (DX/RX) | [FILL IN: present? model?] |
| Memory | [FILL IN] |
| Network interfaces / bonding | [FILL IN: is LACP or failover bonding configured?] |
| IP address | [FILL IN] |

### Storage pools and volumes

| Pool | RAID type | Drives | Volume | Filesystem | Capacity | Used | Purpose |
|------|-----------|--------|--------|------------|----------|------|---------|
| [FILL IN] | [FILL IN: SHR / RAID5 / RAID6 / RAID10] | [FILL IN] | [FILL IN] | [FILL IN: Btrfs or ext4] | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |

**Filesystem matters here.** Snapshot Replication requires **Btrfs**.
Any share on an ext4 volume cannot be snapshotted. [FILL IN: confirm
which volumes are Btrfs — if any share that users care about is on ext4,
that is a gap worth recording and planning to fix.]

### Share layout

| Share | Volume | Purpose | Who has access | Snapshots? | Quota |
|-------|--------|---------|----------------|------------|-------|
| Systems Admin | [FILL IN] | IT working files; holds the "Restore for <user>" restore-delivery folders | [FILL IN: IT staff group] | [FILL IN] | [FILL IN] |
| [FILL IN: share name] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: share name] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |

(The **Systems Admin** share is referenced by
`Retrospect Restore/Retrospect Restore Procedure.md` as the delivery
location for user restores. Everything else in this table needs to be
read out of DSM → Control Panel → Shared Folder.)

## How It Works

DSM presents storage in layers, and knowing which layer you are looking
at is most of troubleshooting:

```
Physical drives
  -> Storage Pool (RAID group: SHR/RAID5/RAID6 — provides redundancy)
    -> Volume (filesystem: Btrfs or ext4 — this is what fills up)
      -> Shared Folder (SMB share — this is what users see and what snapshots)
```

Redundancy lives at the **pool** level, capacity at the **volume** level,
snapshots and permissions at the **shared folder** level. A degraded pool
and a full volume are different problems with different fixes, and the
alerts look similar at a glance.

Access path for a normal user:

```
Windows/Mac client -> SMB (445) -> files.example.com -> shared folder
                                         # authenticated against AD (example.com)
```

Data protection has two independent layers, and they are not
substitutes for each other:

| Layer | Mechanism | Protects against | Does NOT protect against |
|-------|-----------|------------------|--------------------------|
| Snapshots | Synology Snapshot Replication, on-volume, Btrfs copy-on-write | User error: deleted, overwritten, or ransomware-encrypted files. Recovery in minutes. | Loss of the NAS, pool failure, volume corruption |
| Retrospect backup | Agent/network backup to the Retrospect (Archive) machine, onsite + offsite media sets | Loss of the NAS itself; long-term retention; off-site | Being slow — restores take much longer than snapshots |

**Snapshots are not a backup.** They live on the same volume as the data.
If the pool dies, the snapshots die with it. That is what Retrospect is
for.

Where things live:

| Item | Location |
|------|----------|
| DSM logs | Log Center in DSM (and `/var/log/` over SSH) |
| Snapshot config | Snapshot Replication app → Snapshots |
| Share permission config | Control Panel → Shared Folder → Permissions |
| AD join config | Control Panel → Domain/LDAP |
| Notification config | Control Panel → Notification |

## User and Group Permission Model

### AD integration

files.example.com is joined to the **example.com** Active Directory domain, so
staff authenticate with their normal domain credentials rather than
local NAS accounts.

| Field | Value |
|-------|-------|
| Domain | example.com |
| Domain controller(s) used | [FILL IN] |
| DSM join status | Control Panel → Domain/LDAP → should read "Connected" |
| Local (non-AD) accounts that exist and why | [FILL IN: there is at least a DSM admin account; list any others and their purpose] |

Verify the domain join:

```
# In DSM: Control Panel -> Domain/LDAP -> Domain -> check status, then "Domain Users" tab lists AD users
# Over SSH, if enabled:
sudo synogroup --enum                # enumerate groups DSM knows about, AD groups included
id [FILL IN: test AD username]       # resolves the AD user and their groups
```

**Time skew breaks AD authentication.** If domain logins fail suddenly
and nothing changed, check the NAS clock first (Control Panel → Regional
Options → Time). Kerberos will not tolerate more than a few minutes of
drift.

### How permissions layer

Three layers apply, and the effective permission is the **most
restrictive** of them. This is the single most common source of
"permission denied" confusion:

1. **Shared Folder permissions** (Control Panel → Shared Folder →
   Permissions) — per-user or per-group read-write / read-only / no
   access on the share as a whole.
2. **File/folder ACLs** inside the share — Windows-style ACLs on
   subfolders, set either from DSM's File Station or from Windows
   Explorer's Security tab.
3. **Advanced share settings** — a share set read-only, or a
   per-protocol restriction, overrides anything more permissive below.

### The rule to follow

**Assign share and folder permissions to AD groups, never to individual
users.** Individual grants are invisible in six months and are the
reason people still have access to shares they left behind two roles
ago.

| Practice | Why |
|----------|-----|
| Grant to AD security groups | Membership changes handled in AD, once, for all systems |
| One group per share per access level (read-write, read-only) | Makes "who can see this?" answerable |
| Document the group → share mapping in the share table above | Otherwise the NAS is the only record |
| Avoid Everyone / Domain Users on anything sensitive | It always turns out to mean more people than expected |

[FILL IN: record the actual AD group → share mapping in use. Get it from
Control Panel → Shared Folder → each share → Permissions.]

There is an existing permissions reference screenshot at the library
root, `[internal screenshot: DMS permission screen]` — [FILL IN: confirm whether
that documents a NAS share ACL, and if so fold it into the share table
above.]

### Checking effective access for a user

When a user reports they cannot get to something:

```
# In DSM: Control Panel -> Shared Folder -> select share -> Permissions
#   Switch the dropdown to "Local users"/"Domain users" and find the user
# Then: File Station -> navigate to the folder -> right-click -> Properties -> Permission
#   This shows the effective ACL, including inherited entries
```

Check in that order — share level first, then folder ACL. Nine times out
of ten the share level is fine and an inherited folder ACL is the
problem.

## Quota Management

Quotas keep one user or one share from consuming a volume and taking
every other share down with it. DSM has two distinct kinds:

| Quota type | Set where | Applies to |
|------------|-----------|------------|
| User quota | Control Panel → User & Group → user → Quota | That user's data across a volume |
| Shared folder quota | Control Panel → Shared Folder → edit share → Advanced | Total size of that share |

Recommended approach:

- Put a **shared folder quota** on every share that can grow without
  bound. A share with no quota is a share that will eventually fill the
  volume.
- Leave headroom: total of all quotas should sit below volume capacity,
  because snapshots also consume volume space.
- Set a quota **warning** threshold, not just a hard limit, so you hear
  about it before users do.

```
# In DSM: Control Panel -> Shared Folder -> [share] -> Edit -> Advanced -> Enable shared folder quota
# In DSM: Control Panel -> User & Group -> [user] -> Edit -> Quota
# Review current usage: Storage Manager -> Volume, and Control Panel -> Shared Folder (size column)
```

[FILL IN: record which shares currently have quotas and their values, and
decide a standard warning threshold — e.g. warn at 80%.]

**Snapshots consume quota-adjacent space.** A share at its quota whose
snapshots are also large can fill the volume even though the share looks
capped. Watch volume free space, not just per-share usage.

## Snapshot Replication

### What is configured

| Field | Value |
|-------|-------|
| Shares with snapshots enabled | [FILL IN] |
| Schedule | [FILL IN: how often — hourly? daily? at what time?] |
| Retention policy | [FILL IN: how many hourly/daily/weekly/monthly kept] |
| Replication to a second device? | [FILL IN: is replication to another Synology configured, or are these local snapshots only? This is the difference between a second copy and no second copy.] |
| Immutable / locked snapshots? | [FILL IN: DSM supports snapshot locking, which materially improves ransomware resistance — is it enabled?] |

Requirements and limits:

- **Btrfs volumes only.** ext4 shares cannot be snapshotted.
- Snapshots are near-instant and initially consume almost no space; they
  grow as the data diverges from the snapshot.
- Snapshots live on the same volume as the data — again, **not a
  backup**.

### Configuring snapshots on a share

```
# Snapshot Replication app -> Snapshots -> select the share
#   -> Settings -> Schedule   : enable and set frequency
#   -> Settings -> Retention  : set how many are kept (GFS-style rules available)
#   -> Settings -> Advanced   : "Make snapshot visible" (see below)
#   -> Take a manual snapshot with "Take a Snapshot" to confirm it works
```

Retention guidance: enough daily snapshots to cover a long weekend plus
the time it typically takes a user to notice a mistake. **[FILL IN:
agree a retention standard — e.g. 24 hourly, 30 daily, 8 weekly — and
record it here.]** Then confirm the volume has the free space to hold it.

### Restoring from a snapshot

Two routes. Prefer the first — it lets the user do the work.

#### Route 1 — Make snapshots visible to users (self-service)

This is the toggle to know about, carried forward from the original note
(`_ARCHIVE/superseded-stubs/Retrieve files from synology snapshot.txt` (archived 2026-09-11)):

> **Snapshots from files.example.com**
>
> Snapshot Replication (app) → Snapshots → click on Share → Settings →
> Advanced → **"Make snapshot visible"** is the toggle (in Syno web
> admin)

With that enabled, snapshots surface to clients as Windows **Previous
Versions**:

- **Windows:** right-click the file or folder on the mapped drive →
  Properties → **Previous Versions** tab → pick a timestamp → Restore
  or Copy.
- **macOS:** Previous Versions is not exposed in Finder. Mac users need
  the admin-side route below, or you can browse the snapshot in DSM
  File Station and hand them the file.

Notes on the toggle:

- It is **per shared folder**, not global. Enabling it on one share does
  nothing for the others.
- It only exposes what the snapshot retention actually holds. If
  retention keeps 7 days, Previous Versions shows 7 days.
- Making snapshots visible does **not** let users delete snapshots, and
  respects existing permissions — a user sees only what they could
  already read.
- It is the single highest-value setting on this NAS for reducing
  restore tickets. **[FILL IN: confirm it is enabled on every user-
  facing share; enable it where it is not.]**

#### Route 2 — Restore from DSM (admin)

```
# Snapshot Replication app -> Snapshots -> select the share -> select the snapshot
#   -> "Browse"  : open the snapshot read-only, copy individual files out (SAFEST — do this)
#   -> "Restore" : roll the ENTIRE share back to that point in time
```

**"Restore" rolls the whole share back.** Any change made after that
snapshot is lost for every user of the share, not just the one who asked.
Use **Browse** and copy the specific files out unless a full rollback is
genuinely what is wanted — and if it is, tell the share's users first.

For delivering restored files to users, follow the same convention as
Retrospect restores: copy into the **"Restore for <user>"**
folder on the **Systems Admin** share. See
`Retrospect Restore/Retrospect Restore Procedure.md`.

### Which recovery tool to reach for

| Situation | Use |
|-----------|-----|
| File deleted or overwritten today, or in the last few days | **Snapshot** — minutes, self-service if visible |
| File needed from weeks or months ago | **Retrospect** — `Retrospect Restore/Retrospect Restore Procedure.md` |
| Whole share mangled recently | Snapshot, full restore (warn users first) |
| NAS itself lost or pool failed | **Retrospect** — snapshots died with the volume |
| Off-site copy needed | Retrospect offsite media set (disk from the offsite safe) |

## Operations (Day-2)

### Health Check

Run through this weekly, or whenever something feels off:

```
# In DSM:
#   Storage Manager -> Overview      : pool and volume status, expect "Healthy"
#   Storage Manager -> HDD/SSD       : per-drive health and SMART status
#   Log Center                       : errors and warnings since last check
#   Control Panel -> Domain/LDAP     : domain join still "Connected"
#   Snapshot Replication -> Snapshots: last snapshot timestamp is recent
#   Resource Monitor                 : load, and active connections

# Over SSH, if enabled:
df -h                          # volume free space
cat /proc/mdstat               # RAID array state — [UU] good, [U_] degraded
sudo smartctl -H /dev/sata1    # SMART overall-health for one drive; repeat per drive
```

### Logs

```
# DSM: Log Center -> Logs (system, connection, file transfer)
# Log Center -> Notification for alert routing
```

[FILL IN: is the NAS forwarding syslog to graylog01.example.com? If not,
consider it — Log Center → Log Sending. Centralised logs are how you
notice a disk going bad before it fails.]

### Notifications

Alerts you want to actually arrive: disk failure, volume degraded,
volume nearly full, UPS events, failed snapshot/replication.

```
# In DSM: Control Panel -> Notification -> Email  : set recipients and SMTP
#         Control Panel -> Notification -> Rules  : choose which events notify
```

[FILL IN: confirm the notification email address configured and that a
test notification actually arrives. Unmonitored storage alerts are how a
degraded array becomes a failed array.]

### DSM Update Procedure

DSM updates are routine but not risk-free: a DSM major version upgrade
can change SMB behaviour, break the AD join, or deprecate a package.
Treat a major version jump as a change, not a chore.

**Risks to be aware of:**

| Risk | Mitigation |
|------|------------|
| Reboot required — all shares go offline for several minutes | Schedule outside working hours; announce it |
| AD join can break on major upgrades | Test a domain login immediately after; be ready to re-join |
| SMB protocol/signing defaults can change, breaking older clients | Note the current SMB min/max version before upgrading |
| Packages may be unsupported on the new DSM | Check Snapshot Replication and any other installed package |
| Retrospect client compatibility | Verify the next backup actually runs after the upgrade |
| Rollback is not supported | A DSM upgrade is effectively one-way. Have backups verified first |

**Procedure:**

1. **Pre-checks**
   - [ ] Read the release notes for the target DSM version, specifically
         the "important notes" section.
   - [ ] Confirm a recent, verified Retrospect backup exists.
   - [ ] Confirm pool and volumes are **Healthy** — never update a
         degraded array. Fix the array first.
   - [ ] Take a manual snapshot of key shares.
   - [ ] Export the current config: Control Panel → Update & Restore →
         **Configuration Backup**. Save it off the NAS.
   - [ ] Record current state: DSM version, SMB settings, domain join
         status, package versions.
   - [ ] Confirm no backup or replication job is running.
   - [ ] Announce the outage window.
2. **Update**
   ```
   # In DSM: Control Panel -> Update & Restore -> DSM Update -> Download -> Update Now
   #   The NAS reboots. Expect several minutes of unavailability; a major
   #   version upgrade can take considerably longer. Do NOT power-cycle it.
   ```
3. **Post-checks**
   - [ ] DSM version is as expected.
   - [ ] Storage Manager: pool and volumes Healthy.
   - [ ] Domain join still Connected; **test an actual AD login**.
   - [ ] Shares mount from both a Mac and a Windows 11 client.
   - [ ] Snapshot Replication schedules intact; take a manual snapshot.
   - [ ] "Make snapshot visible" still enabled where it should be —
         verify Previous Versions still works from a Windows client.
   - [ ] Next Retrospect backup completes successfully.
   - [ ] Log Center: no new recurring errors.

Auto-update policy: **[FILL IN: is DSM auto-update enabled? Recommended
is automatic for minor/security updates only, with major version
upgrades done manually in a window. Record the decision.]**

### SMART Monitoring and Disk Replacement

**Monitoring:**

```
# In DSM: Storage Manager -> HDD/SSD -> select drive -> Health Info
#   Check: overall status, reallocated sector count, bad sector count,
#          and the SMART test history
# Schedule tests: Storage Manager -> HDD/SSD -> Settings -> S.M.A.R.T. Test
#   Recommended: quick test weekly, extended test monthly
```

[FILL IN: confirm scheduled SMART tests are enabled and record the
schedule.]

Warning signs that mean plan a replacement now, not later: rising
reallocated sector count, any bad sectors, a failed extended SMART test,
or DSM marking a drive "Warning" / "Critical". DSM will often keep using
a drive it has flagged — flagged is not failed, but it is borrowed time.

**Replacement procedure:**

1. **Identify the drive precisely.** Storage Manager → HDD/SSD → note
   the **bay number**, model, and serial. Then confirm the serial
   physically before pulling anything.
   ```
   # In DSM: Storage Manager -> HDD/SSD -> select the drive -> note Bay, Model, Serial
   #   Many models can flash the bay LED to identify it — use that if available
   ```
   **Pulling the wrong drive from a degraded array destroys the pool.**
   If the array is already degraded, you have no redundancy left — verify
   the serial twice.
2. **Confirm you have the right replacement.** Same or larger capacity,
   and a compatible model. [FILL IN: what drive model and capacity is in
   use, and is a cold spare held on site? If not, that is worth fixing.]
3. **Verify backups before touching hardware.** A rebuild puts every
   remaining drive under sustained load, which is exactly when a second
   marginal drive fails. Confirm Retrospect has a current, verified
   backup.
4. **Replace.** These units are hot-swappable. [FILL IN: confirm hot-swap
   support for this specific model — if it is not hot-swappable, shut the
   NAS down first per the power-outage runbook's storage tier.]
   ```
   # Release the bay latch, pull the failed drive, seat the replacement fully
   ```
5. **Repair the pool.**
   ```
   # In DSM: Storage Manager -> Storage Pool -> select pool -> Repair
   #   Select the new drive and confirm. Rebuild starts.
   ```
6. **During the rebuild:**
   - Performance will be noticeably degraded. Expect user complaints.
   - Rebuild time scales with drive size — hours to well over a day.
   - **Do not reboot, update DSM, or start other heavy jobs.**
   - Monitor: Storage Manager shows progress; `cat /proc/mdstat` over SSH
     shows the same with a percentage.
7. **After:**
   - [ ] Pool status back to **Healthy**.
   - [ ] New drive passes an extended SMART test.
   - [ ] Order a replacement spare if you used the only one.
   - [ ] Record the swap in Decisions & History below.

### Retrospect / Backup Interaction

Retrospect is the off-NAS backup layer. Per
`Retrospect Restore/Retrospect Restore Procedure.md`:

- Backups run **daily at 2:00 AM**, so a given day's files land in the
  **next day's** backup. Recovering files from the 8th–9th means
  restoring from the 10th or 11th backup.
- Each server has two media sets: **Server Backups 2026** (onsite) and
  **Server Backups Offsite 2026**. Restore from the **onsite** set; the
  offsite set may ask for a disk stored in the offsite safe.
- Restores are run from the Retrospect console on the Retrospect
  (Archive) machine, restored to the **Test** drive with **"Restore to a
  new folder" checked** (skipping that overwrites the destination
  drive), then copied to the **"Restore for <user>"** folder on
  the **Systems Admin** share.

[FILL IN: how exactly does Retrospect back up the NAS — a Retrospect
client installed on DSM, a network share source mounted by the
Retrospect server, or a rsync/Hyper Backup job that Retrospect then
picks up? This determines what happens when DSM is upgraded and what a
bare-metal NAS recovery looks like.]

Interaction notes that matter:

- Do not run a DSM update or a pool rebuild during the 2:00 AM backup
  window.
- After any NAS outage, confirm the next backup ran — see the post-
  restoration checks in
  `Hardware & Backup/power-outage-shutdown-runbook.md`.
- Snapshots and Retrospect are independent. Losing one does not affect
  the other, which is the point.

### Graceful Shutdown

The NAS is tier 6 in the shutdown order — after hypervisors, before
network gear. Full procedure:
`Hardware & Backup/power-outage-shutdown-runbook.md`.

```
# Preferred: DSM -> top-right user menu -> Shutdown
# Confirm no active connections first: Resource Monitor -> Connections
ssh [FILL IN: DSM admin account]@files.example.com "sudo shutdown -h now"   # if DSM UI is unreachable
```

Let DSM finish. Interrupting its shutdown is a common cause of an
unclean volume on the next boot.

## Troubleshooting

### Symptom: Share unreachable — users cannot connect to \\\\files.example.com

- Likely causes, in order: network/DNS, the NAS is down or rebooting,
  SMB service stopped, or the volume is unmounted.
- Check:
  ```
  ping -c 4 files.example.com                      # is the host up at all?
  nslookup files.example.com                        # does DNS resolve to the right IP?
  nc -vz files.example.com 445                      # is SMB listening?
  nc -vz files.example.com 5001                     # is DSM's web UI listening?
  smbclient -L //files.example.com -U [FILL IN: test account]   # can we enumerate shares?
  ```
- Fix:
  - Host unreachable but powered on → check switch port and link, then
    the console.
  - DSM reachable but SMB not → Control Panel → File Services → SMB →
    confirm enabled, then toggle it off and on.
  - Volume unmounted → Storage Manager will say so. Do **not** force
    anything; see "volume full" and "degraded array" below first.
  - DNS resolving to a stale IP → fix DNS, then have clients reconnect.
- macOS-specific: a Mac that will not reconnect often has a stale
  credential in Keychain. Remove the files.example.com entry from Keychain
  Access and reconnect. Also try `Finder → Go → Connect to Server →
  smb://files.example.com`.
- Windows-specific: a stale mapped drive survives as a dead letter.
  ```
  net use * /delete /y           # drop all mapped drives, then remap
  net use Z: \\files.example.com\[FILL IN: share] /persistent:yes
  ```

### Symptom: Permission denied on a folder the user should be able to access

- Likely cause: the effective permission is the most restrictive of
  share permission, folder ACL, and advanced share settings. Usually an
  inherited folder ACL.
- Check, in this order:
  ```
  # 1. Share level: Control Panel -> Shared Folder -> [share] -> Permissions
  #      Is the user's AD GROUP granted read-write here?
  # 2. Folder ACL: File Station -> navigate to folder -> Properties -> Permission
  #      Look for an explicit Deny, or missing inheritance
  # 3. Advanced: Control Panel -> Shared Folder -> [share] -> Edit -> Advanced
  #      Is the share read-only? Any protocol restriction?
  # 4. Group membership: is the user actually in the group AD says they are?
  id [FILL IN: AD username]      # over SSH: shows resolved group membership
  ```
- Fix: grant at the **group** level, not to the individual. If the user
  genuinely needs access and no group fits, that is a sign the group
  model needs a new group — not a one-off exception.
- If **all** domain users suddenly lose access: this is not a permission
  problem, it is the AD join. Check Control Panel → Domain/LDAP status
  and the NAS clock. Re-join the domain if the trust has broken.
- A user who has just been added to a group may need to log out and back
  in (or disconnect and reconnect the share) before the new membership
  applies to their session.

### Symptom: Volume full or nearly full

- Likely causes: genuine growth, a share with no quota, or snapshots
  consuming more than expected.
- Check:
  ```
  # Storage Manager -> Volume : used vs total
  # Control Panel -> Shared Folder : per-share sizes, find the big one
  # Snapshot Replication -> Snapshots : space consumed by snapshots per share
  # Over SSH:
  df -h                          # volume-level free space
  du -sh /volume1/*              # per-share usage, largest first (adjust volume path)
  ```
- Fix, in order of preference:
  1. Find and remove genuine junk — old exports, duplicate archives,
     abandoned project folders. Get the data owner's agreement first.
  2. Tighten snapshot retention on the largest consumers. Deleting old
     snapshots frees space immediately.
  3. Apply or lower a shared folder quota so it cannot recur.
  4. Expand the volume/pool if the growth is legitimate.
- **Do not delete snapshots to free space without checking what you are
  giving up.** Those snapshots are the fast restore path. Reduce
  retention deliberately rather than clearing everything.
- A **full Btrfs volume misbehaves** — writes fail in odd ways and
  snapshot operations can fail too. Treat above [FILL IN: threshold,
  suggest 85%] as needing action, not as a note for later.

### Symptom: Storage pool degraded / drive failed

- Likely cause: a drive has failed or been dropped from the array.
- Check:
  ```
  # Storage Manager -> Storage Pool : status will read "Degraded"
  # Storage Manager -> HDD/SSD      : identify which drive, note Bay/Model/Serial
  # Over SSH:
  cat /proc/mdstat               # [U_] means one member missing; shows rebuild % if rebuilding
  sudo smartctl -a /dev/sata1    # full SMART for a suspect drive
  ```
- Fix: follow the disk replacement procedure above. In short: verify
  backups first, identify the drive by **serial** before pulling,
  replace, then Storage Manager → Storage Pool → Repair.
- **While degraded you have no redundancy.** For most RAID levels the
  next drive failure loses the pool. Treat a degraded array as urgent,
  not routine, and do not run DSM updates or heavy jobs until the
  rebuild completes.
- If **two** drives have failed on a single-redundancy pool, stop. Do
  not attempt a repair, do not power-cycle hopefully. The recovery path
  is a rebuild plus a full Retrospect restore — and any further
  improvisation can make that harder.

### Symptom: Snapshots not being taken, or Previous Versions is empty for users

- Check:
  ```
  # Snapshot Replication -> Snapshots -> [share] : last snapshot timestamp
  # Snapshot Replication -> [share] -> Settings -> Schedule : is it enabled?
  # Snapshot Replication -> [share] -> Settings -> Advanced : "Make snapshot visible"
  # Log Center : snapshot task failures
  # Storage Manager -> Volume : is the volume Btrfs? ext4 cannot snapshot.
  #                             Is it full? A full volume can fail snapshot creation.
  ```
- Fix:
  - Schedule disabled → enable it.
  - Volume is ext4 → snapshots are not possible on that volume. This is
    a structural gap; record it and plan a migration.
  - Volume full → free space (see above); snapshot creation then
    resumes.
  - Users see an empty Previous Versions tab → "Make snapshot visible"
    is off for that share. It is per-share. Turn it on.
  - Mac users see nothing → expected; Previous Versions is a Windows
    feature. Use the DSM Browse route for them.

### Symptom: NAS is slow

- Check:
  ```
  # Resource Monitor -> Performance : CPU, RAM, disk utilisation, network throughput
  # Resource Monitor -> Connections : who is connected and doing what
  # Storage Manager -> Storage Pool : is a rebuild or parity check running?
  ```
- Common causes: an in-progress RAID rebuild or scheduled parity
  consistency check (both expected, both temporary), a backup job
  running outside its window, a single client doing a bulk copy, or a
  drive that is failing slowly and dragging the array.
- A drive with rising reallocated sectors can make an array slow long
  before DSM calls it failed. If performance degraded with no other
  explanation, look hard at per-drive SMART.

## Security

- Exposure: internal only. SMB 445, DSM 5000/5001. [FILL IN: confirm the
  NAS is not reachable from the internet and that no port forwarding
  exists to it. This is worth verifying explicitly — an internet-exposed
  NAS is a ransomware target.]
- Auth: AD domain example.com for staff. Local DSM admin account for
  administration. [FILL IN: is the default "admin" account disabled or
  renamed? Is 2FA enabled on admin accounts?]
- SMB: [FILL IN: record the configured minimum SMB protocol version.
  SMB1 must be disabled — confirm it is.]
- Certificates: DSM web UI at :5001 uses [FILL IN: self-signed DSM cert,
  or a trusted cert? If trusted, record source, renewal process, and
  expiry].
- Secrets: DSM admin credentials and any service account live in the
  password manager, never in this document. [FILL IN: name the vault
  entry.]
- Ransomware posture: snapshots are the primary fast recovery path, and
  **locked/immutable snapshots** are what stop an attacker with admin
  credentials from deleting them. [FILL IN: enable and record snapshot
  locking, or record the decision not to.]
- [FILL IN: is SSH enabled on the NAS? If it is enabled only for
  occasional admin work, consider disabling it between uses and record
  the decision either way.]

## Monitoring & Alerting

- DSM Log Center is the local record; [FILL IN: confirm whether logs are
  forwarded to graylog01.example.com].
- Notification rules: Control Panel → Notification. [FILL IN: confirm
  recipients and that disk/volume/UPS alerts are enabled.]
- What normal looks like: pool and volumes **Healthy**, volume usage
  below [FILL IN: threshold], all drives SMART-normal, last snapshot
  within the scheduled interval, domain join Connected, last Retrospect
  backup successful.
- [FILL IN: record baseline volume usage so growth trends are visible.]

## Disaster Recovery

- RPO/RTO: file servers target RPO 24 hours / RTO 8 hours per
  `SysAdmin Procedures/Backup_DR_Runbook.txt`. Snapshots give a much
  better effective RPO for user-error recovery, but only while the NAS
  itself survives.
- Single drive failure → disk replacement procedure above. No data loss.
- Volume/pool loss → rebuild the pool, recreate shares and permissions,
  restore data from Retrospect. Snapshots are gone with the volume.
- **Whole NAS loss** → [FILL IN: is there a documented bare-metal
  recovery path? Record: where the DSM Configuration Backup is stored,
  whether a replacement chassis is available or would need purchasing,
  and the expected time to restore the full data set from Retrospect.
  This is the biggest gap in this document.]
- Power events → `Hardware & Backup/power-outage-shutdown-runbook.md`.
- Escalation if the owner is unavailable: [FILL IN: name and contact].

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why / Ticket |
|------------|-------------------|--------------|
| 2026-09-11 | Initial draft; absorbed the standalone snapshot-visibility note | Consolidating NAS knowledge into one guide |
| [FILL IN]  | [FILL IN: when was this NAS commissioned? Any pool expansions or drive replacements to date?] | [FILL IN] |

## References

- `_ARCHIVE/superseded-stubs/Retrieve files from synology snapshot.txt` (archived 2026-09-11) — the
  original snapshot-visibility note, now carried forward above
- `Retrospect Restore/Retrospect Restore Procedure.md` — authoritative
  restore procedure for Retrospect
- `Hardware & Backup/Retrospect_Mac_User_Guide-EN.pdf`
- `Hardware & Backup/power-outage-shutdown-runbook.md`
- `SysAdmin Procedures/Backup_DR_Runbook.txt`
- `Linux & Servers/proxmox-cluster-administration-guide.md`
- Upstream: Synology DSM Administrator's Guide; Snapshot Replication
  help for the DSM version in use

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

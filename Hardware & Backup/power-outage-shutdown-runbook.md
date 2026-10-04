> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Power Outage & Graceful Shutdown Runbook

## Overview

This runbook covers what to do when the server room loses utility power,
or when power must be cut deliberately for electrical work. It defines
when to start a controlled shutdown, the order in which systems go down,
the order in which they come back up, and what to verify at each step.

The goal is simple: **never let the UPS run flat while systems are still
writing to disk.** A clean shutdown with hours of downtime is a good
outcome. A hard power loss mid-write on a database or a storage array is
how you end up restoring from backup.

Read this before you need it. When the room is beeping is not when you
want to be reading a table of contents.

## Quick Facts

| Field            | Value                                                       |
|------------------|-------------------------------------------------------------|
| Owner            | IT lead (it@example.com)                          |
| Environment      | prod — server room                                          |
| Location         | Server room, [FILL IN: room/building identifier]            |
| Access           | Physical access to server room; UPS network management card web UI at [FILL IN: NMC hostname/IP]; SSH to each host |
| Dependencies     | APC Symmetra LX/RM UPS, building power, [FILL IN: generator — is there one? If yes, transfer time and fuel runtime] |
| Dependents       | Every service Example Org runs                                      |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                     |

## UPS Capacity and Expected Runtime

The server room is protected by an **APC Symmetra LX/RM**. Its
configuration is recorded in
`Hardware & Backup/UPS Specs/Symmetra LX_RM Template.txt` — that file is
a blank template and needs to be filled in from the PowerView display.

| Field | Value | Where to read it |
|-------|-------|------------------|
| Capacity (kVA) | [FILL IN] | PowerView → Status |
| Current load (kVA / %) | [FILL IN] | PowerView → Status → Load |
| Fault tolerance (n+_) | [FILL IN] | PowerView → Status |
| Number of power modules | [FILL IN] | PowerView → Status |
| Number of battery modules | [FILL IN] | PowerView → Status |
| **Expected runtime at current load** | **[FILL IN]** | PowerView → Status → Runtime |
| Runtime alarm threshold | [FILL IN] | PowerView → Setup → Alarm Runtime |
| Load alarm threshold (kVA) | [FILL IN] | PowerView → Setup → Alarm Load |
| Battery install / last replacement date | [FILL IN] | Battery module labels; also PowerView → Diags |
| Last self-test result and date | [FILL IN] | PowerView → Status → Self-test |
| NMC firmware / IP | [FILL IN] | Network Management Card web UI |

**Critical caveats about runtime:**

- The runtime figure the UPS reports is for the **current** load. Load
  changes as systems shut down, so runtime extends as you work through
  the shutdown order. Do not plan against a single number.
- Battery capacity degrades with age. A Symmetra with batteries past
  their service life can report optimistic runtime and then collapse
  under load. If the batteries' age is unknown, **assume you have much
  less time than the display claims.**
- **[FILL IN: measure and record the actual wall-clock time a full
  controlled shutdown takes, end to end, from the first command to the
  last host powering off. Until this is measured, the shutdown decision
  threshold below is a guess.]** This is the single most important
  number in this document.

### UPS Client / Notification Chain

The UPS notifies attached hosts so they can shut themselves down. The
current client list is recorded in a screenshot at
`Hardware & Backup/UPS Specs/ups clients list.png` — **[FILL IN: that
screenshot dates from 2017 and cannot be trusted as current. Log in to
the UPS network management card, export the current client list, and
record it in the table below.]**

| Host | Notification method | Shutdown delay | Action on low battery | Verified working |
|------|---------------------|----------------|-----------------------|------------------|
| [FILL IN] | [FILL IN: NUT / PowerChute / SNMP trap] | [FILL IN] | [FILL IN] | [FILL IN: date last tested] |
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |

Questions this table must answer, because the answers change what you do
during an outage:

- **[FILL IN: which hosts shut themselves down automatically, and which
  require a human? Anything not on the automatic list is your job.]**
- **[FILL IN: is there a NUT server, and if so which host runs it? If
  the NUT server itself shuts down early, its clients stop receiving
  notifications and will not shut down at all — confirm the NUT server
  is ordered to go down LAST among the notified hosts.]**
- **[FILL IN: what shutdown delay is configured on each client, and does
  the sum of those delays fit inside the measured UPS runtime?]**
- [FILL IN: where do UPS alerts go — email address, SNMP trap
  destination, Graylog stream on graylog01.example.com?]

Verify the notification chain on the UPS itself:

```
# From the NMC web UI: Configuration -> Shutdown -> client list
# From a NUT client host:
upsc [FILL IN: UPS name]@[FILL IN: NUT server]   # current UPS state: OL, OB, LB
systemctl status nut-monitor                      # is the client actually monitoring?
journalctl -u nut-monitor --since today           # did it see the last event?
```

An untested notification chain is a rumour. **[FILL IN: schedule and
record a test — a self-test or a controlled transfer — confirming that
each client actually receives and acts on the event.]**

## How It Works

Utility power feeds the APC Symmetra LX/RM, which feeds the server room
racks. On utility loss the Symmetra switches to battery without
interruption and begins signalling.

```
Utility power -> APC Symmetra LX/RM -> rack PDUs -> hosts
                        |
                        +-> Network Management Card -> SNMP/NUT -> UPS clients
                        +-> PowerView display (local status and alarms)
```

Two independent things have to happen during an outage: the UPS must
tell the hosts, and the hosts must shut down in the right order. The
automatic chain handles the first. The **order** is what this runbook
exists for, because automatic shutdown clients generally do not
coordinate between hosts — they each shut down on their own timer, which
can take a database down before the application writing to it.

**Assume the automatic chain is not enough.** If you can get to the
server room or to a terminal, drive the shutdown manually using the
order below.

---

# DECISION: Shut Down or Ride It Out?

Make this call **early**. The worst outcome is deciding to shut down
after the batteries are already low, because a rushed shutdown of the
storage and database tier is exactly what a graceful shutdown was
supposed to avoid.

**Start a controlled shutdown immediately if ANY of these is true:**

| Condition | Why |
|-----------|-----|
| Utility restoration is unknown or estimated beyond [FILL IN: threshold, e.g. 20 minutes] | You cannot shut down safely on a dead battery |
| UPS remaining runtime is below **2× the measured full shutdown time** | Below this you have no margin for anything going wrong |
| UPS reports a fault, bad battery module, or bad power module | Reported runtime is not trustworthy |
| The UPS is on bypass, or has failed to transfer cleanly | The load is unprotected — the next dip kills everything |
| Room cooling is also down and temperature is rising | [FILL IN: at what temperature do we shut down regardless of power? Check the room's temperature monitoring.] |
| The outage is planned electrical work | Shut down in advance, on your schedule, not theirs |

**You may ride it out if ALL of these are true:**

- The utility provider has confirmed restoration within a short, known
  window.
- UPS runtime comfortably exceeds that window **plus** a full shutdown's
  worth of margin.
- The UPS reports no faults, all modules healthy.
- Cooling is unaffected, or the room is cool enough to coast.
- Someone is physically present and watching the UPS, ready to trigger
  the shutdown if anything changes.

**When in doubt, shut down.** A planned outage costs an afternoon. A
corrupted database costs days and may cost data.

**Do not walk away from a UPS on battery.** If you decide to ride it
out, someone watches it.

---

# SHUTDOWN ORDER

Work **top to bottom**. Complete and verify each tier before starting
the next. The principle is that everything shuts down before the things
it depends on.

Do not skip the verification steps. A host that says "shutting down" and
then hangs on an unmounting filesystem is a host that is still powered
when you cut the rack.

| # | Tier | What | Why this order |
|---|------|------|----------------|
| 1 | Workstations & user devices | Macs, Windows workstations, Parallels VMs | Stops user writes to file shares first |
| 2 | Application tier | App servers, Docker workloads, internal apps | Stops writes to the database |
| 3 | Web tier | Reverse proxies, web front ends | Stops new requests arriving |
| 4 | Database tier | Database servers | Safe to stop only once nothing is writing |
| 5 | Hypervisors | Proxmox nodes | Only once all guests are down |
| 6 | Storage | Synology NAS files.example.com | Everything that mounts it is already down |
| 7 | Network gear | Switches, firewall, APs | Last — you need the network for all the above |

> **Order note:** tiers 2 and 3 are listed application-then-web because
> stopping the app tier first halts database writes soonest. If you
> would rather stop inbound traffic first, take the web tier down before
> the app tier — either is defensible. What is **not** negotiable is
> that both go down before the database, the database before the
> hypervisors, hypervisors before storage, and storage before network.

---

### Tier 1 — Workstations and User Devices

**Command / action per class of host:**

```
# macOS workstation, remotely:
ssh [FILL IN: admin account]@<mac-host> "sudo shutdown -h now"     # graceful; user is logged out

# Windows workstation, remotely:
shutdown /s /t 60 /m \\<windows-host> /c "Power outage - saving work"   # 60s warning to the user

# Windows guest under Parallels on a Mac:
#   Shut down the Windows guest from inside Windows FIRST, then the Mac host.
#   Suspending the guest instead of shutting it down risks a corrupt saved state.
```

Practical reality: in an outage, most workstations are on desk power and
will already be dead or on their own battery. Focus on machines in the
server room and any workstation known to hold open files on
files.example.com.

**Verify before proceeding:**

```
ping -c 2 <workstation>        # expect no reply — host is down
```

- [ ] Open file handles on the NAS checked: DSM → Control Panel →
      Shared Folder → [FILL IN: confirm the exact DSM path for open
      file/connection listing in the DSM version in use] — or Resource
      Monitor → Connections.
- [ ] Any user with unsaved work on a share has been told to save and
      disconnect (see Communications).

---

### Tier 2 — Application Tier

**Hosts:** [FILL IN: which hosts are the application tier? Candidates
from the environment include db02.example.com, sign.example.com, app02.example.com,
app01.example.com, prod01.example.com, graylog01.example.com — confirm each host's
actual role before relying on this list.]

**Command per class of host:**

```
# Linux application host — stop the app cleanly, then the OS:
ssh root@<app-host> "systemctl stop [FILL IN: service name]"   # clean app stop, flushes state
ssh root@<app-host> "systemctl stop docker"                      # if Docker workloads run here
ssh root@<app-host> "shutdown -h now"                            # then the host

# Docker host — stop containers gracefully first:
ssh root@<docker-host> "docker stop \$(docker ps -q) -t 60"      # 60s grace per container
ssh root@<docker-host> "shutdown -h now"

# LXC container on Proxmox (from the PVE node):
pct shutdown <vmid>            # graceful shutdown, honours the guest's init

# VM on Proxmox (from the PVE node):
qm shutdown <vmid>             # ACPI shutdown; needs guest agent or ACPI handling
```

Stop the application service before shutting down the host. A plain
`shutdown` usually does the right thing, but "usually" is not the
standard for an outage.

**Verify before proceeding:**

```
ssh root@<app-host> "uptime"   # expect connection refused — host is down
pct list                        # on each PVE node: app containers show 'stopped'
qm list                         # on each PVE node: app VMs show 'stopped'
```

- [ ] Every application host is confirmed down, not merely "shutting
      down". Give each host up to [FILL IN: grace period, e.g. 3
      minutes] then check the console before assuming it is off.

---

### Tier 3 — Web Tier

**Hosts:** [FILL IN: which hosts run the reverse proxy and web front
ends? Confirm roles for data.example.com / db02.example.com / sign.example.com /
prod01.example.com rather than assuming.]

**Command per class of host:**

```
ssh root@<web-host> "systemctl stop [FILL IN: caddy / nginx / apache2]"   # stop accepting requests
ssh root@<web-host> "shutdown -h now"

# If the web tier is a container or VM on Proxmox:
pct shutdown <vmid>            # container
qm shutdown <vmid>             # VM
```

**Verify before proceeding:**

```
curl -sS -m 5 https://[FILL IN: primary site URL]   # expect connection failure — site is down
ping -c 2 <web-host>                                 # expect no reply
```

- [ ] All web front ends are down and no longer accepting requests.
- [ ] Nothing is left holding a database connection. Confirm on the
      database host in the next step before stopping the engine.

---

### Tier 4 — Database Tier

**This is the tier where a hard power loss hurts most.** Do not rush it,
and do not let the UPS get here on a low battery.

**Hosts:** [FILL IN: confirm the database hosts — dms-db.example.com and
data.example.com are candidates; confirm which engines run where — MySQL/
MariaDB, PostgreSQL, MS SQL for Dynamics GP/DMS.]

**Command per engine:**

```
# Check nothing is still connected before stopping:
mysql -e "SHOW PROCESSLIST;"                          # MySQL/MariaDB: expect only system threads
sudo -u postgres psql -c "SELECT count(*) FROM pg_stat_activity WHERE state='active';"   # Postgres: expect 0-1

# Stop the engine cleanly:
systemctl stop mariadb                                 # or mysql — flushes and closes cleanly
systemctl stop postgresql                              # checkpoints before exiting

# MS SQL Server on Windows (Dynamics GP / DMS):
Stop-Service -Name MSSQLSERVER -Force                  # stops dependent services too

# Then the host:
shutdown -h now                                        # Linux
shutdown /s /t 30                                      # Windows, local
```

**A clean database stop can take minutes** while it checkpoints and
flushes. Let it finish. Do not "help" it with `kill -9`, and do not cut
power to a database that has not confirmed it is stopped.

**Verify before proceeding:**

```
systemctl is-active mariadb        # expect 'inactive'
systemctl is-active postgresql     # expect 'inactive'
journalctl -u mariadb -n 20        # look for a clean shutdown message, not a crash
ping -c 2 <db-host>                # after host shutdown: expect no reply
```

- [ ] Each database engine reported a **clean** shutdown in its log.
- [ ] Each database host is fully powered off.
- [ ] Note the time. If the databases went down dirty, record that —
      it changes the post-restoration checks.

---

### Tier 5 — Hypervisors (Proxmox nodes)

Only once **every guest** is down.

```
# On each PVE node, confirm nothing is still running:
qm list                        # expect all 'stopped'
pct list                       # expect all 'stopped'

# Stop any stragglers:
qm shutdown <vmid>             # graceful
pct shutdown <vmid>

# Then shut the node down:
shutdown -h now
```

Notes:

- Shut down PVE nodes **one at a time** and note that as nodes go down
  the cluster loses quorum. That is expected and fine during a full
  shutdown — quorum loss does not stop already-running guests, and you
  are stopping them deliberately anyway.
- If the last node is non-quorate and refuses to stop a guest, you can
  use `pvecm expected 1` on that node to regain write access to
  `/etc/pve` long enough to finish. This is safe **only** because you
  know the other nodes are genuinely down. See
  `Linux & Servers/proxmox-cluster-administration-guide.md`.
- If a guest will not shut down gracefully within [FILL IN: grace
  period], force it: `qm stop <vmid>` / `pct stop <vmid>`. This is a
  hard power-off for that guest — acceptable for a stateless web front
  end, **not** for a database.

**Verify before proceeding:**

```
ping -c 2 <pve-node>           # expect no reply, per node
```

- [ ] Every hypervisor node is powered off (check the front panel LEDs
      in the rack, not just ping).

---

### Tier 6 — Storage (Synology NAS, files.example.com)

Everything that mounts the NAS is now down, so it is safe to stop.

```
# Preferred: DSM web UI -> top-right user menu -> Shutdown
# Confirm no active connections first:
#   DSM -> Resource Monitor -> Connections   (expect none, or only your own session)

# Alternative, over SSH if DSM is unreachable:
ssh [FILL IN: DSM admin account]@files.example.com "sudo shutdown -h now"
```

Notes:

- Let DSM finish its own shutdown sequence. It unmounts volumes and
  parks the array; interrupting it is how you get a degraded or
  unclean volume on the way back up.
- If a Snapshot Replication or backup job is running, let it finish or
  cancel it cleanly in DSM first. Do not shut down mid-replication if it
  can be avoided.
- **[FILL IN: is there an expansion unit (DX/RX shelf)? If so, it must
  be powered off after the main unit and powered on BEFORE it on the way
  back up.]**

**Verify before proceeding:**

```
ping -c 2 files.example.com         # expect no reply
```

- [ ] NAS status LED shows powered off, not just "shutting down". The
      Synology takes a while — watch the chassis LED, not the clock.
- [ ] Expansion unit, if present, powered off after the main unit.

---

### Tier 7 — Network Gear (LAST)

Everything else is down. Now you can lose the network.

```
# Managed switch, if console access is available:
#   [FILL IN: exact save-and-shutdown commands for the switch models in use —
#    write configuration / copy running-config startup-config, then power off]

# Firewall / router:
#   [FILL IN: model and graceful shutdown method]
```

- Save any unsaved switch configuration **before** powering off.
- Note which port each device is on if you are unracking anything.
- Power off PDUs / cut the rack feed only after all of the above.

**Verify:**

- [ ] Switch configs saved.
- [ ] All devices powered off.
- [ ] UPS load has dropped to near zero — check PowerView. A load that
      is still significant means something is still running. **Find it
      before you stop.**
- [ ] Record the time the shutdown completed and the UPS runtime
      remaining at that moment. This number is how you calibrate the
      decision threshold next time.

---

# BRING-UP ORDER

Reverse of the shutdown, with a **dependency gate** at each tier: do not
start a tier until the tier below it is verified working. Starting an
application server before its database is up produces a broken
application that then needs restarting anyway, plus a pile of confusing
errors.

**Before powering anything on:**

- [ ] Utility power is confirmed **stable**, not just present. If power
      has been flickering, wait. Repeated transfers are harder on the
      UPS and on the hardware than a longer outage.
- [ ] UPS is back on line power (PowerView shows on-line, not on
      battery/bypass), reports no faults, and is charging.
- [ ] Room cooling is running.
- [ ] Physical inspection: nothing tripped, no burnt smell, no alarming
      LEDs.

| # | Tier | Gate before starting the next tier |
|---|------|-------------------------------------|
| 1 | Network gear | Switches up, links green, DNS/DHCP answering |
| 2 | Storage (files.example.com) | Volumes mounted, healthy, shares reachable |
| 3 | Hypervisors (Proxmox) | Nodes up, cluster quorate, all storages active |
| 4 | Database tier | Engines accepting connections, no recovery errors |
| 5 | Application tier | Apps started, connected to their databases |
| 6 | Web tier | Sites serving |
| 7 | Workstations | Users can log in and reach shares |

---

### Bring-up 1 — Network Gear

Power on switches, firewall, and APs first. Wait for them to fully boot —
a switch that is still booting looks like a dead network.

**Gate — verify before proceeding:**

```
ping -c 2 [FILL IN: default gateway IP]          # gateway responds
nslookup files.example.com [FILL IN: DNS server IP]   # DNS resolving
```

- [ ] Switch uplinks and server ports show link.
- [ ] DNS answering. **[FILL IN: where does internal DNS run? If it is
      on a VM that has not booted yet, note that DNS will not fully work
      until tier 3 — plan for it and use IPs until then.]**
- [ ] DHCP answering, if relevant. [FILL IN: where does DHCP run?]

---

### Bring-up 2 — Storage (files.example.com)

Power on the NAS. **[FILL IN: if an expansion unit exists, power it on
first and let it spin up before the main unit.]**

Synology boot takes several minutes. Be patient; do not power-cycle it
because DSM has not answered yet.

**Gate — verify before proceeding:**

```
ping -c 2 files.example.com                  # NAS network is up
# In DSM:
#   Storage Manager -> Storage Pool / Volume -> status must be "Healthy"
#   NOT "Degraded", NOT "Checking parity consistency" blocking access
```

- [ ] All volumes mounted and **Healthy**. If a volume is degraded or
      running a consistency check, see
      `Hardware & Backup/synology-nas-administration-guide.md` before
      putting load on it.
- [ ] Shares reachable from a test client:
      ```
      smbclient -L //files.example.com -U [FILL IN: test account]   # lists shares
      ```
- [ ] No disk shows a new SMART warning after the power event. Check
      Storage Manager → HDD/SSD.

**Do not start hypervisors until storage is healthy.** Guests whose
disks live on the NAS will fail to start, or worse, start against a
half-available storage backend.

---

### Bring-up 3 — Hypervisors (Proxmox)

Power on the PVE nodes. Bring up **all** nodes before starting guests, so
the cluster forms quorum properly.

**Gate — verify before proceeding:**

```
pvecm status                   # "Quorate: Yes", all nodes present
pvesm status                   # every storage 'active' — especially NAS-backed ones
systemctl --failed             # nothing failed at boot
journalctl -p err -b           # boot errors worth knowing about
```

- [ ] Cluster quorate, every node green in the web UI.
- [ ] Every storage active. If an NFS/CIFS storage is inactive, it is
      usually a mount race against the NAS — re-check the NAS then
      re-activate the storage before starting guests.
- [ ] Time is correct on every node: `timedatectl`. A wrong clock after
      a long outage breaks certificates, Kerberos/AD, and corosync.

---

### Bring-up 4 — Database Tier

Start database hosts and engines before anything that talks to them.

```
# Start the host, then:
systemctl start mariadb                        # or postgresql / MSSQLSERVER
systemctl status mariadb                       # confirm active (running)

# Read the startup log — this is the important part:
journalctl -u mariadb -n 50                    # look for crash recovery / InnoDB recovery messages
journalctl -u postgresql -n 50                 # look for "database system is ready to accept connections"
```

**Gate — verify before proceeding:**

```
mysql -e "SELECT 1;"                                       # engine answering
sudo -u postgres psql -c "SELECT 1;"                       # engine answering
mysqlcheck --all-databases --check                          # optional but wise after an unclean stop
```

- [ ] Engine started and accepting connections.
- [ ] **Startup log reviewed.** If the database performed crash
      recovery, note it — it means the stop was not clean, and the
      post-restoration checks below become mandatory rather than
      advisable.
- [ ] Replication, if any, is caught up. [FILL IN: is there database
      replication between hosts? If so, record how to check its lag.]

---

### Bring-up 5 — Application Tier

```
# Start the guest, then the app:
pct start <vmid>                                  # LXC
qm start <vmid>                                   # VM
systemctl start docker                            # if applicable
systemctl start [FILL IN: service name]           # the application itself
systemctl status [FILL IN: service name]          # confirm active
docker ps                                         # containers up, not restart-looping
```

**Gate — verify before proceeding:**

- [ ] Application service active and **stayed** active for a minute. A
      service that starts and dies is usually a database it cannot reach
      yet — go back to tier 4.
- [ ] Application logs show a successful database connection, not
      retry loops:
      ```
      journalctl -u [FILL IN: service name] --since "10 min ago"
      docker logs --since 10m <container>
      ```
- [ ] Scheduled jobs and timers are running: `systemctl list-timers`.

---

### Bring-up 6 — Web Tier

```
systemctl start [FILL IN: caddy / nginx / apache2]
systemctl status [FILL IN: caddy / nginx / apache2]
```

**Gate — verify before proceeding:**

```
curl -sSI https://[FILL IN: primary site URL]     # expect HTTP 200/302, valid TLS
```

- [ ] Each site loads, over HTTPS, with a valid certificate. A
      certificate error after an outage usually means a wrong clock —
      check `timedatectl` before assuming the certificate expired.
- [ ] A real user journey works, not just the front page. Log in and
      do one meaningful thing.

---

### Bring-up 7 — Workstations and Users

- [ ] Power on server-room workstations.
- [ ] Confirm a test user can log in (AD authentication working).
- [ ] Confirm mapped drives to files.example.com reconnect. Parallels guests
      may need their shared-folder drive mappings re-established — see
      `Windows & Mac Workstations/parallels-shared-folders-and-drive-mapping.md`.
- [ ] Confirm printing works. [FILL IN: where does the print service
      run?]
- [ ] Tell users they are back (see Communications).

---

# POST-RESTORATION CHECKS

Run these after everything is up. Some can wait until the next morning,
but the filesystem and backup checks should not.

### Filesystem integrity

```
# On each Linux host:
dmesg -T | grep -i -E "error|i/o|ext4|xfs|remount"   # I/O errors or read-only remounts after the event
mount | grep " ro,"                                    # any filesystem that went read-only
df -h                                                  # nothing unexpectedly full

# On the NAS:
#   Storage Manager -> Volume -> run a File System Check if any uncleanliness is suspected
#   (this takes a long time and the volume is unavailable during it — schedule it)
```

- [ ] No filesystem remounted read-only.
- [ ] No new I/O errors in the kernel log.
- [ ] NAS volumes Healthy; RAID not degraded or rebuilding unexpectedly.

### Replication

- [ ] Synology Snapshot Replication: confirm jobs resumed and the last
      successful replication timestamp is recent. See
      `Hardware & Backup/synology-nas-administration-guide.md`.
- [ ] [FILL IN: any database replication — confirm it resumed and lag
      returned to normal.]
- [ ] Proxmox replication jobs, if configured: `pvesr status`.

### Backup jobs

- [ ] Retrospect: confirm the scheduled backups ran, or run them
      manually if the window was missed. Media sets and schedules are
      described in `Retrospect Restore/Retrospect Restore Procedure.md`
      (daily at 2:00 AM). A missed night is a gap in the restore chain —
      note it.
- [ ] Proxmox vzdump jobs: Datacenter → Backup → check the task log.
- [ ] Scripted backups on the backup server — check the logs named in
      `SysAdmin Procedures/Backup_DR_Runbook.txt` section 6.
- [ ] If the outage caused a missed backup, run one now rather than
      waiting for the next scheduled window.

### Certificates and clock

```
timedatectl                                      # on every host: synced, correct timezone
chronyc sources                                  # or: systemctl status systemd-timesyncd
openssl s_client -connect [FILL IN: site]:443 </dev/null 2>/dev/null | openssl x509 -noout -dates
```

- [ ] Clocks correct and syncing everywhere. A host that booted with a
      dead RTC battery will have a wildly wrong clock and will fail AD
      authentication and TLS in confusing ways.
- [ ] AD domain trust intact — a large clock skew breaks Kerberos.
      Test a domain login.
- [ ] No certificates expired during the outage.

### General

- [ ] `systemctl --failed` clean on every Linux host.
- [ ] Monitoring/alerting is back: [FILL IN: confirm graylog01.example.com is
      receiving logs again, and any uptime checks are green].
- [ ] UPS: back on line, charging, self-test passed, no bad modules.
      Record the runtime figure now that the load is back to normal.
- [ ] **Write down what actually happened**, including how long the
      shutdown took and anything in this runbook that was wrong or
      missing. Update this document. The next outage is the only test
      that matters.

---

# COMMUNICATIONS

### Who to notify

| When | Who | Channel | Message content |
|------|-----|---------|-----------------|
| Outage detected, deciding | [FILL IN: IT lead / manager] | [FILL IN: phone/SMS — email will not work if mail is down] | Power is out, assessing, decision within N minutes |
| Shutdown starting | All staff | [FILL IN: channel that works without internal systems] | Systems going down now, save your work, expected duration |
| Shutdown complete | [FILL IN: IT lead / manager] | [FILL IN] | All systems down cleanly, awaiting power |
| Power restored, bringing up | [FILL IN: IT lead / manager] | [FILL IN] | Starting bring-up, estimate for service restoration |
| Services restored | All staff | [FILL IN] | Systems are back, report anything not working |
| After | [FILL IN: management] | [FILL IN] | Brief summary: cause, duration, impact, any data loss |

**[FILL IN: the critical gap — if email and internal chat run on
infrastructure that is being shut down, how do we reach staff? Record
the out-of-band method: personal mobile numbers held where, an SMS
list, a phone tree, or an external status page. Establish this before
it is needed.]**

### External contacts

| Contact | Purpose | Number |
|---------|---------|--------|
| [FILL IN: utility provider] | Outage report / restoration estimate | [FILL IN] |
| [FILL IN: building management / facilities] | Building power, generator, cooling | [FILL IN] |
| [FILL IN: electrician] | Electrical faults | [FILL IN] |
| [FILL IN: APC/Schneider support] | UPS faults, battery replacement | [FILL IN] |
| [FILL IN: escalation if the IT lead is unavailable] | Authority to start shutdown | [FILL IN] |

### What to say

Keep it short and concrete. Staff need to know: is it down, what should I
do right now, and when will it be back. Do not promise a restoration time
you do not have — "we will update you by [time]" is better than a guess
that turns out wrong.

---

## Operations (Day-2)

Routine work that keeps this runbook true. A runbook nobody rehearses is
a document, not a capability.

### Monthly — UPS health

```
# PowerView display, or the NMC web UI:
#   Status  : load, runtime remaining, battery capacity
#   Diags   : any bad power modules or battery modules
#   Self-test: result and date of the last test
```

- [ ] Run or confirm a **self-test**. [FILL IN: is the automatic
      self-test schedule enabled on the Symmetra? Record its interval.]
- [ ] Record capacity, load, and runtime into
      `Hardware & Backup/UPS Specs/Symmetra LX_RM Template.txt`. Runtime
      trending downward over months is batteries ageing.
- [ ] Note any bad modules. A bad battery module means the runtime
      figure is already wrong.

### Quarterly — verify the notification chain

The clients only matter if they actually receive the event.

```
upsc [FILL IN: UPS name]@[FILL IN: NUT server]   # confirm each client can query the UPS
systemctl status nut-monitor                      # monitoring service running on each client
journalctl -u nut-monitor --since "90 days ago" | grep -i "on battery"   # did it see past events?
```

- [ ] Every host in the UPS client table responds.
- [ ] The client list on the NMC matches the table in this document.
- [ ] Update the table when hosts are added or removed. A new server
      that nobody added to the UPS client list will not shut itself down.

### Annually — battery service and a rehearsal

- [ ] Check battery age against service life. [FILL IN: record the
      install date and the replacement interval for these modules.]
      Budget for replacement before they fail, not after.
- [ ] **Walk the shutdown order on paper** with whoever might have to do
      it. Time it. Update the measured shutdown duration in the Decision
      section — that number drives the whole decision threshold.
- [ ] Reprint the one-page checklist below if anything changed, and
      replace the copy in the server room.

### After any host is added, removed, or renamed

- [ ] Add or remove it in the shutdown and bring-up tiers above.
- [ ] Add or remove it on the UPS client list.
- [ ] Confirm which tier it belongs to — getting a database host filed
      as an application host is how a database gets cut mid-write.

## Troubleshooting

Failure modes during the outage itself. These are the moments when the
plan stops working and judgement is needed.

### Symptom: A host will not shut down — it hangs on "shutting down"

- Likely cause: a filesystem that will not unmount, usually because a
  process still holds files on a network mount that has already gone
  away, or an NFS/SMB mount whose server is unreachable.
- Check: connect to the console (not SSH — SSH may already be gone) and
  read the last messages on screen.
  ```
  # From another host, if the network is still up:
  ssh root@<host> "systemctl list-jobs"     # what is the shutdown waiting on?
  ```
- Fix: give it [FILL IN: grace period, e.g. 5 minutes]. If it is still
  hung and the UPS clock is running, a hard power-off is acceptable for
  a **stateless** host — a web front end, an application server with no
  local state. It is **not** acceptable for a database or the NAS. For
  those, keep waiting, and if you truly cannot wait, record that the
  stop was unclean so the post-restoration checks catch the damage.

### Symptom: UPS runtime is dropping faster than expected

- Likely cause: the reported runtime assumed a lighter load, the
  batteries are degraded, or a module has failed under load.
- Check PowerView: current load, battery capacity, bad module count.
- Fix: **skip ahead in the shutdown order.** Drop the remaining
  workstation and application tiers immediately — force them off if
  necessary — to get to the database and storage tiers while there is
  still power to do it cleanly. The tiers that must be graceful are
  4, 5 and 6. Everything above them can be sacrificed to protect them.

### Symptom: The network died before the shutdown was finished

- Likely cause: network gear was not on the UPS, or was on a circuit
  that dropped first.
- Consequence: no more remote shutdowns. Everything remaining has to be
  done at the console.
- Fix now: work at the rack. Fix later: **[FILL IN: confirm that
  switches, firewall, and the NAS are all actually fed from the
  Symmetra, not from wall power. If any of them is not, that is a
  finding — the shutdown order above assumes the network outlives the
  servers.]**

### Symptom: A host did not come back after power was restored

- Check, in order: is it powered (front panel LED)? Does it POST
  (console)? Does it reach the OS?
- Likely causes: BIOS/UEFI power-restore policy is set to "stay off"
  rather than "last state" or "power on", a failed PSU, or a
  filesystem check blocking the boot.
- Check and fix:
  ```
  # At the console, if it is sitting at an fsck or emergency prompt:
  journalctl -xb                 # why the boot stopped
  # A filesystem needing a manual fsck will say so and give you a shell
  ```
- **[FILL IN: confirm the BIOS/UEFI "power restore" setting on every
  server — it should be set so hosts come back automatically after
  power returns. A host set to "stay off" needs someone physically
  present after every outage.]**

### Symptom: Proxmox cluster will not form quorum after bring-up

- Likely cause: nodes booted at different times and corosync has not
  settled, or a switch port for the ring network is not up yet.
- Check:
  ```
  pvecm status                   # which nodes are visible?
  systemctl status corosync
  timedatectl                    # clock skew after a long outage breaks corosync
  ```
- Fix: bring up the remaining nodes, correct the clocks, then
  `systemctl restart corosync` one node at a time. Details in
  `Linux & Servers/proxmox-cluster-administration-guide.md`.

### Symptom: Database will not start, or reports crash recovery

- Expected if the stop was not clean. Let recovery finish — it can take
  a long time on a large database, and interrupting it makes things
  worse.
- Check the startup log before assuming success:
  ```
  journalctl -u mariadb -n 100          # InnoDB recovery messages
  journalctl -u postgresql -n 100       # expect "database system is ready to accept connections"
  ```
- If recovery fails outright, stop improvising and restore from backup:
  `SysAdmin Procedures/Backup_DR_Runbook.txt` section 4.2, and
  `Retrospect Restore/Retrospect Restore Procedure.md`.

### Symptom: NAS volume is degraded or checking parity after the outage

- A hard power loss can drop a drive from the array.
- Do **not** put full production load on it until you know the state.
  See the degraded-array procedure in
  `Hardware & Backup/synology-nas-administration-guide.md`.
- A parity consistency check after an unclean shutdown is normal and
  will run for hours. The volume is usable but slow. Let it finish.

### Symptom: Everything is up but users report certificate or login errors

- Almost always a **clock** problem, not an expiry or an AD problem.
  A host that booted with a dead RTC battery comes up with a wildly
  wrong date, which breaks TLS validation and Kerberos simultaneously.
- Check `timedatectl` on the affected host and on the NAS, then let NTP
  correct it and retry. Only investigate certificates and AD after the
  clocks are confirmed correct.

---

# ONE-PAGE CHECKLIST — PRINT AND POST IN THE SERVER ROOM

```
================================================================
  POWER OUTAGE QUICK CHECKLIST            ORG IT — rev 2026-09-11
  Full runbook: Hardware & Backup/power-outage-shutdown-runbook.md
  Call: IT lead [FILL IN: phone]
        Escalation:     [FILL IN: name / phone]
================================================================

FIRST 5 MINUTES
  [ ] Check UPS PowerView: on battery? runtime remaining? any faults?
  [ ] Call [FILL IN: utility/facilities] for a restoration estimate
  [ ] Notify [FILL IN: IT lead] — [FILL IN: phone, not email]

DECIDE
  SHUT DOWN NOW if:
    - restoration unknown or > [FILL IN: threshold]
    - runtime < 2x full shutdown time ([FILL IN: measured minutes])
    - any UPS fault / bad module / on bypass
    - cooling is also down
  WHEN IN DOUBT, SHUT DOWN. Never walk away from a UPS on battery.

SHUTDOWN ORDER  (verify each tier before the next)
  1. Workstations ........ shutdown -h now / shutdown /s
                           verify: no ping
  2. App tier ............ stop service, stop docker, shutdown -h now
                           verify: no ping; pct/qm list = stopped
  3. Web tier ............ stop proxy, shutdown -h now
                           verify: site does not answer
  4. DATABASES ........... check no connections; systemctl stop <engine>
                           WAIT for clean stop — do not force
                           verify: is-active = inactive; log shows clean
  5. Hypervisors ......... all guests stopped, then shutdown -h now
                           verify: no ping, front-panel LEDs off
  6. Storage (NAS) ....... DSM -> Shutdown; let it finish
                           verify: no ping, chassis LED off
  7. Network gear ........ SAVE CONFIGS, then power off
                           verify: UPS load near zero

  Record: time completed ________  UPS runtime left ________

BRING-UP ORDER  (gate at each step — do not skip ahead)
  Power stable? UPS on line, no faults, charging? Cooling on?
  1. Network ......... gate: gateway pings, DNS resolves
  2. Storage/NAS ..... gate: volumes HEALTHY, shares reachable
  3. Hypervisors ..... gate: pvecm status Quorate, pvesm all active,
                             timedatectl correct
  4. Databases ....... gate: engine answers; READ THE STARTUP LOG
                             for crash recovery
  5. App tier ........ gate: service stays up, connects to DB
  6. Web tier ........ gate: curl -I returns 200, TLS valid
  7. Workstations .... gate: AD login works, shares map

AFTER
  [ ] dmesg for I/O errors; no filesystem read-only
  [ ] NAS volumes healthy; Snapshot Replication resumed
  [ ] Retrospect backups ran (2:00 AM daily) — rerun if missed
  [ ] Clocks correct everywhere (timedatectl) — wrong clock breaks
      AD and TLS
  [ ] UPS self-test passed, no bad modules
  [ ] Tell staff we are back
  [ ] WRITE DOWN what happened and fix this runbook

DO NOT
  - Do not cut power to a database that has not confirmed it stopped
  - Do not interrupt the NAS mid-shutdown
  - Do not start the app tier before the database is verified
  - Do not power-cycle the Synology because it is "slow to boot"
================================================================
```

---

## References

- `Hardware & Backup/UPS Specs/Symmetra LX_RM Template.txt` — UPS status
  recording template (blank; needs filling in)
- `Hardware & Backup/UPS Specs/ups clients list.png` — 2017 screenshot of
  the UPS client list (stale; re-export from the NMC)
- `Hardware & Backup/UPS Specs/UPS Config and specs.rtfd`
- `Hardware & Backup/synology-nas-administration-guide.md`
- `Linux & Servers/proxmox-cluster-administration-guide.md`
- `Linux & Servers/lxc_backup_restore_proxmox91.txt`
- `SysAdmin Procedures/Backup_DR_Runbook.txt`
- `Retrospect Restore/Retrospect Restore Procedure.md`

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

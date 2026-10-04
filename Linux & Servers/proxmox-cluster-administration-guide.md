> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Proxmox VE Cluster Administration

## Overview

Proxmox VE is the organization's virtualisation platform. It hosts the VMs and LXC
containers that run most internal services — the web tier, application
servers, and database servers behind hosts such as db01.example.com,
db02.example.com, sign.example.com, appdb2.example.com, appdb1.example.com, erpdb.example.com,
prod01.example.com and graylog01.example.com. This guide covers day-to-day cluster
administration: inventory, quorum, node lifecycle, storage, maintenance
windows, and version upgrades.

This document does **not** cover LXC backup and restore. That procedure
is already written up in full and should not be duplicated here — see
`Linux & Servers/lxc_backup_restore_proxmox91.txt` for vzdump, `pct
restore`, same-server revert and cross-server migration.

## Quick Facts

| Field            | Value                                                      |
|------------------|------------------------------------------------------------|
| Environment      | prod                                                       |
| Location         | Server room, [FILL IN: rack and rack unit positions]       |
| Access           | Web UI https://[FILL IN: PVE node FQDN]:8006 ; SSH as root to each node |
| Dependencies     | DNS (example.com), NTP, corosync ring network, [FILL IN: shared storage target], APC Symmetra LX/RM UPS |
| Dependents       | All virtualised services — [FILL IN: confirm which of db01 / db02 / sign / appdb2 / appdb1 / erpdb / prod01 / graylog01 are PVE guests vs. bare metal] |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                    |

## Cluster and Node Inventory

Fill this in from `pvecm nodes` and `pvecm status` on any node. Do not
guess node names — the cluster is the source of truth.

| Node name | Mgmt IP | Corosync ring IP | CPU / RAM | Local storage | Role / notes |
|-----------|---------|------------------|-----------|---------------|--------------|
| [FILL IN: node 1 name] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: node 2 name] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: node 3 name] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |

| Field | Value |
|-------|-------|
| Cluster name | [FILL IN: output of `pvecm status` — "Name:" field] |
| Node count | [FILL IN] |
| Expected votes | [FILL IN] |
| Quorum votes needed | [FILL IN] |
| PVE version in use | [FILL IN: confirm — LXC doc references 9.1; run `pveversion -v`] |
| QDevice / qdisk in use? | [FILL IN: yes/no — matters for 2-node clusters] |

Commands to populate the table:

```
pvecm nodes                    # node IDs, votes, names, membership
pvecm status                   # cluster name, quorum math, ring status
pveversion -v                  # PVE and kernel versions on this node
pvesm status                   # storage IDs, type, active, total/used/avail
```

Run them on **each** node — version drift between nodes is a real risk
and is the usual cause of failed migrations.

## How It Works

A Proxmox cluster is several independent hypervisor nodes sharing one
management plane. Two pieces make that work:

**corosync** carries cluster membership and voting over a dedicated ring
network. Every node votes. The cluster is *quorate* when a majority of
expected votes are present. Corosync is latency-sensitive: it wants a
low-jitter link, ideally its own VLAN or physical NIC, separate from
guest and storage traffic.

**pmxcfs** is the Proxmox cluster filesystem, mounted at `/etc/pve` on
every node. It is a FUSE filesystem backed by a replicated SQLite
database, synchronised over corosync. Everything in `/etc/pve` —
guest configs, storage definitions, user/permission data, firewall
rules, the cluster's own config — is replicated to every node
automatically. **`/etc/pve` becomes read-only when the node loses
quorum.** That single fact explains most "I can't start a VM" and
"I can't edit anything in the web UI" incidents.

Rough layout:

```
Web UI (node:8006, pveproxy) -> pvedaemon -> qemu-server / pct
                                          # guest lifecycle
corosync ring (UDP, own VLAN)             # membership + voting
/etc/pve (pmxcfs, replicated)             # all cluster config
/var/lib/vz or shared storage             # guest disks and backups
```

Key paths:

| Path | Contents |
|------|----------|
| `/etc/pve/corosync.conf` | Cluster membership and ring config (edit with care) |
| `/etc/pve/storage.cfg` | Storage backend definitions, cluster-wide |
| `/etc/pve/qemu-server/<vmid>.conf` | VM configuration |
| `/etc/pve/lxc/<vmid>.conf` | LXC container configuration |
| `/etc/pve/nodes/<node>/` | Per-node config, including its certificates |
| `/var/log/pve/tasks/` | Task logs (what the UI shows in the Task Log pane) |
| `journalctl -u corosync` | Cluster membership events |
| `journalctl -u pve-cluster` | pmxcfs events |

## Quorum

Quorum is the cluster's rule for deciding whether it is safe to act. Each
node normally has one vote. The cluster needs `floor(expected/2) + 1`
votes to be quorate.

**When quorum is lost:**

- `/etc/pve` goes read-only on the minority nodes.
- Guests **already running keep running**. They are not killed. This is
  important and often misunderstood — a quorum loss is not an outage for
  running workloads by itself.
- You cannot start, stop, migrate, or reconfigure guests on a
  non-quorate node.
- HA-managed guests, if HA is configured, will be fenced and restarted
  elsewhere by the HA stack. **[FILL IN: is Proxmox HA enabled on this
  cluster? If yes, list which guests are HA-managed — the fencing
  behaviour below is only relevant if it is.]**

Check quorum:

```
pvecm status                   # look for "Quorate: Yes"
corosync-quorumtool -s         # votes held vs. votes needed
```

**Two-node clusters have no safe majority.** If this cluster has two
nodes, either add a QDevice (a third voting witness that is not a
hypervisor) or accept that losing either node makes the survivor
non-quorate. Do not routinely run `pvecm expected 1` to paper over
this — see the emergency note in Troubleshooting.

## Operations (Day-2)

### Adding a Node

Pre-checks, in order:

1. The new node runs the **same PVE major/minor version** as the
   existing cluster. Mismatched versions are the top cause of join
   failures and later migration failures.
2. Time is synchronised. `timedatectl` on both — corosync will not
   tolerate drift.
3. Forward and reverse DNS resolve for the new node's name and IP, on
   every existing node.
4. The new node has **no guests on it**. Joining wipes its
   `/etc/pve`. Any VM or CT on the new node is lost. This is not
   recoverable — back up and move guests off first.
5. Root SSH from the new node to an existing node works.

On the new node:

```
pvecm add [FILL IN: existing cluster node IP or FQDN]   # joins the cluster; prompts for that node's root password
```

Then, on any node:

```
pvecm nodes                    # new node should be listed
pvecm status                   # expected votes should have increased by 1
```

Post-join: confirm the new node sees shared storage (`pvesm status`),
confirm its web UI is reachable, and add it to the backup schedule and
to the UPS client list — see
`Hardware & Backup/power-outage-shutdown-runbook.md`.

### Removing a Node

Removing a node is **destructive and one-way**. A removed node can never
be rejoined to the same cluster without a full reinstall of PVE on it.

1. Migrate or back up every guest off the node. Confirm with
   `pct list` and `qm list` on that node — both must be empty.
2. Note the node's guests' VMIDs somewhere before you start.
3. Power the node **off**. It must be off, not just idle, before the
   removal command runs.
4. On a remaining node:

```
pvecm nodes                    # confirm the node is offline, note exact name
pvecm delnode [FILL IN: node name to remove]   # removes it from the cluster
pvecm status                   # expected votes should have decreased by 1
```

5. Clean up the stale config directory on a remaining node if it
   lingers:

```
ls /etc/pve/nodes/             # stale node dir may remain
# remove only after confirming the node is gone from pvecm nodes
```

6. Do **not** power the removed node back on while it is still on the
   corosync network. It will try to rejoin and can disrupt the cluster.
   Reinstall it before reconnecting.

### VM vs LXC — Which to Use

| Use a VM (KVM) when | Use an LXC container when |
|---------------------|---------------------------|
| The guest is not Linux (Windows workstations, appliances) | The guest is Linux and shares the host kernel version well |
| You need a different kernel, or kernel modules | You want low overhead and fast boot |
| You need full device passthrough, nested virt, or hardware emulation | You mainly need process and filesystem isolation |
| You need live migration between nodes with no downtime | Downtime of a few seconds on a move is acceptable |
| The workload runs Docker heavily | You accept that Docker-in-LXC needs extra tuning and is fragile across PVE upgrades |
| Vendor support requires a "real machine" | The service is a simple internal daemon |

Practical notes:

- Docker workloads should generally be in a **VM**, not an LXC. Docker
  inside an unprivileged LXC needs nesting and keyctl tweaks and tends
  to break on PVE upgrades. Multi-arch Docker image builds in particular
  want a full VM — see
  `Linux & Servers/Building Multi-Arch Docker Images – Systems Knowledge Base.pdf`.
- LXCs cannot live-migrate. Moving one means a brief stop, so schedule
  it. VMs on shared storage can live-migrate.
- Prefer **unprivileged** LXCs unless a specific mount or capability
  requires otherwise. Note the tradeoff in the container's notes field.
- [FILL IN: record the current VM vs LXC split — which services are VMs,
  which are containers. Get it from `qm list` and `pct list` on each node.]

### Storage Backends

| Storage ID | Type | Backing | Shared? | Used for | Notes |
|------------|------|---------|---------|----------|-------|
| [FILL IN] | [FILL IN: dir/lvmthin/zfs/nfs/cifs/pbs] | [FILL IN] | [FILL IN] | [FILL IN: guest disks / backups / ISOs] | [FILL IN] |
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |

Populate from:

```
pvesm status                   # all storages: type, active, total/used/avail
cat /etc/pve/storage.cfg       # the definitions themselves, including content types
```

Things that matter about storage in a cluster:

- A storage marked **shared** must actually be shared (NFS, CIFS, iSCSI,
  Ceph, PBS). Marking a local directory as shared is a classic
  self-inflicted wound: migrations will "succeed" and the guest will
  then fail to find its disk.
- Live migration of a VM requires its disk to be on shared storage. On
  local storage, PVE will do a storage migration instead — slower, and
  it needs free space on the target.
- Only storages with the `backup` content type can receive vzdump
  archives.

**Where backups land:** [FILL IN: confirm the vzdump target storage ID
and its physical location — local `/var/lib/vz/dump`, an NFS/CIFS share
on files.example.com, or a Proxmox Backup Server. Also confirm the schedule
under Datacenter → Backup.] Retrospect also backs up
[FILL IN: confirm whether Retrospect backs up the PVE hosts themselves,
the guests via in-guest Linux clients, or both].

Backup and restore procedure for containers: see
`Linux & Servers/lxc_backup_restore_proxmox91.txt`. Broader backup and DR
context: `SysAdmin Procedures/Backup_DR_Runbook.txt`.

### Node Maintenance Procedure

Use this for any planned work on a node — updates, reboots, firmware,
hardware, or moving it in the rack. Work on **one node at a time** and
verify quorum before starting the next.

**Step 1 — Pre-checks**

```
pvecm status                   # must read "Quorate: Yes" before you begin
pvesm status                   # all expected storages active
qm list                        # VMs on this node
pct list                       # containers on this node
```

Confirm the cluster can lose this node and stay quorate. On a 3-node
cluster that is fine. On a 2-node cluster it is not — see Quorum.

**Step 2 — Take a backup**

Back up any guest you are about to move, especially containers.
Migration is usually clean; "usually" is doing work in that sentence.
Procedure: `Linux & Servers/lxc_backup_restore_proxmox91.txt`.

**Step 3 — Migrate guests off**

```
ha-manager crm-command node-maintenance enable [FILL IN: node name]   # only if HA is enabled; moves HA guests off
qm migrate <vmid> [FILL IN: target node] --online                     # live-migrate a VM, no downtime
pct shutdown <vmid>                                                   # containers cannot live-migrate — stop first
pct migrate <vmid> [FILL IN: target node]                             # then move it
pct start <vmid>                                                      # start on the target node
```

Verify the node is empty:

```
qm list                        # expect no running guests
pct list                       # expect no running guests
```

**Step 4 — Update**

```
apt update                     # refresh package lists
apt full-upgrade               # PVE requires full-upgrade, not upgrade — 'upgrade' can hold back kernel/PVE packages
pveversion -v                  # record the versions after the upgrade
```

Read the output before rebooting. If it pulled a new kernel or new
`pve-manager`, expect the reboot to matter.

**Step 5 — Reboot**

```
reboot
```

**Step 6 — Verify before returning to service**

```
pvecm status                   # node rejoined, cluster quorate
pvesm status                   # all storages active again — check this, mounts do not always come back
systemctl --failed             # nothing failed at boot
journalctl -p err -b           # boot errors
pveversion -v                  # confirm expected version
```

Check the web UI shows the node green, and that it can see the other
nodes' guests.

**Step 7 — Return to service**

```
ha-manager crm-command node-maintenance disable [FILL IN: node name]   # only if HA is enabled
```

Migrate guests back if you want them on their home node. Then verify
each guest is actually working, not merely running:

```
pct exec <vmid> -- systemctl is-system-running    # 'running' or 'degraded'
qm status <vmid>                                   # 'running'
```

Confirm the service itself responds — check the relevant application
endpoint, not just the hypervisor's opinion of it.

### Cluster / PVE Version Upgrade

A major PVE upgrade (e.g. 8.x → 9.x) is a bigger operation than a node
reboot and should get a scheduled window.

**Pre-upgrade checklist — do not skip any of these:**

- [ ] Read the upstream upgrade notes for the specific version jump.
      Every major PVE upgrade has version-specific gotchas; do not work
      from memory.
- [ ] Every node is currently on the **same** version and fully
      up to date within its current major release.
- [ ] `pvecm status` shows a healthy, quorate cluster with no node
      flapping.
- [ ] A current backup exists for **every** guest, and the backup
      target has free space. Verify a restore recently actually worked —
      an untested restore is a rumour.
- [ ] Backup `/etc/pve` off-cluster:
      `tar czf /root/etc-pve-$(date +%F).tgz /etc/pve` then copy that
      file off the node.
- [ ] Enough free space on `/` and on the storage holding guest disks.
      Upgrades fail badly when root fills.
- [ ] Out-of-band access confirmed working (IPMI/iDRAC/iLO or physical
      access). **[FILL IN: what OOB management do these nodes have, and
      what are its addresses?]** If the upgrade goes wrong you need a
      console, not SSH.
- [ ] Maintenance window communicated. See the communications section of
      `Hardware & Backup/power-outage-shutdown-runbook.md`.
- [ ] Known-good rollback plan: for each node, the plan is restore
      guests from backup onto a reinstalled node. There is no
      "downgrade PVE" path. Accept that before starting.
- [ ] Run `pve[FILL IN: version-specific upgrade checker, e.g. pve8to9]`
      on every node and resolve every warning it reports **before**
      touching the repositories.

**Upgrade sequence:**

Upgrade **one node at a time**, keeping the cluster quorate throughout.

1. Put the node into maintenance and empty it (Node Maintenance steps
   1–3 above).
2. Update the APT repositories to the new release.
   **[FILL IN: which repo does the organization use — pve-no-subscription, pvetest, or
   an enterprise subscription repo? This determines the exact sources
   list entries.]**
3. `apt update` then `apt dist-upgrade`, answering config-file prompts
   deliberately. When asked to keep or replace a config file, keep the
   local version only if you know why it was changed.
4. Reboot.
5. Run the full Step 6 verification. **Do not proceed to the next node
   until this one is verified healthy and quorate.**
6. Repeat for each remaining node.
7. Once every node is upgraded, confirm cross-node migration works
   again by live-migrating one low-risk guest and migrating it back.

Mixed-version clusters are supported only for the duration of the
rolling upgrade. Do not leave the cluster in a mixed state — migrations
between different major versions will fail, which is exactly the
capability you need in an emergency.

### Certificates and Web UI Access

The web UI listens on port 8006 on every node, served by `pveproxy`.
Each node has its own certificate under `/etc/pve/nodes/<node>/`:

| File | Purpose |
|------|---------|
| `pve-ssl.pem` / `pve-ssl.key` | Default self-signed cert, issued by the cluster CA |
| `pveproxy-ssl.pem` / `pveproxy-ssl.key` | Custom cert, if one is installed. Overrides the above |

By default PVE uses its own cluster CA, which browsers do not trust —
hence the certificate warning. **[FILL IN: does the organization install a trusted
certificate on the PVE web UI (wildcard \*.example.com or ACME), or do
admins accept the self-signed warning? If a custom cert is installed,
record where it comes from and its renewal process and expiry date.]**

Useful commands:

```
pvenode cert info                             # shows installed certs and expiry
systemctl restart pveproxy                    # apply a new cert; drops web UI sessions briefly
openssl x509 -enddate -noout -in /etc/pve/nodes/$(hostname)/pve-ssl.pem   # expiry check
```

Access notes:

- Authentication realms: **[FILL IN: are PVE logins local (PAM/PVE
  realm) or AD-backed? If AD, record the realm name and the bind
  account location — not the credentials.]**
- Two-factor: [FILL IN: is TFA enforced for admin accounts?]
- Exposure: the web UI should be internal-only. [FILL IN: confirm port
  8006 is not reachable from outside — check the firewall/VLAN rules.]
- A node's web UI can manage the whole cluster, so any node will do when
  one is down.

### Console Access to Guests

```
qm terminal <vmid>             # serial console to a VM, if configured
pct enter <vmid>               # root shell inside a container, from the host
pct console <vmid>             # container console (exit with Ctrl-a q)
```

`pct enter` is the fastest way into a container whose network is broken.

## Troubleshooting

### Symptom: A node shows red/offline in the web UI, but the node itself is up

- Likely cause: corosync membership lost — ring network problem, MTU
  change, switch port change, or time drift. The node is usually fine;
  the cluster just cannot hear it.
- Check:
  ```
  pvecm status                        # is this node in the member list?
  systemctl status corosync           # running? recently restarted?
  journalctl -u corosync -n 100       # membership churn, "Retransmit" storms
  timedatectl                         # clock sync — drift breaks corosync
  ping -c 4 [FILL IN: other node ring IP]   # ring network reachability
  ```
- Fix: resolve the underlying network or time problem first. Then, on
  the affected node:
  ```
  systemctl restart corosync          # rejoin; safe, does not touch running guests
  systemctl restart pve-cluster       # restarts pmxcfs if /etc/pve is stuck
  ```
- Note: restarting corosync does **not** stop guests. Restarting it on
  several nodes at once, however, can drop quorum — do one node at a
  time.

### Symptom: "cluster not ready - no quorum" / `/etc/pve` is read-only

- Likely cause: this node is in the minority partition. Enough nodes are
  down or unreachable that the majority is lost.
- Check:
  ```
  pvecm status                        # "Quorate: No", and votes held vs. needed
  corosync-quorumtool -s              # same, more detail
  ```
- Fix: bring the missing nodes back. Quorum returns on its own and
  `/etc/pve` becomes writable again. Running guests are unaffected
  throughout.
- **Emergency override only:**
  ```
  pvecm expected 1                    # DANGEROUS: forces this node to consider itself quorate
  ```
  Use this only when you are certain the other nodes are genuinely
  down and will stay down — for example during a controlled shutdown
  where you must still stop guests on the last node. If the other nodes
  are actually alive but partitioned, this creates a split brain and you
  can end up with divergent `/etc/pve` state and, with shared storage,
  two nodes writing the same disk. Undo it by restoring real quorum; the
  setting does not survive a corosync restart.

### Symptom: Migration fails

- Likely causes, in the order worth checking:
  1. **Version mismatch** between source and target node. Check
     `pveversion -v` on both.
  2. **Storage not available on the target** — the guest's storage ID
     does not exist there, or is not active.
  3. **Local disk, live migration requested.** VMs on local storage
     cannot live-migrate without a storage migration; LXCs cannot live-
     migrate at all.
  4. **CPU model mismatch** — a guest pinned to `host` CPU type will not
     live-migrate to different hardware. Use a common model like
     `x86-64-v2-AES` for guests that need to move.
  5. Passthrough devices (PCI, USB) pin a guest to one node.
- Check:
  ```
  pveversion -v                       # on BOTH nodes, compare
  pvesm status                        # on the TARGET node — is the storage active?
  qm config <vmid>                    # look at cpu:, and any hostpci/usb lines
  cat /var/log/pve/tasks/active       # the failing task, then read its log file
  ```
- Fix: correct the mismatch. For an LXC, do it offline:
  ```
  pct shutdown <vmid>                 # clean stop
  pct migrate <vmid> [FILL IN: target node]
  pct start <vmid>                    # verify the service, not just the container
  ```
  If migration cannot be made to work in the time available, fall back
  to backup-and-restore across nodes — see
  `Linux & Servers/lxc_backup_restore_proxmox91.txt` section 4.

### Symptom: Storage full — backups failing, guests pausing, or "no space left on device"

- Likely cause: backup retention not pruning, a runaway guest disk, old
  ISOs, or thin-pool overcommit catching up with you.
- Check:
  ```
  pvesm status                        # per-storage used/avail
  df -h /                             # root filesystem on the node itself
  du -sh /var/lib/vz/dump/*           # backup archives, largest first
  lvs -a                              # thin pool Data% — over ~90% is dangerous
  ```
- Fix:
  ```
  # Remove specific old backup archives only after confirming retention elsewhere:
  ls -lt /var/lib/vz/dump/            # newest first; identify what is genuinely expendable
  ```
  Then fix the cause: set a retention policy on the backup job
  (Datacenter → Backup → Edit → Retention) rather than deleting by hand
  every few months.
- **A full LVM-thin pool can corrupt guest filesystems.** If `Data%` is
  near 100, treat it as an incident: stop non-essential guests to halt
  writes, then free space or extend the pool before restarting them.
- If a guest is paused due to an I/O error after a storage full event,
  free space first, then `qm resume <vmid>`, then check the guest's
  filesystem from inside.

### Symptom: Web UI unreachable on one node

- Likely cause: `pveproxy` down, certificate problem, or the node is
  genuinely offline.
- Check:
  ```
  systemctl status pveproxy           # from SSH on that node
  journalctl -u pveproxy -n 50        # cert parse errors show up here
  ss -lntp | grep 8006                # is anything listening?
  ```
- Fix:
  ```
  systemctl restart pveproxy          # brief UI interruption, no guest impact
  ```
  Meanwhile, manage the cluster from any other node's web UI — they are
  equivalent.

### Symptom: A guest will not start after a host reboot

- Check:
  ```
  qm start <vmid>                     # read the actual error, do not guess
  pct start <vmid>
  pvesm status                        # is the guest's storage active?
  cat /etc/pve/qemu-server/<vmid>.conf
  ```
- Common causes: storage not mounted yet (NFS/CIFS mount race at boot),
  a passthrough device absent, or the guest was never set to start on
  boot in the first place. For the mount race, confirm the storage is
  active then start the guest manually.

## Security

- Exposure: internal only. Web UI on 8006, SSH on 22, corosync on its
  own ring network. [FILL IN: confirm the management VLAN and which
  subnets can reach 8006 and 22.]
- Auth: [FILL IN: realms in use — PAM/PVE local, and/or AD. Record where
  admin accounts live.]
- Certificates: see Certificates and Web UI Access above — [FILL IN].
- Secrets: root passwords and any AD bind credential live in the
  password manager, never in this document. [FILL IN: name the password
  manager/vault entry.]
- Root SSH between nodes is required by PVE for migration; that trust is
  by design. Keep the nodes' SSH exposure limited to the management
  network.

## Monitoring & Alerting

- [FILL IN: is graylog01.example.com receiving PVE syslog? If so, name the
  stream and any alert rules.]
- Backup job notifications: Datacenter → Backup → Edit → Notification.
  [FILL IN: confirm the notification target address and whether
  "on failure only" or "always" is set.]
- What normal looks like: all nodes green in the UI, `pvecm status`
  quorate, every storage active in `pvesm status`, no entries in
  `systemctl --failed`.
- [FILL IN: record baseline figures — typical node CPU/RAM utilisation
  and storage used%, so deviations are recognisable.]

## Disaster Recovery

- RTO/RPO: inherit from `SysAdmin Procedures/Backup_DR_Runbook.txt`
  per service class. [FILL IN: confirm whether the hypervisor layer
  itself has its own RTO target distinct from the guests.]
- Loss of one node: migrate or restore its guests to a surviving node
  from backup. This is the routine case and is the reason the backup
  schedule exists.
- Loss of the whole cluster: rebuild PVE on available hardware, recreate
  storage definitions from the `/etc/pve` backup tarball, then restore
  guests from vzdump archives —
  `Linux & Servers/lxc_backup_restore_proxmox91.txt` section 4 covers
  the restore mechanics onto a different host.
- Power events: `Hardware & Backup/power-outage-shutdown-runbook.md`.
- Escalation if the owner is unavailable: [FILL IN: name and contact].

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why / Ticket |
|------------|-------------------|--------------|
| 2026-09-11 | Initial draft of this guide | Cluster administration was undocumented; only LXC backup/restore existed |
| [FILL IN]  | [FILL IN: when was the cluster built, and on what hardware?] | [FILL IN] |

## References

- `Linux & Servers/lxc_backup_restore_proxmox91.txt` — LXC backup,
  restore, same-server revert, cross-server migration. **Authoritative;
  not duplicated here.**
- `Linux & Servers/docker-migration-procedure.md`
- `Linux & Servers/Building Multi-Arch Docker Images – Systems Knowledge Base.pdf`
- `Linux & Servers/Formatting and Mounting Disks in Linux – Systems Knowledge Base.pdf`
- `SysAdmin Procedures/Backup_DR_Runbook.txt`
- `Hardware & Backup/power-outage-shutdown-runbook.md`
- `Hardware & Backup/synology-nas-administration-guide.md`
- Upstream: Proxmox VE Administration Guide, and the version-specific
  upgrade wiki page for the exact release jump being performed.

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

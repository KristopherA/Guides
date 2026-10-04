> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Parallels Shared Folders and Drive Mapping

<!-- Tier 2 (standard): Core + Operations + Troubleshooting. -->

## Overview

Several staff run a Windows 11 guest under Parallels Desktop on a
Mac, rather than having a separate Windows workstation. Those Windows
guests usually need access to files that live on the Mac side — the
user's Desktop and Documents, or a folder holding a mounted share from
files.example.com. Parallels handles this with **shared folders**, which
appear inside Windows as a network location.

The problem this document solves: Windows applications, especially older
line-of-business software, frequently insist on a **drive letter**
(`Z:\`) rather than a UNC path. This guide covers sharing a Mac folder
into the guest, assigning it a persistent drive letter, and the ways that
mapping breaks.

## Quick Facts

| Field            | Value                                                       |
|------------------|-------------------------------------------------------------|
| Owner            | IT lead (it@example.com)                          |
| Environment      | prod — user workstations                                    |
| Location         | Per-workstation; Mac hosts running Parallels Desktop with Windows 11 guests |
| Access           | Physical or remote access to the Mac host and the Windows guest |
| Dependencies     | Parallels Desktop, Parallels Tools installed in the guest, macOS Full Disk Access grant for Parallels, files.example.com for any share being re-shared |
| Dependents       | Windows applications that require a drive letter — [FILL IN: which applications specifically? Dynamics GP / DMS are likely candidates — confirm] |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                     |

| Field | Value |
|-------|-------|
| Parallels Desktop version in use | [FILL IN] |
| Windows guest version | Windows 11, [FILL IN: build/feature release] |
| Machines using this setup | [FILL IN: list the Macs and users] |
| Standard drive letter convention | [FILL IN: agree one — e.g. Z: for the Mac home folder — and use it everywhere] |

## How It Works

Parallels Tools (installed inside the Windows guest) runs a **Shared
Folders** service that presents selected macOS folders to Windows through
a virtual network redirector. Windows sees them under a UNC-style path:

```
macOS folder  ->  Parallels Tools shared-folder service  ->  Windows guest
                                                             \\Mac\Home
                                                             \\Mac\<share name>
                                                             (also under "Mac" in This PC)
```

Two things follow from that architecture and explain most of the trouble:

1. **Shared folders depend on Parallels Tools.** If Tools is missing,
   out of date after a Parallels or Windows upgrade, or its service is
   not running, the shares vanish from Windows entirely.
2. **The share is not a real network drive.** It is provided by the
   virtualisation layer, so it only exists while the VM is running under
   Parallels, and Windows services running as SYSTEM or as another user
   may not see a mapping made in an interactive user session.

If the macOS folder being shared is itself a mounted SMB share from
files.example.com, there is a further dependency: the Mac must be connected to
that share for the guest to see anything inside it. This double hop is
fragile — see Troubleshooting.

## Operations

### Step 1 — Enable and configure shared folders (macOS side)

With the virtual machine **shut down** (not just suspended):

1. Parallels Desktop → **Actions → Configure…** (or the gear icon).
2. **Options → Sharing → Share Mac**.
3. Set **Share folders with Windows** to the appropriate scope:
   - *Home folder only* — the user's home directory. The usual choice.
   - *All disks* — broader; only if genuinely needed.
   - *None* — sharing off.
4. Click **Custom Folders…** to add a specific folder and give it a
   share name. Use this for anything outside the home folder, including
   a folder containing a mounted files.example.com share.
5. Note the share name exactly — it becomes the UNC path
   `\\Mac\<share name>` and is what the drive mapping will point at.

Notes:

- macOS will prompt to grant Parallels access to Desktop, Documents, and
  removable volumes. **These prompts must be accepted**, or the folders
  appear empty inside Windows with no error. If they were dismissed
  earlier, grant access under System Settings → Privacy & Security →
  Files and Folders (and Full Disk Access) for Parallels Desktop.
- Prefer **Custom Folders with an explicit, stable share name** over
  sharing the whole home folder when a drive letter is involved. A
  stable name is what keeps the mapping from breaking.

### Step 2 — Verify the share appears in Windows

In the Windows guest:

```
dir \\Mac\Home                      # should list the Mac home folder contents
dir \\Mac\[FILL IN: share name]     # should list the custom shared folder
```

Or open **File Explorer → This PC** and look for the **Mac** section. If
nothing is there, stop and fix that before attempting a drive mapping —
see Troubleshooting.

### Step 3 — Map a drive letter

Parallels publishes a knowledge base article on this, which was the
entire content of the original stub and is worth keeping:

> **https://kb.parallels.com/116127** — How to manually assign a drive
> letter to a shared Mac folder or volume

A local copy of that article is also filed alongside this document:
`Windows & Mac Workstations/How to manually assign a drive letter to a shared Mac folder or volume.pdf`

**GUI method** (in the Windows guest):

1. File Explorer → **This PC** → **Map network drive…** (under the
   three-dot menu or the Computer ribbon tab).
2. **Drive:** pick the agreed letter. [FILL IN: standard letter.] Pick
   one late in the alphabet to avoid colliding with USB devices, and use
   the *same* letter on every machine — applications get configured with
   the path, and a different letter per machine is a permanent support
   burden.
3. **Folder:** `\\Mac\[FILL IN: share name]`
4. Tick **Reconnect at sign-in**. This is what makes it persist.
5. Leave "Connect using different credentials" unticked — Parallels
   shared folders do not take Windows credentials.
6. Finish. The drive should open immediately.

**Command-line method** (faster, and scriptable across several machines):

```
net use Z: \\Mac\[FILL IN: share name] /persistent:yes    # map and remember across reboots
net use                                                    # list current mappings and their state
net use Z: /delete                                         # remove the mapping
```

### Step 4 — Make it persist across reboots

`/persistent:yes` (or "Reconnect at sign-in") is usually enough, but two
extra considerations matter here:

- **Timing.** Windows restores mapped drives early in the sign-in
  process, sometimes before the Parallels Tools shared-folder service is
  ready. The drive then shows a red X until it is touched. Opening it
  once normally reconnects it. If this is a recurring complaint on a
  given machine, a logon script that re-runs `net use` after a short
  delay is the pragmatic fix:
  ```
  # Scheduled Task, trigger "At log on", delayed 30 seconds, action:
  cmd /c timeout /t 30 && net use Z: \\Mac\[FILL IN: share name] /persistent:yes
  ```
- **Elevated and service contexts.** A drive mapped in the user's normal
  session is **not** visible to a process running as administrator or as
  a service. If a line-of-business application runs elevated and cannot
  see `Z:`, this is why. Point that application at the UNC path
  `\\Mac\<share name>` instead of the drive letter, or map the drive
  again from an elevated prompt.

### Step 5 — Verify

```
net use                             # Z: should show status OK
dir Z:\                             # contents list
echo test > Z:\_writetest.txt       # confirm write access, not just read
del Z:\_writetest.txt               # clean up
```

- [ ] Drive letter present and correct.
- [ ] Read **and write** both work — a share can mount read-only and
      look fine until someone tries to save.
- [ ] Still mapped after a full guest reboot.
- [ ] The application that needed the drive letter can actually see it,
      including when run elevated.

## Troubleshooting

### Symptom: No `\\Mac` shares appear in the Windows guest at all

- Likely cause: Parallels Tools not installed, out of date, or its
  service is not running. This is by far the most common cause, and
  typically follows a Parallels update or a Windows feature update.
- Check, in the guest:
  ```
  sc query prl_tools                  # Parallels Tools service state — expect RUNNING
  dir \\Mac\Home                       # expect a listing, not "network path not found"
  ```
  Also check Parallels Desktop's menu bar — it reports when Tools needs
  reinstalling.
- Fix: Parallels Desktop → **Actions → Install/Update Parallels Tools**,
  then reboot the guest. After any Windows feature update, reinstalling
  Tools should be a standard post-upgrade step — see
  `Windows & Mac Workstations/windows-feature-update-procedure.md`.

### Symptom: Shared folder appears but is empty, or access is denied

- Likely cause: macOS privacy permissions. Parallels has not been
  granted access to Desktop/Documents/removable volumes, so it shares a
  folder it cannot itself read.
- Check: macOS → System Settings → Privacy & Security → **Files and
  Folders**, and **Full Disk Access**, and look at the Parallels Desktop
  entries.
- Fix: grant access, then restart the VM. Granting Full Disk Access to
  Parallels Desktop resolves the whole class of these at once.
- Also check the underlying macOS folder permissions — if the Mac user
  cannot read the folder in Finder, the guest certainly will not.

### Symptom: Drive letter shows a red X, or "the network path was not found" after reboot

- Likely cause: the mapping was restored before the Parallels Tools
  service was ready.
- Check:
  ```
  net use                             # status column will read "Unavailable" or "Disconnected"
  ```
- Fix: open the drive once — it usually reconnects on access. For a
  recurring case, use the delayed logon task in Step 4. Or remap:
  ```
  net use Z: /delete
  net use Z: \\Mac\[FILL IN: share name] /persistent:yes
  ```

### Symptom: The application cannot see `Z:` but File Explorer can

- Likely cause: the application runs elevated or as a service, and
  mapped drives do not cross session/elevation boundaries.
- Fix: configure the application with the UNC path `\\Mac\<share name>`
  rather than the drive letter, or map the drive again from an elevated
  command prompt so the elevated session has its own mapping.

### Symptom: The share points at a files.example.com folder and is empty or stale

- Likely cause: the **Mac** has lost its connection to the SMB share.
  The guest can only see what the Mac currently has mounted, so a
  disconnected share on the Mac becomes an empty folder in Windows.
- Check on the Mac:
  ```
  mount | grep smbfs                  # is the files.example.com share actually mounted?
  ping -c 2 files.example.com              # is the NAS reachable?
  ```
- Fix: reconnect on the Mac (Finder → Go → Connect to Server →
  `smb://files.example.com`), then re-open the drive in Windows.
- **Better long-term fix:** where possible, map the Windows guest
  **directly** to `\\files.example.com\<share>` over the network instead of
  going Mac → Parallels → Windows. It removes a whole layer of failure,
  and the guest authenticates to AD on its own. Use the Parallels shared
  folder route only for folders that genuinely live on the Mac.
  [FILL IN: decide per application which approach applies, and record
  it.] For NAS-side issues see
  `Hardware & Backup/synology-nas-administration-guide.md`.

### Symptom: Drive letter conflicts with a USB device or another mapping

- Fix: standardise on a high letter (X:, Y:, Z:) and apply it
  consistently across all machines. Change the conflicting device's
  letter in Disk Management rather than moving the share mapping, since
  applications are configured against the share's letter.

### Symptom: Poor performance on large files over the shared folder

- Expected: Parallels shared folders add overhead, and are noticeably
  slower than a native drive — more so for many small files.
- Fix: for bulk work, copy the files into the guest, work locally, and
  copy back. Or map directly to `\\files.example.com\<share>` and skip the
  Parallels layer entirely.

## References

- https://kb.parallels.com/116127 — Parallels KB: how to manually assign
  a drive letter to a shared Mac folder or volume (from the original
  `_ARCHIVE/superseded-stubs/Parallels Share mapping for windows.txt` (archived 2026-09-11) stub)
- `Windows & Mac Workstations/How to manually assign a drive letter to a shared Mac folder or volume.pdf`
  — local copy of the above
- `Windows & Mac Workstations/windows-feature-update-procedure.md` —
  reinstall Parallels Tools after a feature update
- `Hardware & Backup/synology-nas-administration-guide.md` — the NAS
  behind files.example.com

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

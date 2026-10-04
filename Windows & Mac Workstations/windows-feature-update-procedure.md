> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Windows Feature Update Procedure

<!-- Tier 2 (standard): Core + Operations + Troubleshooting. -->

## Overview

Windows 11 feature releases (22H2 → 23H2 → 24H2 → 25H2) go end of
servicing on a schedule, and a workstation left behind stops receiving
security updates. Microsoft's phased rollout means a machine may sit on
an old release for months without ever being offered the new one, so
moving a workstation forward is usually a deliberate act rather than
something that happens on its own.

This procedure covers moving the organization's workstations between Windows feature
releases: what to check first, the supported in-place upgrade, the
registry/Group Policy method for machines that will not offer the
update, the ARM/Parallels special case, and what to verify afterwards.

## Quick Facts

| Field            | Value                                                       |
|------------------|-------------------------------------------------------------|
| Owner            | IT lead (it@example.com)                          |
| Environment      | prod — user workstations                                    |
| Location         | Windows 11 workstations, including Windows guests under Parallels on Macs |
| Access           | Local admin on the workstation; [FILL IN: is there an RMM/remote tool, or is this hands-on?] |
| Dependencies     | Windows Update / [FILL IN: WSUS or Intune in use? This changes whether the registry method is appropriate], AD domain example.com, network access to Microsoft update servers |
| Dependents       | Line-of-business apps on the workstation — [FILL IN: which apps need compatibility confirmation before a feature update? Dynamics GP / DMS are candidates] |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                     |

| Field | Value |
|-------|-------|
| Target feature release | [FILL IN: what release is the organization standardising on right now?] |
| Current fleet spread | [FILL IN: how many machines on each release? Get this before planning a rollout] |
| Update management | [FILL IN: Windows Update for Business, WSUS, Intune, or unmanaged?] |
| ARM machines (Parallels on Apple Silicon) | [FILL IN: which machines?] |

## How It Works

A feature update is a full OS replacement performed in place. Windows
Setup stages the new build, migrates apps, settings, and user data, and
reboots into the new version. The old installation is retained under
`C:\Windows.old` for a limited rollback window.

Three delivery paths, in order of preference:

```
1. Windows Update           # offered when Microsoft's rollout reaches this machine
2. Registry / Group Policy  # tells Windows Update to target a specific release now
3. Installation Assistant / Media Creation Tool   # direct, bypasses the rollout entirely
```

Microsoft gates the rollout on telemetry-driven compatibility
("safeguard holds"). A machine with a known-bad driver or application
genuinely may not be offered the update, and the registry method
overrides that gate. **That is a deliberate decision, not a formality** —
see the warning under Method 2.

## Operations

### Pre-Flight Checks

Do these before starting, on every machine. A feature update on an
unprepared workstation is how an afternoon becomes two days.

- [ ] **Back up.** Confirm the user's data is backed up and, for a
      Parallels guest, take a **snapshot of the VM** first — that is the
      cleanest rollback available. See
      `SysAdmin Procedures/Backup_DR_Runbook.txt`.
- [ ] **Record the current state:**
      ```
      winver                                 # current version and build, in a dialog
      systeminfo | findstr /B /C:"OS Name" /C:"OS Version"
      Get-ComputerInfo | Select-Object WindowsProductName,WindowsVersion,OsBuildNumber
      ```
- [ ] **Free disk space** — at least 25–30 GB free on `C:`. This is the
      most common blocker:
      ```
      Get-PSDrive C | Select-Object Used,Free
      cleanmgr /sageset:1                    # configure, then run cleanmgr /sagerun:1
      ```
- [ ] **Hardware requirements** for Windows 11 are met: TPM 2.0, Secure
      Boot, supported CPU:
      ```
      Get-Tpm                                # TpmPresent and TpmReady must be True
      Confirm-SecureBootUEFI                 # expect True
      ```
      For Parallels guests, TPM is provided virtually — confirm the VM
      config has a TPM chip before assuming.
- [ ] **Application compatibility confirmed** for anything
      business-critical on that machine. [FILL IN: list the apps that
      must be checked, and against what — vendor support statements for
      the target release.]
- [ ] **Pending updates installed and the machine rebooted.** A pending
      reboot will block Setup.
- [ ] **Drivers current**, especially storage, graphics, and network.
      Old drivers are the usual cause of a safeguard hold.
- [ ] **Disconnect non-essential peripherals** — external drives, docks,
      and USB devices cause avoidable Setup failures.
- [ ] **Time budget:** allow 1–2 hours per machine, longer on slower
      hardware. Schedule it outside the user's working time.
- [ ] **AC power connected** on laptops.

### Method 1 — Standard In-Place Upgrade (preferred)

If Windows Update offers the feature update, take it. This is the
supported path and the one Microsoft has decided is safe for that
specific machine.

```
# Settings -> Windows Update -> Check for updates
#   A feature update appears as a separate, optional "Download and install" item
```

Or force a check:

```
usoclient StartScan                        # trigger a Windows Update scan
Get-WindowsUpdateLog                        # generate a readable update log on the Desktop
```

The machine downloads, stages, and prompts for a restart. The long part
is after the reboot; do not interrupt it.

If no feature update is offered, do not assume the machine is broken —
it is usually just the phased rollout. Move to Method 2.

### Method 2 — Registry / Group Policy Target Release

This tells Windows Update to target a specific feature release
immediately, rather than waiting to be offered it. It is the method
captured in the original stub, whose links are kept below.

> **Fix Windows not upgrading from 22H2** (original note)
>
> - https://www.windowslatest.com/2024/10/15/get-windows-11-24h2-quickly-and-skip-microsofts-wait-with-registry-group-policy-editor/
> - https://www.howtogeek.com/how-to-get-the-windows-11-24h2-update/

The same approach is recorded for 25H2 in
`_ARCHIVE/superseded-stubs/Windows 25h2 registy key update.txt` (archived 2026-09-11).

**Before using this:** a machine not being offered the update may be
under a **safeguard hold** for a real compatibility reason. Check
Windows Update for a message about a known issue first. Overriding a
genuine hold can produce a broken driver or a non-working peripheral
after the upgrade. Use this method when the machine is simply waiting
its turn in the rollout — not to force past a stated incompatibility.

### Registry method

Back up the registry key before changing it.

```
:: Path:
HKEY_LOCAL_MACHINE\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate

:: Values to create:
::   TargetReleaseVersion       DWORD   = 1
::   TargetReleaseVersionInfo   String  = the target release, e.g. 24H2 or 25H2
::   ProductVersion             String  = Windows 11
```

PowerShell, run as administrator:

```powershell
$k = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
New-Item -Path $k -Force                                                    # create the key if absent
Set-ItemProperty -Path $k -Name "TargetReleaseVersion"     -Value 1 -Type DWord
Set-ItemProperty -Path $k -Name "TargetReleaseVersionInfo" -Value "[FILL IN: target release, e.g. 25H2]" -Type String
Set-ItemProperty -Path $k -Name "ProductVersion"           -Value "Windows 11" -Type String
gpupdate /force                                                             # apply policy immediately
usoclient StartScan                                                         # trigger an update scan
```

Then reboot and check Settings → Windows Update. The targeted release
should now be offered.

Note that the original 25H2 note describes creating only
`TargetReleaseVersionInfo`. Setting `TargetReleaseVersion = 1` alongside
it is what actually enables the policy; without it the string value is
ignored. Set all three.

### Group Policy method (equivalent, and preferable on domain machines)

```
gpedit.msc
  Computer Configuration
    -> Administrative Templates
      -> Windows Components
        -> Windows Update
          -> Manage updates offered from Windows Update
            -> "Select the target Feature Update version"
               Enabled
               Product version:  Windows 11
               Target version:   [FILL IN: e.g. 25H2]
```

On domain-joined machines, prefer setting this in a **domain GPO** so it
applies consistently and can be removed centrally afterwards.
[FILL IN: does the organization have a GPO for Windows Update targeting? If so, name
it — changing it there is better than touching individual registries.]

### Afterwards — remove the pin

This setting **pins** the machine to that release. Leave it in place and
the workstation will never move forward again, which is the same problem
you just solved. Once the fleet is on the target release, either update
the target value or remove it:

```powershell
$k = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate"
Remove-ItemProperty -Path $k -Name "TargetReleaseVersion","TargetReleaseVersionInfo","ProductVersion" -ErrorAction SilentlyContinue
gpupdate /force
```

[FILL IN: record which machines currently have a target pin set, and at
what value. An undocumented pin is a machine that silently stops getting
feature updates.]

### Method 2b — Installation Assistant / Media Creation Tool

If the registry method does not produce an offer, run the upgrade
directly. This bypasses Windows Update entirely.

- **Windows 11 Installation Assistant** — download from Microsoft,
  run on the machine, follow the prompts. Simplest for a single
  workstation.
- **Media Creation Tool** — creates ISO or USB media; better for
  several machines. Run `setup.exe` from the mounted ISO and choose
  **Keep personal files and apps**.

```
:: From mounted ISO or USB, unattended-ish upgrade keeping data and apps:
setup.exe /auto upgrade /dynamicupdate enable /eula accept
```

Always confirm "Keep personal files and apps" is selected before
proceeding. The other options wipe user data.

### Method 3 — ARM / Parallels Special Case (Apple Silicon Macs)

Windows 11 on ARM, running as a Parallels guest on an Apple Silicon Mac,
does **not** follow the x64 path. The standard Installation Assistant and
the normal Media Creation Tool download x64 media, which will not install
on an ARM guest.

**The organization holds the required files already**, in:

`Windows & Mac Workstations/WIN11 Arm upgrade procedure/`

| File | Notes |
|------|-------|
| `MediaCreationTool.exe` | [FILL IN: confirm which release and architecture this build of the tool targets — verify before use on an ARM guest] |
| `windows11.0-kb5027397-arm64_bacb74fba9077a5b7ae2f74a3ebb0b506f9708f3.msu` | ARM64 enablement package (`KB5027397`). Enablement packages flip a machine to the next feature release by switching on features already staged by cumulative updates — a fast upgrade with a short reboot rather than a full Setup pass |

**[FILL IN: this folder has no written procedure alongside the files —
only the two binaries. Document the exact steps that were used, the
source release and target release this MSU moves a guest between, and
whether the MediaCreationTool.exe there is actually part of the ARM path
or a leftover from an x64 attempt. Until that is confirmed, do not run
these files on a user's machine.]**

### Procedure for an ARM Parallels guest

1. **Snapshot the VM first.** Parallels → Actions → Take Snapshot. This
   is the rollback, and it is better than anything Windows offers.
2. Confirm the guest is genuinely ARM64:
   ```powershell
   (Get-CimInstance Win32_Processor).Architecture      # 12 = ARM64
   $env:PROCESSOR_ARCHITECTURE                          # expect ARM64
   ```
3. Confirm the guest's current build and that the prerequisite
   cumulative update for the enablement package is installed:
   ```powershell
   winver
   Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10
   ```
   An enablement package **will refuse to install** if its prerequisite
   servicing-stack and cumulative updates are not present. Run Windows
   Update fully first.
4. Try Windows Update and the registry target method first — on ARM they
   often work, and the supported path is preferable.
5. Only if those fail, apply the ARM64 enablement package from the
   folder above:
   ```
   wusa.exe "[FILL IN: full path]\windows11.0-kb5027397-arm64_....msu" /quiet /norestart
   ```
   Then reboot. The upgrade completes in a short reboot rather than a
   long Setup sequence.
6. **Reinstall Parallels Tools** after the upgrade — Parallels Desktop →
   Actions → Install/Update Parallels Tools. A feature update commonly
   breaks the Tools integration, and with it shared folders and drive
   mappings. See
   `Windows & Mac Workstations/parallels-shared-folders-and-drive-mapping.md`.
7. Update Parallels Desktop itself on the Mac first if it is not
   current — older Parallels versions do not support newer Windows
   builds.

### Post-Upgrade Verification

```powershell
winver                                              # confirm the new version and build
Get-ComputerInfo | Select-Object WindowsVersion,OsBuildNumber
Get-PSDrive C | Select-Object Used,Free             # Windows.old consumes significant space
Get-WinEvent -LogName System -MaxEvents 50 | Where-Object LevelDisplayName -eq 'Error'
```

Check each of these:

- [ ] Version is the intended release (`winver`).
- [ ] Machine is still **domain-joined** and a domain login works:
      ```
      Test-ComputerSecureChannel -Verbose         # expect True
      ```
- [ ] **Mapped drives** reconnect — `net use`. Parallels guests: see the
      shared-folders guide.
- [ ] **Printers** present and a test page prints.
- [ ] **Line-of-business applications** launch and perform one real
      task. Not just "it opens".
- [ ] **Parallels Tools** reinstalled and working (Parallels guests).
- [ ] No error-level events at boot in the System log.
- [ ] Security software present and reporting in.
- [ ] Windows Update runs cleanly afterwards and pulls the latest
      cumulative update.
- [ ] Device Manager shows no devices with warnings.
- [ ] User confirms their environment looks right before you close the
      ticket.

### Rollback Window

Windows keeps the previous installation in `C:\Windows.old` and allows a
rollback for a **limited period — 10 days by default.** After that the
folder is deleted automatically and rollback is no longer possible.

```
# Settings -> System -> Recovery -> "Go back"
#   Only available while C:\Windows.old exists and the window has not expired

DISM /Online /Get-OSUninstallWindow             # how many days remain
DISM /Online /Set-OSUninstallWindow /Value:30   # extend, up to 60 days. Run BEFORE the window expires
```

Practical guidance:

- **Extend the window before the upgrade** on any machine running
  business-critical software. Ten days is not long enough for a
  quarter-end-only workflow to surface a problem.
- **Do not run Disk Cleanup on `C:\Windows.old`** during the rollback
  window. It reclaims a lot of space and permanently removes the ability
  to go back. Wait until the machine is confirmed good.
- For **Parallels guests, the VM snapshot is the real rollback** — it is
  faster, more complete, and not time-limited. Keep it until the machine
  has been in normal use for a while, then delete it to reclaim disk on
  the Mac.
- [FILL IN: agree a standard — e.g. extend to 30 days, and keep
  Parallels snapshots for two weeks — and record it here.]

## Troubleshooting

### Symptom: Feature update is not offered even after the registry change

- Check for a safeguard hold: Settings → Windows Update often names the
  specific known issue. Also check `C:\$WINDOWS.~BT\Sources\Panther\` for
  compatibility reports.
- Confirm the registry values applied: `gpresult /h report.html` and read
  the Windows Update section — a **domain GPO will override a local
  registry edit**, which is a common surprise on domain-joined machines.
- If a hold is genuine, resolve the underlying driver or application
  first rather than forcing past it.
- Otherwise use Method 2b (Installation Assistant / Media Creation Tool).

### Symptom: Upgrade fails and rolls back automatically

- Check the Setup logs, in this order:
  ```
  C:\$WINDOWS.~BT\Sources\Panther\setuperr.log      # errors only — start here
  C:\$WINDOWS.~BT\Sources\Panther\setupact.log      # full action log
  C:\Windows\Panther\                                # logs after a completed upgrade
  ```
- Most common causes: insufficient disk space, an incompatible driver,
  security software blocking Setup, or an attached peripheral.
- Fixes to try: free more space, disconnect everything external, update
  or temporarily remove the offending driver, and temporarily disable
  third-party security software for the upgrade.

### Symptom: Machine is very low on disk after a successful upgrade

- Cause: `C:\Windows.old` — expected, and often 15–25 GB.
- Fix: wait out the rollback window, **then**:
  ```
  cleanmgr                                 # -> "Clean up system files" -> "Previous Windows installation(s)"
  ```
  Do not do this while you might still need to roll back.

### Symptom: Parallels guest broken after the upgrade — no shared folders, poor graphics, no clipboard

- Cause: Parallels Tools no longer matches the new Windows build.
- Fix: Parallels Desktop → Actions → Install/Update Parallels Tools,
  then reboot the guest. Update Parallels Desktop itself first if it is
  behind. Details in
  `Windows & Mac Workstations/parallels-shared-folders-and-drive-mapping.md`.

### Symptom: Domain trust broken after the upgrade

- Check:
  ```
  Test-ComputerSecureChannel -Verbose      # expect True
  ```
- Fix:
  ```
  Test-ComputerSecureChannel -Repair -Credential (Get-Credential)   # needs domain admin credentials
  ```
  Rejoin the domain only if the repair fails.

## References

- `_ARCHIVE/superseded-stubs/Upgrade windows version.txt` (archived 2026-09-11) — original
  stub; its links are carried forward above
- `_ARCHIVE/superseded-stubs/Windows 25h2 registy key update.txt` (archived 2026-09-11) —
  `TargetReleaseVersionInfo` note for 25H2
- `Windows & Mac Workstations/WIN11 Arm upgrade procedure/` —
  MediaCreationTool.exe and the ARM64 KB5027397 enablement package
- `Windows & Mac Workstations/parallels-shared-folders-and-drive-mapping.md`
- `Windows & Mac Workstations/Find Windows hangs in logs.txt`
- `SysAdmin Procedures/Backup_DR_Runbook.txt`
- https://www.windowslatest.com/2024/10/15/get-windows-11-24h2-quickly-and-skip-microsofts-wait-with-registry-group-policy-editor/
- https://www.howtogeek.com/how-to-get-the-windows-11-24h2-update/

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

# Windows 11 Desktop Maintenance & Troubleshooting Runbook

Reference guide for intermediate/advanced users supporting Windows 11 desktops/laptops — x64 and ARM64. Native tools only. Includes a dedicated section on Windows 11 on Arm devices and running Windows 11 on Apple Silicon Macs via Parallels Desktop.

Conventions: `PS>` = PowerShell, `>` = cmd.exe. Run elevated unless noted.

---

## 1. System Health Check

| Task | GUI |
|---|---|
| OS build/edition/architecture | Settings > System > About, or `winver` |
| Full hardware/software dump | `msinfo32` |
| Reliability/crash history | Reliability Monitor (`perfmon /rel`) |
| Recent problem reports | Settings > System > Troubleshoot > Other troubleshooters; `wercon` |

```powershell
PS> winver                                                        # quick build/version popup
PS> Get-ComputerInfo | select OsName,OsArchitecture,OsBuildNumber,CsSystemType   # CsSystemType shows ARM64-based PC vs x64-based PC
PS> systeminfo                                                     # legacy full dump
PS> (Get-CimInstance Win32_Processor).Architecture                 # 9 = x64, 12 = ARM64
PS> Get-HotFix | sort InstalledOn -Descending | select -First 10   # recent patches
PS> Get-WinEvent -FilterHashtable @{LogName='System';Level=2;StartTime=(Get-Date).AddDays(-1)}   # errors, last 24h
```

First-response checklist: build/architecture → recent updates → System/Application log errors → Task Manager resource check → free disk space.

---

## 2. Windows Update & Patching

GUI: Settings > Windows Update.

```powershell
PS> Get-HotFix | sort InstalledOn                                  # installed update history
PS> Get-WindowsUpdateLog                                            # decode ETW trace to Desktop\WindowsUpdate.log
UsoClient StartScan                                                  # trigger a scan (replaces deprecated wuauclt /detectnow)
UsoClient StartDownload
UsoClient StartInstall
> wusa /uninstall /kb:5001234                                        # remove a specific update
> ms-settings:windowsupdate-action                                    # deep-link straight to the Update page
```

Stuck update / repeated failure: run Settings > Troubleshoot > Other troubleshooters > Windows Update first (native, automated fix for the common stuck-service/corrupt-cache cases), then fall back to section 3 (DISM/SFC) if it still fails.

---

## 3. Component Store & System File Repair

GUI: none — CLI only.

```powershell
> sfc /scannow                                                         # verify + repair protected system files
> DISM /Online /Cleanup-Image /CheckHealth                              # quick corruption check of the component store
> DISM /Online /Cleanup-Image /ScanHealth                                # deeper scan
> DISM /Online /Cleanup-Image /RestoreHealth                              # actually repair (uses Windows Update as source by default)
> DISM /Online /Cleanup-Image /RestoreHealth /Source:D:\sources\install.wim /LimitAccess   # repair from local media if no internet
> DISM /Online /Cleanup-Image /StartComponentCleanup /ResetBase           # trim WinSxS after repairs/updates (frees space, prevents rollback of superseded updates)
```

Run order: `DISM /RestoreHealth` first (fixes the component store SFC pulls from), then `sfc /scannow` (fixes the live files against that store).

---

## 4. Boot & Startup

GUI: Settings > System > Recovery > Advanced startup; hold Shift while clicking Restart to jump straight into WinRE.

```powershell
> bcdedit /enum                                                        # view boot configuration
> bcdedit /set {current} safeboot minimal                               # force Safe Mode next boot
> bcdedit /deletevalue {current} safeboot                                 # clear it, resume normal boot
> shutdown /r /o /t 0                                                       # reboot directly into Advanced Options
> reagentc /info                                                             # check WinRE status/location
> reagentc /enable                                                            # re-enable WinRE if disabled/missing
PS> Get-CimInstance Win32_QuickFixEngineering | sort InstalledOn -Descending | select -First 5   # correlate boot issues with a recent patch
```

Fast Startup (hybrid shutdown/hibernate) is a common source of "changes didn't take effect after reboot" and driver-lock issues on desktops — disable via Control Panel > Power Options > Choose what the power buttons do, if a real full shutdown is needed for troubleshooting.

---

## 5. Performance & Resource Monitoring

GUI: Task Manager (`taskmgr`), Resource Monitor (`resmon`), Performance Monitor (`perfmon`).

```powershell
PS> Get-Process | Sort-Object CPU -Descending | select -First 10                    # top CPU consumers
PS> Get-Process | Sort-Object WS -Descending | select -First 10 Name,WS               # top memory consumers
PS> Get-Counter '\Processor(_Total)\% Processor Time' -SampleInterval 2 -MaxSamples 5
PS> Get-Counter '\Memory\Available MBytes'
PS> Get-CimInstance Win32_LogicalDisk | select DeviceID,@{n='FreeGB';e={[math]::Round($_.FreeSpace/1GB,1)}}
> powercfg /energy                                                                    # 60-sec report on power/perf issues (throttling, driver faults)
> powercfg /batteryreport                                                             # battery health history, laptops
```

Hung app: Task Manager > Details tab > right-click > "Analyze wait chain" (shows what it's blocked on before you kill it).

---

## 6. Networking

GUI: Settings > Network & internet; Network troubleshooter (`ms-settings:network-status` > Network troubleshooter).

```powershell
PS> Get-NetIPConfiguration                                            # modern ipconfig replacement
PS> ipconfig /all
PS> Test-NetConnection -ComputerName 1.1.1.1 -InformationLevel Detailed
> ipconfig /release & ipconfig /renew                                    # force new DHCP lease
> ipconfig /flushdns
> netsh winsock reset                                                    # fix corrupted Winsock catalog (reboot required)
> netsh int ip reset                                                       # reset TCP/IP stack (reboot required)
PS> Get-NetAdapter | ft Name,Status,LinkSpeed
PS> Restart-NetAdapter -Name "Wi-Fi"                                          # quick adapter reset without full driver reinstall
```

Wi-Fi flaky after sleep/wake is one of the most common desktop tickets — `Restart-NetAdapter`, then check for a pending driver update in Device Manager (section 8) before reinstalling drivers.

---

## 7. Drivers & Device Manager

GUI: Device Manager (`devmgmt.msc`).

```powershell
PS> Get-WindowsDriver -Online -All | select Driverclass,OriginalFileName,ProviderName    # inventory installed drivers
> pnputil /enum-devices /connected                                        # list connected devices, native CLI (replaces devcon)
> pnputil /enum-drivers                                                      # list installed third-party driver packages
> pnputil /add-driver C:\drivers\device.inf /install                           # install a driver package
> pnputil /delete-driver oem12.inf /uninstall /force                             # remove a driver package
PS> Get-PnpDevice | Where Status -ne 'OK'                                          # find anything in a problem state
PS> Get-PnpDeviceProperty -InstanceId <id> -KeyName DEVPKEY_Device_ProblemCode      # numeric error code for a flagged device
```

Driver rollback: Device Manager > device > Properties > Driver tab > "Roll Back Driver" (only available if Windows retained the previous package — most reliable native path after a bad update).

---

## 8. Windows 11 UI & Feature Quirks

GUI: Settings app (replaces most Control Panel functions); legacy Control Panel still reachable via `control`.

```powershell
> ms-settings:personalization-start                                    # Start menu layout settings
> ms-settings:taskbar                                                    # taskbar behavior/icons
> ms-settings:privacy-widgets                                              # widgets on/off
> ms-settings:defaultapps                                                    # set default apps/file associations (per-app, not global .exe swaps)
PS> Get-AppxPackage -AllUsers | select Name,PackageFullName                    # inventory of installed Store/UWP apps
PS> Get-AppxPackage *Microsoft.WindowsCalculator* | Remove-AppxPackage           # remove a built-in app for a user
PS> Get-AppxPackage -AllUsers | Foreach {Add-AppxPackage -DisableDevelopmentMode -Register "$($_.InstallLocation)\AppXManifest.xml"}   # re-register/repair Store apps
```

Right-click context menu regressions (old menu buried under "Show more options") and taskbar limitations (no vertical/side placement, limited third-party overflow) are by-design Windows 11 changes, not bugs — worth ruling out before troubleshooting "broken" behavior that's actually intentional.

---

## 9. User Profiles & Sign-in

GUI: Settings > Accounts; `netplwiz` for quick local account management.

```powershell
PS> Get-LocalUser                                                         # list local accounts
PS> New-LocalUser -Name "tempadmin" -NoPassword; Add-LocalGroupMember -Group Administrators -Member "tempadmin"
> control userpasswords2                                                   # legacy account manager, sometimes faster than Settings
PS> Get-WinEvent -LogName Security -FilterHashtable @{Id=4625} -MaxEvents 20     # failed logon attempts
```

Corrupt profile ("temporary profile" banner): back up `C:\Users\<name>`, then create a new local profile and migrate data manually — Windows 11 has no supported one-click profile repair tool.

Skip Microsoft account setup during install: at the "Let's connect you to a network"/account screen, `Shift+F10` for a command prompt, then `oobe\bypassnro` and reboot — offers a local account path again. Useful for kiosk/lab machines and for Parallels VMs where you don't want the VM tied to a personal Microsoft account.

---

## 10. Storage, Backup & Recovery

GUI: Settings > System > Storage (Storage Sense); Settings > System > Recovery; Backup and Restore (Windows 7) applet still present for File History-adjacent workflows.

```powershell
PS> Get-Volume | ft DriveLetter,FileSystemLabel,SizeRemaining,Size
> chkdsk C: /scan                                                          # online scan, no dismount needed
PS> Repair-Volume -DriveLetter C -Scan
PS> Optimize-Volume -DriveLetter C -Defrag -Verbose                          # HDD; SSDs auto-trim instead
PS> Get-BitLockerVolume                                                        # encryption status
> manage-bde -status C:
> systemreset -cleanpc                                                          # "Reset this PC" from the command line
```

Recovery drive: Settings > System > Recovery > Create a recovery drive (native, produces a bootable USB for repair/reset without the recovery partition). For OneDrive-related sync issues (a common source of "files disappeared" tickets on desktops), check Settings > Accounts > Windows backup / OneDrive sync status icon first — most cases are a paused sync, not lost data.

---

## 11. Windows 11 on Arm — Architecture-Specific Issues

Applies to Snapdragon X-series laptops, Surface Pro X/11, and any Arm64 Windows 11 install (including the Parallels VMs in section 12, which are Arm64 by definition on Apple Silicon).

**Emulation model:** Windows 11 on Arm runs x86 and x64 apps through emulation — x86 apps still use the WOW64 layer (with filesystem/registry redirection); x64 apps run without WOW64 via Arm64X binaries and can access the full OS directly. Windows 11 24H2+ uses the newer **Prism** emulator, which is meaningfully faster than the emulation in earlier Windows 11 on Arm releases — if a machine is stuck on an older build, that alone can explain sluggish emulated-app performance.

```powershell
PS> (Get-CimInstance Win32_Processor).Architecture                         # confirm 12 = ARM64 vs 9 = x64
PS> Get-ComputerInfo | select CsSystemType                                  # "ARM64-based PC"
> dxdiag                                                                      # check DirectX level + whether a given app is running under emulation
PS> Get-AppxPackage -AllUsers | select Name,Architecture                       # architecture-tag check for Store apps
```

Known problem areas:

- **Kernel-mode drivers must be native Arm64.** Emulation only covers user-mode code. Corporate VPN clients, many security/antivirus suites, USB license dongles, and older printer/scanner drivers that ship only x86/x64 kernel drivers will fail to install or silently not function — check the vendor's site for an Arm64 build before assuming it's a config issue.
- **Anti-cheat and DRM-heavy games** frequently refuse to run under emulation (kernel-mode anti-cheat isn't supported) or perform poorly even when they do launch.
- **Some installers detect architecture and refuse to run**, even though the app itself would work fine emulated — check for an Arm64-native or "universal" installer from the vendor first.
- **32-bit (x86) kernel drivers cannot run at all** on Arm64 Windows, unlike on x64 Windows where 32-bit drivers are simply unsupported for a different reason (WOW64 doesn't apply to drivers on either architecture, but the failure mode confuses people coming from x64).
- **.NET/Visual C++ runtime mismatches:** install the Arm64 runtime package where available rather than defaulting to x86/x64 — mixing architectures is a common source of "app won't launch, no error" tickets.
- **WSL2 and Docker Desktop** work on Arm64 Windows but with a smaller set of prebuilt Arm64 Linux images; x64-only container images run under QEMU-based emulation inside WSL, which is slow and occasionally fails outright for kernel-dependent workloads.
- **Peripheral driver gaps** are the single most common real-world issue: printers, scanners, webcams, and audio interfaces with old x86/x64-only INF packages simply have no Arm64 driver — check Windows Update and the vendor site; if neither has one, that peripheral isn't supported on this device.

---

## 12. Windows 11 on Apple Silicon via Parallels Desktop

Parallels Desktop on M-series Macs runs **Windows 11 on Arm** as a native Arm64 guest using Apple's Hypervisor.framework — it does not and cannot run x64 Windows (there's no x64-on-Arm-Mac virtualization path analogous to Rosetta at the OS level). Everything in section 11 applies to these VMs on top of the Parallels-specific issues below.

**Setup/install:**

```
Parallels' installation assistant downloads the official Arm64 Windows 11 ISO from Microsoft directly — don't substitute an x64 ISO, it will not boot.
Parallels provisions a virtual TPM 2.0 and UEFI Secure Boot automatically, so Windows 11's TPM/Secure Boot install requirements are satisfied without manual steps.
```

Known problem areas specific to this setup:

- **Double emulation penalty.** An x64 Windows app running under Prism/WOW64 emulation inside an Arm64 Windows VM that's itself virtualized on Apple silicon is about as far from native as it gets — expect noticeably worse performance than the same x64 app would show on a native Arm64 PC. Favor Arm64-native Windows apps inside the VM wherever possible.
- **GPU/graphics limitations.** Parallels presents a paravirtualized display adapter with DirectX support sufficient for desktop use and light 3D, but there's no real GPU passthrough — avoid it for serious gaming or GPU-compute workloads.
- **Kernel-mode drivers fail the same way as on physical Arm64 hardware**, plus Parallels itself doesn't support passing through arbitrary kernel drivers from the host — corporate VPN clients and security suites with kernel components are the most frequent support case here; check the vendor for an Arm64-native, Parallels-aware build.
- **Windows Hello / biometrics don't work** — there's no fingerprint/face-recognition passthrough from macOS to the VM, so sign-in falls back to PIN or password only.
- **USB passthrough is selective.** Storage and most HID devices pass through fine; devices needing a kernel-mode Windows driver run into the same Arm64 driver-availability problem as any physical device.
- **Printing:** if the target printer has no Arm64 Windows driver, route printing through Parallels' shared/virtual printer bridge to the Mac's printer queue instead of trying to install a native driver in the VM.
- **Networking mode matters for troubleshooting:** Parallels defaults to Shared networking (NAT through macOS); switch to Bridged if the VM needs to appear as its own device on the LAN (common requirement for domain-joined machines, RDP-in scenarios, or anything relying on device-based network policy).
- **Parallels Tools** (the guest integration package — coherence mode, shared folders, clipboard sync, dynamic resolution, drag-and-drop) needs to be installed/updated inside the VM separately from Windows Update; a stale Parallels Tools version after a macOS or Parallels Desktop update is a common source of broken clipboard sync, wrong display scaling on Retina displays, or a coherence mode that stops responding.
- **Keyboard mapping:** Mac keyboards lack direct equivalents for some Windows keys (right-click, Delete/Forward-delete, PrtScn); Parallels remaps these by default (e.g., `fn+Backspace` for Delete) but custom keyboard shortcuts and some remote-access tools inside the VM can behave unexpectedly until this is accounted for.
- **Snapshots vs. backup:** Parallels snapshots are convenient for pre-change rollback but are not a substitute for real backup — exclude large VM disk files from Time Machine (they change constantly and bloat backup size/time) and back up important data inside the VM through normal Windows means (File History, OneDrive, etc.) instead.
- **No Boot Camp equivalent.** Apple Silicon Macs cannot dual-boot Windows natively — Parallels (or another Hypervisor.framework-based hypervisor) is the only supported path, which is why every limitation above (no GPU passthrough, no biometrics, Arm64-only) is inherent to the platform rather than a Parallels shortcoming specifically.

---

## 13. Event Viewer Quick Reference for Desktops

GUI: Event Viewer (`eventvwr.msc`) — Custom Views > Administrative Events for a cross-log error feed; the IDs below are the ones actually worth memorizing rather than scrolling through everything.

```powershell
PS> Get-WinEvent -FilterHashtable @{LogName='System';Id=41,6008} -MaxEvents 10          # unexpected shutdowns/crashes
PS> Get-WinEvent -FilterHashtable @{LogName='Application';Id=1000,1001} -MaxEvents 20   # app crashes + WER reports
PS> Get-WinEvent -FilterHashtable @{LogName='System';ProviderName='Microsoft-Windows-WHEA-Logger'} -MaxEvents 10   # hardware (CPU/RAM) errors
```

Crashes / unexpected shutdowns:

| Log | Event ID | Source | Meaning |
|---|---|---|---|
| System | 41 | Kernel-Power | System rebooted without a clean shutdown — classic BSOD or hard-reset indicator, check first for any crash |
| System | 1001 | BugCheck | The actual stop code (0x...) for a bluescreen — pair with Event 41 |
| System | 6008 | EventLog | "The previous system shutdown was unexpected" |
| System | 6005 / 6006 | EventLog | Event Log service started/stopped — brackets a boot/shutdown cycle |
| System | 1074 | User32/Kernel-General | Shutdown/restart was requested, and by what process/user |
| Application | 1000 | Application Error | An app crashed — check the faulting module name, not just the exe |
| Application | 1001 | Windows Error Reporting | Crash report generated for the above — has the bucket ID for searching known issues |

Hardware:

| Log | Event ID | Source | Meaning |
|---|---|---|---|
| System | 17 / 18 / 19 | WHEA-Logger | Corrected/uncorrected hardware error (CPU, memory, PCIe) — recurring entries point to failing RAM or an unstable overclock |
| System | 7 / 11 / 51 | Disk | Bad block / controller / paging error on a physical disk — early warning for drive failure |
| System | 153 | Disk | Device reset — often a flaky SATA/USB connection or failing external drive |
| System | 55 | Ntfs | File system corruption detected — run `chkdsk` (section 10) |

Services & policy:

| Log | Event ID | Source | Meaning |
|---|---|---|---|
| System | 7000 / 7001 | Service Control Manager | A service failed to start / failed due to a dependency |
| System | 7009 / 7011 | Service Control Manager | Service start timed out — common with slow-starting antivirus or VPN services |
| Application | 1058 / 1030 | Group Policy (GroupPolicy) | GPO processing failure — usually SYSVOL/DNS reachability, pairs with `gpresult /h` |
| System | 10016 | DistributedCOM | DCOM permission error — extremely common, usually benign/cosmetic but worth filtering out early so it doesn't distract from a real issue |

Logon/security:

| Log | Event ID | Source | Meaning |
|---|---|---|---|
| Security | 4624 / 4625 | Microsoft-Windows-Security-Auditing | Logon success / failure — check Logon Type (2=interactive, 10=RDP, 3=network) |
| Security | 4720 | Microsoft-Windows-Security-Auditing | Local user account created |

Working an unknown problem: filter System + Application to Critical/Error, sort by time, and look for anything clustered right around when the user says the issue started — Event 41/1001 pairs mean crash, repeated 7/11/51 means failing disk, repeated 7009/7011 means a slow/hanging service is the actual bottleneck behind a "slow boot" complaint.

---

## Quick Symptom Index

| Symptom | First commands/checks |
|---|---|
| Slow/unresponsive desktop | Task Manager, `Get-Counter`, wait chain analysis |
| Update stuck or failing | Update troubleshooter, `Get-WindowsUpdateLog`, then DISM/SFC |
| System files suspect | `DISM /RestoreHealth` then `sfc /scannow` |
| Won't boot normally | `bcdedit /enum`, Advanced Startup, `reagentc /info` |
| Wi-Fi drops after sleep | `Restart-NetAdapter`, check Device Manager for driver update |
| Device shows error/yellow bang | `Get-PnpDevice \| Where Status -ne 'OK'`, driver rollback |
| Corrupt/temporary user profile | Back up profile folder, create new local profile |
| App won't launch on Arm64 device | Check for native Arm64 build; check for kernel-mode driver dependency |
| Peripheral has no driver on Arm64 | Check vendor site + Windows Update for Arm64 INF; if none, unsupported |
| Windows 11 VM in Parallels sluggish | Check if the app is x64 emulated (double translation); confirm Parallels Tools is current |
| VPN/security software fails in Arm64 VM | Vendor almost certainly ships x64-only kernel driver — check for Arm64-native release |
| Clipboard/coherence broken in Parallels | Reinstall/update Parallels Tools inside the guest |
| Fingerprint/face sign-in unavailable in VM | Expected — no biometric passthrough; use PIN/password |
| Random crashes / BSOD | Event Viewer System log for ID 41 + 1001 (BugCheck) pair |
| Disk seems to be failing | Event Viewer System log for ID 7/11/51/153 recurring |
| Slow boot, unclear cause | Event Viewer System log for ID 7009/7011 (hanging service) |

---

*Reference guide — validate steps against your specific build/hardware before running in production. On Arm64 devices and Parallels VMs, always check for a native Arm64 driver/build before assuming an issue is configuration-related.*

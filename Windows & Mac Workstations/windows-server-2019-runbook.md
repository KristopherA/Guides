# Windows Server 2019+ Troubleshooting & Sysadmin Runbook

Reference guide for intermediate/advanced admins. Native tools only (PowerShell, built-in CLI, RSAT/MMC consoles). Applies to Server 2019, 2022, 2025 unless noted.

Conventions: `PS>` = PowerShell, `>` = cmd.exe. Run elevated unless noted. Replace bracketed values.

---

## 1. System Info & General Triage

| Task | GUI |
|---|---|
| Quick hardware/OS summary | `msinfo32` |
| Full server dashboard | Server Manager (`ServerManager.msc`) |
| Reliability/crash history | Reliability Monitor (`perfmon /rel`) |
| Scheduled jobs | Task Scheduler (`taskschd.msc`) |

```powershell
PS> Get-ComputerInfo | select OsName,OsVersion,OsBuildNumber,CsUptime   # OS build + uptime
PS> systeminfo                                                          # legacy full dump, works everywhere
PS> Get-HotFix | sort InstalledOn -Descending | select -First 10        # recent patches
PS> Get-EventLog -LogName System -EntryType Error -Newest 20            # legacy, fast triage
PS> (Get-CimInstance Win32_OperatingSystem).LastBootUpTime               # last boot time
PS> Restart-Computer -Force                                             # remote-safe reboot
```

First-response checklist: uptime → recent patches → System/Application event log errors → running services → disk space → CPU/memory.

---

## 2. Event Logs & Diagnostics

GUI: Event Viewer (`eventvwr.msc`) — Custom Views > Administrative Events is the fastest cross-log error view.

```powershell
PS> Get-WinEvent -LogName System -MaxEvents 50 | Format-Table TimeCreated,Id,LevelDisplayName,Message -Wrap
PS> Get-WinEvent -FilterHashtable @{LogName='System';Level=2;StartTime=(Get-Date).AddHours(-24)}   # errors, last 24h
PS> Get-WinEvent -FilterHashtable @{LogName='Application';ID=1000,1001}     # specific event IDs
PS> Get-EventLog -List                                                      # list classic logs + sizes
PS> wevtutil el                                                             # list ALL logs (incl. Applications and Services Logs)
PS> wevtutil qe System /q:"*[System[(Level=2)]]" /f:text /c:20 /rd:true      # query without loading full PS object model
PS> wevtutil epl System C:\logs\system.evtx                                 # export log
PS> wevtutil cl Application /bu:C:\logs\app_bak.evtx                        # clear log, backup first
PS> Limit-EventLog -LogName Application -MaximumSize 256MB                  # resize log
```

Boot/logon diagnostics: `bootcfg` is gone in modern Server — use `bcdedit /enum` and Reliability Monitor. For slow boot/logon, enable verbose status messages via Group Policy or check `Microsoft-Windows-Diagnostics-Performance/Operational` log.

---

## 3. Networking

GUI: `ncpa.cpl` (adapters), Network & Sharing Center, Windows Admin Center for remote view.

```powershell
PS> Get-NetIPConfiguration                          # modern ipconfig replacement, per-interface
PS> ipconfig /all                                   # classic, still fastest for quick read
PS> Get-NetAdapter | ft Name,Status,LinkSpeed,MacAddress
PS> Get-NetAdapterStatistics -Name "Ethernet0"       # packet/error counters
PS> Test-NetConnection -ComputerName srv01 -Port 443 -InformationLevel Detailed   # TCP port test + route
PS> Test-Connection -ComputerName srv01 -Count 4 -TraceRoute                       # ping + traceroute in one
PS> tracert -d 10.0.0.5                              # classic traceroute, -d skips DNS
PS> pathping 10.0.0.5                                # traceroute + packet loss stats per hop
PS> Get-NetTCPConnection -State Listen | ft LocalAddress,LocalPort,OwningProcess
PS> netstat -anob                                    # ports + owning process/binary (needs elevation)
PS> Get-NetRoute -AddressFamily IPv4                 # routing table
PS> route print                                      # classic routing table view
PS> New-NetFirewallRule -DisplayName "Allow-8080" -Direction Inbound -Protocol TCP -LocalPort 8080 -Action Allow
PS> Get-NetFirewallProfile | ft Name,Enabled          # firewall profile status
PS> netsh advfirewall show allprofiles                # classic firewall check
PS> netsh interface ipv4 show config                  # per-interface IP config, classic
PS> arp -a                                            # ARP cache
PS> Clear-DnsClientCache                              # flush local DNS resolver cache
PS> Get-DnsClientCache                                 # view cached entries
```

Slow/dropped connections: check `Get-NetAdapterStatistics` for errors, `Test-NetConnection -DiagnoseRouting`, and NIC driver via Device Manager (`devmgmt.msc`).

---

## 4. DNS

GUI: DNS Manager (`dnsmgmt.msc`).

```powershell
PS> Resolve-DnsName srv01.contoso.com -Type A         # nslookup replacement
PS> nslookup srv01.contoso.com                        # classic, good for testing alternate DNS server: nslookup name server
PS> Get-DnsServerZone                                 # on a DNS server: list zones
PS> Get-DnsServerResourceRecord -ZoneName contoso.com -RRType A
PS> Add-DnsServerResourceRecordA -Name "web01" -ZoneName contoso.com -IPv4Address 10.0.0.20
PS> Get-DnsServerDiagnostics                          # current diagnostic logging config
Set-DnsServerDiagnostics -All $true                   # enable verbose DNS debug logging (revert after)
PS> Clear-DnsServerCache                              # flush server-side cache
PS> dnscmd /clearcache                                # classic equivalent
PS> Restart-Service DNS                               # restart DNS Server service
```

Common issue: stale/duplicate records from DHCP-registered clients — check with `Get-DnsServerResourceRecord` and scavenging settings (`Get-DnsServerScavenging`).

---

## 5. DHCP

GUI: DHCP console (`dhcpmgmt.msc`).

```powershell
PS> Get-DhcpServerv4Scope                             # list scopes
PS> Get-DhcpServerv4ScopeStatistics -ScopeId 10.0.0.0 # % utilization, free addresses
PS> Get-DhcpServerv4Lease -ScopeId 10.0.0.0            # active leases
PS> Get-DhcpServerv4Reservation -ScopeId 10.0.0.0      # reservations
PS> Add-DhcpServerv4Reservation -ScopeId 10.0.0.0 -IPAddress 10.0.0.50 -ClientId "AA-BB-CC-DD-EE-FF"
PS> Get-DhcpServerv4Statistics                         # server-wide scope stats
Backup-DhcpServer -Path C:\dhcp_backup                 # backup DHCP config/leases
```

Client-side: `ipconfig /release` then `ipconfig /renew` to force new lease; `ipconfig /displaydns` to check resolver cache.

---

## 6. Active Directory & Group Policy

GUI: Active Directory Users and Computers (`dsa.msc`), AD Sites and Services (`dssite.msc`), Group Policy Management Console (`gpmc.msc`), AD Administrative Center (`dsac.exe`).

```powershell
PS> Get-ADDomainController -Filter *                   # list all DCs
PS> Get-ADUser -Identity jsmith -Properties *           # user object detail
PS> Get-ADUser -Filter {Enabled -eq $false}             # find disabled accounts
PS> Unlock-ADAccount -Identity jsmith                   # unlock a locked account
PS> Set-ADAccountPassword -Identity jsmith -Reset -NewPassword (Read-Host -AsSecureString)
PS> Get-ADGroupMember -Identity "Domain Admins"          # group membership
PS> repadmin /replsummary                                # replication health summary across DCs
PS> repadmin /showrepl                                   # detailed replication status/errors
PS> dcdiag /v                                            # full DC health diagnostics
PS> dcdiag /test:dns                                     # DNS-specific DC test
PS> nltest /dsgetdc:contoso.com                          # locate a DC for a domain
PS> nltest /sc_query:contoso.com                         # secure channel status
PS> Test-ComputerSecureChannel -Repair                    # fix broken machine trust relationship
```

Group Policy:

```powershell
PS> gpupdate /force                                      # reapply all policy now
PS> gpresult /r                                           # summary of applied GPOs for current session
PS> gpresult /h C:\gpo_report.html                         # full HTML report (RSOP)
PS> Get-GPO -All | ft DisplayName,GpoStatus,ModificationTime
PS> Get-GPOReport -Guid <guid> -ReportType Html -Path C:\report.html
PS> Backup-GPO -All -Path C:\gpo_backups                   # backup all GPOs
```

DC health check order: `dcdiag /v` → `repadmin /replsummary` → SYSVOL/NTDS event logs → DNS resolution of `_ldap._tcp.dc._msdcs.<domain>`.

---

## 7. Storage & Disks

GUI: Disk Management (`diskmgmt.msc`), Storage Spaces in Server Manager.

```powershell
PS> Get-Disk                                             # list physical disks
PS> Get-Volume                                           # list volumes + health/free space
PS> Get-Partition -DiskNumber 1                          # partitions on a disk
PS> Get-PhysicalDisk | ft FriendlyName,MediaType,HealthStatus,OperationalStatus
PS> Get-Volume | ft DriveLetter,FileSystemLabel,SizeRemaining,Size
Resize-Partition -DriveLetter D -Size 200GB               # grow/shrink a partition
PS> New-Volume -DiskNumber 2 -FriendlyName Data -FileSystem ReFS -DriveLetter E
PS> Repair-Volume -DriveLetter D -Scan                    # chkdsk equivalent, scan only
PS> Repair-Volume -DriveLetter D -SpotFix                 # fix without full offline chkdsk
> chkdsk D: /f /r                                          # classic, requires dismount/reboot for system volume
PS> Optimize-Volume -DriveLetter D -Defrag -Verbose        # defrag (HDD) — SSD auto-uses -ReTrim
PS> Get-StoragePool                                        # Storage Spaces pools
PS> Get-VirtualDisk                                         # Storage Spaces virtual disks
diskpart                                                    # interactive: list disk / select disk / clean / create partition
```

Disk full triage: `Get-ChildItem -Recurse | Sort-Object Length -Descending | Select -First 20` on suspect volume, or use `dirquota` (FSRM) if quotas are configured. Check `Get-Volume` for SizeRemaining vs Size first.

---

## 8. Services & Processes

GUI: Services (`services.msc`), Task Manager (`taskmgr`), Resource Monitor (`resmon`).

```powershell
PS> Get-Service | Where-Object Status -eq 'Stopped' | ft Name,DisplayName,StartType
PS> Get-Service -Name W32Time | Restart-Service -Force
PS> Set-Service -Name Spooler -StartupType Automatic
> sc.exe query spooler                                    # classic service query
> sc.exe config spooler start= demand                     # classic set startup type
> sc.exe qc spooler                                        # show service config (path, dependencies)
PS> Get-Process | Sort-Object CPU -Descending | select -First 10       # top CPU consumers
PS> Get-Process | Sort-Object WS -Descending | select -First 10 Name,WS  # top memory consumers
PS> Stop-Process -Name notepad -Force                      # kill by name
> taskkill /PID 4321 /F                                     # kill by PID, classic
> tasklist /svc                                             # map PIDs to hosted services (svchost)
PS> Get-CimInstance Win32_Process -Filter "Name='w3wp.exe'" | select ProcessId,CommandLine
```

Hung process: check Resource Monitor > CPU tab > "Analyze Wait Chain" (right-click) before killing — shows what it's blocked on.

---

## 9. Performance Monitoring

GUI: Performance Monitor (`perfmon`), Resource Monitor (`resmon`), Task Manager Performance tab.

```powershell
PS> Get-Counter '\Processor(_Total)\% Processor Time' -SampleInterval 2 -MaxSamples 5
PS> Get-Counter -ListSet Memory | select -ExpandProperty Counter        # discover counter names
PS> (Get-Counter '\Memory\Available MBytes').CounterSamples.CookedValue
typeperf "\Processor(_Total)\% Processor Time" -sc 5                   # classic, no PS needed
PS> Get-Counter -Counter '\PhysicalDisk(*)\Avg. Disk sec/Read' -SampleInterval 1 -MaxSamples 3
logman create counter DiskTrace -c "\PhysicalDisk(*)\% Disk Time" -si 5 -o C:\perf\disk.blg   # scheduled data collector
logman start DiskTrace
logman stop DiskTrace
```

Baseline counters to watch: `% Processor Time`, `Available MBytes`, `Avg. Disk sec/Read|Write`, `\Network Interface(*)\Output Queue Length`. Use Data Collector Sets in `perfmon` for scheduled, long-running captures instead of live `Get-Counter`.

---

## 10. Windows Updates & Patching

GUI: Settings > Windows Update (Server 2019+ still has classic Windows Update UI too, via `control update`); WSUS console if used.

```powershell
PS> Get-WindowsUpdateLog                                  # decodes ETW trace to C:\Users\<you>\Desktop\WindowsUpdate.log
> sconfig                                                  # text-menu server config incl. update settings/install now (Server Core friendly)
UsoClient StartScan                                        # trigger update scan (replaces deprecated wuauclt /detectnow)
UsoClient StartDownload
UsoClient StartInstall
PS> Get-HotFix | sort InstalledOn                          # installed update history
PS> wusa /uninstall /kb:5001234                            # remove a specific update
PS> Get-WindowsUpdate                                       # requires PSWindowsUpdate module (not built-in — RSAT/native alt: Get-HotFix)
```

Note: `PSWindowsUpdate` is a community module, not native — stick to `UsoClient`, `Get-HotFix`, and the Settings UI for a pure native-tools workflow.

---

## 11. Backup & Recovery

GUI: Windows Server Backup (`wbadmin.msc`, install via Server Manager feature first).

```powershell
> wbadmin get versions                                     # list available backups
> wbadmin start backup -backupTarget:E: -include:C: -allCritical -quiet
> wbadmin start systemstaterecovery -version:07/01/2026-02:00 -backupTarget:E:
> wbadmin start recovery -version:07/01/2026-02:00 -itemtype:Volume -items:D: -recoveryTarget:D:
PS> Get-WBSummary                                           # PowerShell equivalent status check
PS> vssadmin list shadows                                   # list Volume Shadow Copy snapshots
PS> vssadmin create shadow /for=C:                          # manual VSS snapshot
PS> vssadmin delete shadows /for=C: /oldest                 # prune old snapshots
Checkpoint-Computer -Description "Pre-patch"                # System Restore point (needs feature enabled)
```

Bare-metal/AD recovery: boot to WinRE, `bcdedit /enum` to inspect boot config, or use Directory Services Restore Mode (DSRM) for AD database repair via `ntdsutil`.

---

## 12. Remote Management

GUI: Server Manager (multi-server), Windows Admin Center, Computer Management (`compmgmt.msc /computer:\\srv01`), RSAT tools pointed at remote server.

```powershell
PS> Enter-PSSession -ComputerName srv01                     # interactive remote shell
PS> Invoke-Command -ComputerName srv01,srv02 -ScriptBlock { Get-Service W32Time }
PS> $s = New-PSSession -ComputerName srv01; Invoke-Command -Session $s { hostname }
PS> Enable-PSRemoting -Force                                 # enable WinRM on target (run locally on target)
PS> Test-WSMan srv01                                          # confirm WinRM is reachable
> winrm quickconfig                                           # classic WinRM setup
PS> Get-CimSession                                             # list active CIM sessions
PS> Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine="notepad.exe"} -ComputerName srv01
> psexec \\srv01 cmd                                           # Sysinternals — not native, mentioned for completeness only
```

Cross-domain/workgroup remoting requires TrustedHosts: `Set-Item WSMan:\localhost\Client\TrustedHosts -Value srv01` on the initiating machine.

---

## 13. Certificate Management (AD CS / PKI)

GUI: Certification Authority console (`certsrv.msc`), Certificate Templates (`certtmpl.msc`), local/machine cert store (`certlm.msc`), current-user store (`certmgr.msc`), enterprise PKI health viewer (`pkiview.msc`).

```powershell
PS> Get-ChildItem Cert:\LocalMachine\My | ft Subject,Thumbprint,NotAfter          # inventory certs in machine personal store
PS> Get-ChildItem Cert:\LocalMachine\My -Recurse | Where EnhancedKeyUsageList -match "Server Authentication"
> certutil -CAInfo                                                                 # CA health/config summary (run on the CA)
> certutil -view -restrict "Disposition=20"                                       # list issued certs from CA DB
> certutil -dump cert.cer                                                          # dump a cert file's fields
> certutil -verify -urlfetch cert.cer                                              # validate chain + fetch CRL/AIA live
> certutil -crl                                                                    # force-publish a new CRL from the CA
> certutil -pulse                                                                  # force client-side autoenrollment now
> certreq -new request.inf request.req                                            # generate CSR from an INF template
> certreq -submit request.req cert.cer                                            # submit CSR to CA, get back issued cert
> certreq -accept cert.cer                                                         # install issued cert (matches pending private key)
PS> Get-Certificate -Template WebServer -Url "ldap:" -SubjectName "CN=web01.contoso.com" -CertStoreLocation Cert:\LocalMachine\My   # AD-integrated autoenroll request
PS> Export-Certificate -Cert Cert:\LocalMachine\My\<thumbprint> -FilePath C:\certs\server.cer
PS> Import-Certificate -FilePath C:\certs\root.cer -CertStoreLocation Cert:\LocalMachine\Root
PS> New-SelfSignedCertificate -DnsName "test.contoso.com" -CertStoreLocation Cert:\LocalMachine\My   # lab/test only, not for prod trust chains
PS> Restart-Service CertSvc                                                       # apply CA config changes (e.g., certutil -setreg)
```

Renewal/expiry triage: `Get-ChildItem Cert:\LocalMachine\My -Recurse | where {$_.NotAfter -lt (Get-Date).AddDays(30)}` finds anything expiring soon — check this on DCs, NPS servers, and any TLS endpoints before it becomes an outage. Autoenrollment is controlled by GPO ("Certificate Services Client – Auto-Enrollment"), not a cmdlet; verify with `gpresult /r`.

---

## 14. LDAPS (LDAP over SSL)

LDAPS needs no manual IIS-style binding: a Domain Controller with a valid cert (Server Authentication EKU, Subject/SAN = DC's FQDN, chained to a trusted root) in its machine Personal store will automatically bind it to TCP 636 (and 3269 for Global Catalog over SSL) the next time NTDS reads certs — usually via autoenrollment from the built-in "Domain Controller Authentication" template, no CA admin action per-DC required.

GUI: LDP.exe (primary native tool to test an SSL bind), Certificates - Local Computer (`certlm.msc`) on the DC, Group Policy Management (`gpmc.msc`) for signing requirements, Event Viewer > Directory Service log.

```powershell
PS> Get-ChildItem Cert:\LocalMachine\My | Where EnhancedKeyUsageList -match "Server Authentication"   # candidate cert on the DC
> certutil -store My                                                              # classic listing of the Personal store
PS> Test-NetConnection -ComputerName dc01 -Port 636 -InformationLevel Detailed     # LDAPS reachable
PS> Test-NetConnection -ComputerName dc01 -Port 3269                              # Global Catalog over SSL
PS> Get-WinEvent -LogName "Directory Service" -MaxEvents 20 | where Id -in 1220,1202  # 1220=cert bound OK, 1202=no usable cert found
PS> Restart-Service NTDS -Force                                                   # re-read cert store after issuing/renewing a cert (or reboot)
```

Manual test: open `ldp.exe` > Connection > Connect, enter the DC FQDN, port 636, check "SSL" — a successful bind confirms the cert, chain, and port are all correct end to end. To enforce LDAPS/signing, configure "Domain controller: LDAP server signing requirements" and "Domain controller: LDAP server channel binding token requirements" in the Default Domain Controllers Policy, then verify with `gpresult /h`.

---

## 15. RADIUS (Network Policy Server)

GUI: Network Policy Server console (`nps.msc`); add the role via Server Manager > Add Roles ("Network Policy and Access Services").

```powershell
PS> Install-WindowsFeature NPAS -IncludeManagementTools               # install NPS role + RSAT console
> netsh ras add registeredserver domain=contoso.com server=NPS01       # register NPS in AD (grants read on user dial-in props) — GUI equivalent: right-click NPS root > "Register server in Active Directory"
> netsh nps show client                                                # list configured RADIUS clients (switches/APs/VPN)
> netsh nps add client name="switch01" address="10.0.0.5" sharedsecret="<SHARED_SECRET>" vendor="RADIUS Standard"
> netsh nps show np                                                    # list network policies
> netsh nps export filename="C:\nps\nps-backup.xml" exportPSK=YES      # full config backup, incl. shared secrets
> netsh nps import filename="C:\nps\nps-backup.xml"                    # restore/migrate to a new NPS server
PS> Get-WinEvent -LogName Security -FilterHashtable @{Id=6272,6273} -MaxEvents 20   # 6272=RADIUS auth granted, 6273=denied
PS> Get-NetFirewallRule -DisplayGroup "Network Policy Server" | ft DisplayName,Enabled,Direction
```

Certificates for RADIUS: for PEAP/EAP-TLS, the NPS server needs a cert with Server Authentication EKU — typically from the built-in "RAS and IAS Server" enterprise template (supports autoenrollment, same mechanism as section 13). Bind it inside a Network Policy's Constraints > Authentication Methods > EAP Properties dropdown in `nps.msc` — there is no CLI bind step. For EAP-TLS client auth, endpoints need a client cert (Workstation Authentication/User template) or the issuing CA's root trusted for PEAP server validation.

RADIUS uses UDP 1812/1813, so `Test-NetConnection -Port 1812` only confirms firewall/routing reachability, not a real auth handshake — treat the Security log Event IDs above as the ground truth for whether requests are actually succeeding, and check the shared secret matches exactly on both the NPS client entry and the NAS device.

---

## 16. Registry

GUI: Registry Editor (`regedit`).

```powershell
PS> Get-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters" -Name TcpTimedWaitDelay
PS> New-ItemProperty -Path "HKLM:\SOFTWARE\Policies\X" -Name "Setting" -Value 1 -PropertyType DWord
PS> Set-ItemProperty -Path "HKLM:\SOFTWARE\X" -Name "Setting" -Value 0
> reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion /v ProgramFilesDir
> reg add HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters /v TcpTimedWaitDelay /t REG_DWORD /d 30 /f
> reg export HKLM\SOFTWARE\MyApp C:\backup\myapp.reg           # backup before editing
> reg import C:\backup\myapp.reg                                # restore
```

Always export the key before editing when working outside of documented, supported settings.

---

## 17. Kerberos & Authentication Troubleshooting

GUI: none dedicated — Event Viewer Security log is the primary surface.

```powershell
> klist                                                   # cached Kerberos tickets for current session
> klist tgt                                                # ticket-granting-ticket detail
> klist purge                                               # clear ticket cache, forces re-authentication
> setspn -L svcAccount                                      # list SPNs registered to an account
> setspn -X                                                  # scan domain for duplicate SPNs (common auth failure cause)
> setspn -A HTTP/web01.contoso.com svcAccount                # register a new SPN
PS> nltest /sc_verify:contoso.com                            # verify machine secure channel
PS> Get-WinEvent -LogName Security -FilterHashtable @{Id=4768,4769,4771} -MaxEvents 20   # TGT issued/service ticket/pre-auth failed
```

`KRB_AP_ERR_SKEW` errors mean clock drift beyond the 5-minute default tolerance — check section 18 before digging into SPNs or trusts.

---

## 18. Time Synchronization (w32tm)

GUI: none native — Date and Time (`timedate.cpl`) shows current time only, not sync source.

```powershell
> w32tm /query /status                                       # current sync state, source, offset
> w32tm /query /source                                        # what this machine syncs from
> w32tm /query /peers                                          # configured NTP peers
> w32tm /resync /force                                          # force immediate resync
> w32tm /monitor                                                # check offset across all DCs in the domain
> w32tm /config /manualpeerlist:"time.contoso.com" /syncfromflags:manual /update   # point PDC emulator at an external source
PS> Get-ADDomainController -Filter * | select Name,OperationMasterRoles           # find the PDC emulator (the one DC that should sync externally)
PS> Restart-Service w32time -Force
```

In an AD domain, only the PDC emulator should sync externally — every other DC and member server follows the domain hierarchy automatically. Chasing time drift on non-PDC machines usually means the PDC itself is out of sync.

---

## 19. Boot & Recovery (WinRE / DSRM / AD Database)

GUI: Advanced Startup (Settings > Recovery, or hold Shift while clicking Restart), WinRE boot menu, `msconfig` Boot tab (Safe boot options).

```powershell
> bcdedit /enum                                               # view current boot configuration
> bcdedit /set {current} safeboot minimal                     # force Safe Mode on next boot
> bcdedit /set {current} safeboot dsrepair                     # force DSRM (AD repair mode) on a DC's next boot
> bcdedit /deletevalue {current} safeboot                       # clear safe-boot flag, resume normal boot
> shutdown /r /o /t 0                                             # reboot straight into WinRE / advanced options menu
> reagentc /info                                                   # check WinRE status/location
> reagentc /enable                                                  # re-enable WinRE if it's been disabled
```

AD database maintenance (from DSRM or an elevated prompt on a DC):

```
> ntdsutil "activate instance ntds" "authoritative restore" q q      # authoritative restore of AD objects (interactive, run in DSRM)
> ntdsutil "activate instance ntds" files "integrity" q q             # AD database integrity check
> esentutl /g "C:\Windows\NTDS\ntds.dit"                                # direct integrity check of the AD DB file (service stopped/DSRM)
```

Only boot a DC into DSRM when you specifically need to work on the AD database offline (authoritative restore, offline defrag, corruption repair) — it takes the DC out of the replication topology until reset.

---

## 20. Security & Hardening

GUI: Windows Security app, Local Security Policy (`secpol.msc`), BitLocker Drive Encryption (Control Panel).

```powershell
PS> Get-MpComputerStatus                                      # Defender: AV enabled, definitions age, last scan
PS> Get-MpThreatDetection                                       # recent threat detections
PS> Start-MpScan -ScanType QuickScan
PS> Update-MpSignature                                           # force definition update
> secedit /export /cfg C:\secpol.cfg                              # export current local security policy
> secedit /configure /db secedit.sdb /cfg C:\baseline.inf          # apply a security template baseline
PS> Get-BitLockerVolume                                            # encryption status per volume
> manage-bde -status C:                                             # classic BitLocker status check
> manage-bde -protectors -get C:                                     # view key protectors / recovery key IDs
PS> Get-LocalGroupMember -Group Administrators                       # audit local admins — check this on every member server
> auditpol /get /category:*                                            # current audit policy by category
PS> Get-NetFirewallRule | where Enabled -eq True | measure             # quick count of active firewall rules
```

Baseline hygiene: confirm Defender is current before assuming malware isn't a factor, confirm local Administrators group membership on member servers periodically (privilege creep is common), and export `secedit` state before applying any new baseline so you can roll back.

---

## 21. Failover Clustering

GUI: Failover Cluster Manager (`CluAdmin.msc`).

```powershell
PS> Install-WindowsFeature Failover-Clustering -IncludeManagementTools
PS> Test-Cluster -Node srv01,srv02                              # validation report — always run before build and after major changes
PS> New-Cluster -Name Clus01 -Node srv01,srv02 -StaticAddress 10.0.0.100
PS> Get-ClusterNode                                              # node up/down/paused status
PS> Get-ClusterResource                                           # resource health within roles
PS> Get-ClusterGroup                                               # roles and current owning node
PS> Move-ClusterGroup -Name "Role1" -Node srv02                     # manual failover
PS> Get-ClusterQuorum                                                # current quorum model/witness
PS> Get-ClusterLog -Destination C:\clusterlogs                       # generate the cluster debug log — primary troubleshooting artifact
PS> Suspend-ClusterNode -Name srv01 -Drain                             # pause + drain a node for maintenance
PS> Resume-ClusterNode -Name srv01                                       # bring it back
```

Split-brain / quorum loss is the highest-severity cluster failure — `Get-ClusterQuorum` and the cluster log are the first two things to pull.

---

## 22. IIS & Certificate Bindings

GUI: IIS Manager (`inetmgr`).

```powershell
PS> Import-Module WebAdministration; Get-Website               # list sites (requires IIS management tools)
PS> Get-WebBinding -Name "Default Web Site"
PS> New-WebBinding -Name "Default Web Site" -Protocol https -Port 443 -HostHeader web01.contoso.com -SslFlags 1   # SNI binding
> netsh http show sslcert                                        # list ALL HTTPS cert bindings at the http.sys level (works without IIS)
> netsh http add sslcert ipport=0.0.0.0:443 certhash=<thumbprint> appid="{00000000-0000-0000-0000-000000000000}"
> netsh http delete sslcert ipport=0.0.0.0:443                     # remove a manual binding
> iisreset                                                            # restart the full IIS service stack
> appcmd list sites                                                     # native IIS CLI, no module import needed
> appcmd list apppools /state:Stopped
```

`netsh http show sslcert` is the ground truth for what's actually bound at the OS level — useful when IIS Manager shows a binding but connections still fail (stale/duplicate http.sys entries are a common cause).

---

## 23. Task Scheduler

GUI: Task Scheduler (`taskschd.msc`).

```powershell
> schtasks /query /fo LIST /v                                     # detailed list of all tasks
> schtasks /query /tn "\MyTask"                                     # query one task
> schtasks /create /tn "NightlyBackup" /tr "C:\scripts\backup.ps1" /sc daily /st 02:00 /ru SYSTEM
> schtasks /run /tn "NightlyBackup"                                    # trigger immediately
> schtasks /end /tn "NightlyBackup"                                      # stop a running instance
> schtasks /change /tn "NightlyBackup" /disable
> schtasks /delete /tn "NightlyBackup" /f
PS> Get-ScheduledTask | Get-ScheduledTaskInfo                             # PowerShell equivalent, richer object output incl. LastTaskResult
```

---

## Quick Command Index

| Symptom | First commands |
|---|---|
| Server slow/unresponsive | `Get-Counter`, `resmon`, `Get-Process \| sort CPU -desc` |
| No network / DNS | `Get-NetIPConfiguration`, `Test-NetConnection`, `Resolve-DnsName`, `Clear-DnsClientCache` |
| Disk full or failing | `Get-Volume`, `Get-PhysicalDisk`, `Repair-Volume -Scan` |
| AD logon/replication issues | `dcdiag /v`, `repadmin /replsummary`, `Test-ComputerSecureChannel` |
| GPO not applying | `gpresult /h`, `gpupdate /force`, check `Get-GPO -All` status |
| Service won't start | `Get-Service`, `sc.exe qc`, check dependent services, Event Viewer System log |
| Patch/update failure | `Get-WindowsUpdateLog`, `Get-HotFix`, `UsoClient StartScan` |
| Need to restore data | `wbadmin get versions`, `vssadmin list shadows` |
| Remote box unreachable via PS | `Test-WSMan`, `winrm quickconfig`, check firewall/TrustedHosts |
| Cert expired/expiring | `Get-ChildItem Cert:\LocalMachine\My`, `certutil -verify`, check `pkiview.msc` |
| LDAPS bind failing | `Test-NetConnection -Port 636`, `ldp.exe` SSL bind, Directory Service log Event 1202/1220 |
| RADIUS auth failing | Security log Event 6272/6273, `netsh nps show client`, verify shared secret + NPS cert |
| Kerberos auth errors | `klist`, `setspn -X`, Security log Event 4768/4769/4771, check `w32tm /query /status` for skew |
| Clock drift / KRB_AP_ERR_SKEW | `w32tm /query /status`, `w32tm /monitor`, confirm PDC emulator's external source |
| Server won't boot normally | `bcdedit /enum`, boot to WinRE, `reagentc /info` |
| AD database corruption | `ntdsutil` integrity check, `esentutl /g`, boot DC to DSRM first |
| Malware / AV concern | `Get-MpComputerStatus`, `Get-MpThreatDetection`, `Start-MpScan` |
| Drive encryption status | `Get-BitLockerVolume`, `manage-bde -status` |
| Cluster role/node down | `Get-ClusterNode`, `Get-ClusterGroup`, `Get-ClusterQuorum`, `Get-ClusterLog` |
| HTTPS/cert binding broken | `netsh http show sslcert`, `Get-WebBinding`, `iisreset` |
| Scheduled task not running | `schtasks /query /fo LIST /v`, `Get-ScheduledTaskInfo`, check "Last Run Result" |

---

*Reference guide — verify command syntax against your specific build (`Get-Help <cmdlet> -Online`) before running in production. Test destructive operations (chkdsk /f, reg edits, service config changes) in a maintenance window.*

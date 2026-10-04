# Active Directory Administration & Security Guide
### Windows Server 2019 and 2022

A practical reference for managing Active Directory Domain Services (AD DS): GUI and command-line tools, common system administration and cyber security problems, and how to use logging to find and diagnose them.

---

## Table of Contents

1. [Core Concepts & Architecture](#1-core-concepts--architecture)
2. [GUI Management Tools](#2-gui-management-tools)
3. [Command-Line & PowerShell Tools](#3-command-line--powershell-tools)
4. [Common System Administration Problems](#4-common-system-administration-problems)
5. [Common Cyber Security Problems](#5-common-cyber-security-problems)
6. [Logging: Finding Issues in the Event Logs](#6-logging-finding-issues-in-the-event-logs)
7. [Key Event ID Reference Tables](#7-key-event-id-reference-tables)
8. [Detection & Investigation Workflows](#8-detection--investigation-workflows)
9. [Active Directory Certificate Services (AD CS) Attacks](#9-active-directory-certificate-services-ad-cs-attacks)
10. [Windows LAPS — Local Admin Password Management](#10-windows-laps--local-admin-password-management)
11. [LDAP Signing & Channel Binding Hardening](#11-ldap-signing--channel-binding-hardening)
12. [Trusts & Multi-Domain/Forest Security](#12-trusts--multi-domainforest-security)
13. [Read-Only Domain Controllers (RODCs)](#13-read-only-domain-controllers-rodcs)
14. [Hybrid Identity — Entra Connect & Entra ID](#14-hybrid-identity--entra-connect--entra-id)
15. [Fine-Grained Password Policies & gMSAs](#15-fine-grained-password-policies--gmsas)
16. [Assessment & Detection Tooling](#16-assessment--detection-tooling)
17. [DNS Security](#17-dns-security)
18. [Backup, DC Recovery & Forest Recovery Runbook](#18-backup-dc-recovery--forest-recovery-runbook)
19. [Legacy Credential Exposure & SIEM Queries](#19-legacy-credential-exposure--siem-queries)
20. [Hardening Checklist](#20-hardening-checklist)

---

## 1. Core Concepts & Architecture

Active Directory Domain Services (AD DS) is the identity and authentication backbone for a Windows environment. Understanding the moving parts makes both administration and incident response far easier.

**Logical structure.** A *forest* is the top-level security boundary. Inside it are one or more *domains* (e.g., `corp.example.com`), which contain *organizational units* (OUs) used to group objects and target Group Policy. Objects are users, computers, groups, and service accounts.

**Domain controllers (DCs).** Every DC holds a writable copy of the domain database (`NTDS.dit`) and runs the services that make AD work: the Kerberos Key Distribution Center (KDC), LDAP, and DNS (usually). DCs replicate changes to each other using multi-master replication.

**FSMO roles.** Five Flexible Single Master Operation roles are held by specific DCs: Schema Master and Domain Naming Master (forest-wide), plus PDC Emulator, RID Master, and Infrastructure Master (per-domain). The PDC Emulator is the most operationally sensitive — it is the time source for the domain and the authority for password changes and account lockouts.

**Authentication.** Kerberos is the default protocol (ticket-based); NTLM is the legacy fallback. Most modern attacks target the way tickets and hashes are issued, cached, and reused.

**Group Policy.** Group Policy Objects (GPOs) are linked to sites, domains, or OUs and push configuration and security settings to users and computers. `SYSVOL` (a replicated share on every DC) stores GPO files and logon scripts.

What changed in 2019 → 2022: Server 2022 adds Secured-core server features, stronger default TLS (TLS 1.3), SMB encryption improvements (AES-256, SMB over QUIC), and hardware-rooted security options. The AD DS management surface itself is essentially the same, so the tools and event IDs below apply to both versions unless noted.

---

## 2. GUI Management Tools

These are installed with the AD DS role or the Remote Server Administration Tools (RSAT). On a management workstation, add them with:

```powershell
Add-WindowsCapability -Online -Name Rsat.ActiveDirectory.DS-LDS.Tools~~~~0.0.1.0
```

**Active Directory Users and Computers (`dsa.msc`)** — the workhorse. Create/modify/delete users, groups, computers, and OUs; reset passwords; unlock accounts; move objects; delegate control. Enable *View → Advanced Features* to see the Security and Attribute Editor tabs.

**Active Directory Administrative Center (`dsac.exe`)** — a modern console that surfaces the Recycle Bin, fine-grained password policies, and a PowerShell History pane that shows the cmdlet behind every action you click (great for learning).

**Active Directory Sites and Services (`dssite.msc`)** — manage replication topology, sites, subnets, and site links. Use it to force replication and confirm subnet-to-site mappings.

**Active Directory Domains and Trusts (`domain.msc`)** — manage trust relationships between domains/forests and raise domain/forest functional levels.

**Group Policy Management Console (`gpmc.msc`)** — create, link, edit, back up, and model GPOs; run Group Policy Modeling and Results (RSoP) to see what actually applies to a user or computer.

**DNS Manager (`dnsmgmt.msc`)** — manage the DNS zones AD depends on. Broken DNS is the single most common root cause of AD problems.

**Event Viewer (`eventvwr.msc`)** — read the logs described in section 6. Custom Views let you filter by event ID across logs.

**Server Manager / Windows Admin Center** — role installation, health dashboards, and increasingly the browser-based path (Windows Admin Center) for remote management.

---

## 3. Command-Line & PowerShell Tools

The GUI is fine for one-off tasks; the command line is how you do bulk work, automation, and investigation.

### PowerShell (ActiveDirectory module)

Import once per session (auto-loads on modern systems): `Import-Module ActiveDirectory`

```powershell
# Find a user and view attributes
Get-ADUser jdoe -Properties LastLogonDate,LockedOut,PasswordLastSet

# Create a user
New-ADUser -Name "Jane Doe" -SamAccountName jdoe -AccountPassword (Read-Host -AsSecureString) -Enabled $true

# Reset a password and force change at next logon
Set-ADAccountPassword jdoe -Reset -NewPassword (Read-Host -AsSecureString); Set-ADUser jdoe -ChangePasswordAtLogon $true

# Unlock an account
Unlock-ADAccount jdoe

# Find all locked-out accounts
Search-ADAccount -LockedOut

# Find stale computer accounts (no logon in 90 days)
Search-ADAccount -ComputersOnly -AccountInactive -TimeSpan 90.00:00:00

# Group membership
Add-ADGroupMember "IT Admins" -Members jdoe
Get-ADGroupMember "Domain Admins"

# Report all members of privileged groups (recursively)
Get-ADGroupMember "Domain Admins" -Recursive | Get-ADUser -Properties Enabled
```

### repadmin — replication

```cmd
repadmin /replsummary          :: quick health summary across all DCs
repadmin /showrepl             :: detailed inbound replication status for this DC
repadmin /replicate DC2 DC1 "DC=corp,DC=example,DC=com"   :: force a specific replication
repadmin /queue                :: pending replication operations
```

### dcdiag — domain controller health

```cmd
dcdiag /v                      :: verbose full test suite
dcdiag /test:dns /v            :: focused DNS diagnostics
dcdiag /test:replications      :: replication-specific checks
```

### Other essential CLI tools

```cmd
gpupdate /force                :: reapply Group Policy immediately
gpresult /h report.html        :: RSoP report for the current machine/user
nltest /dsgetdc:corp.example.com   :: locate a DC / test the secure channel
nltest /sc_query:corp.example.com  :: verify this member's secure channel
netdom query fsmo              :: show which DC holds each FSMO role
w32tm /query /status           :: time sync status (critical for Kerberos)
klist                          :: list cached Kerberos tickets
klist purge                    :: clear the ticket cache
ntdsutil                       :: authoritative restore, FSMO seizure, DB maintenance
ldp.exe                        :: raw LDAP query/browse (GUI but scriptable via LDAP)
dsquery / dsget / dsmod        :: legacy directory query/modify utilities
csvde / ldifde                 :: bulk import/export of directory objects
```

---

## 4. Common System Administration Problems

### 4.1 Replication failures

Symptoms: password changes or new objects not appearing on all DCs, `dcdiag` failures, event log errors in the Directory Service log.

Diagnose: `repadmin /replsummary` shows the largest replication deltas and failure counts; `repadmin /showrepl` details the failing partner and error code. Common causes are DNS misconfiguration, a DC being offline longer than the tombstone lifetime (default 180 days — such a DC must never be brought back), and lingering objects.

Fix: correct DNS first, then force replication with `repadmin /replicate`. For lingering objects use `repadmin /removelingeringobjects`. Look for **event 1988** (lingering object detected) and **2042** (DC has not replicated in tombstone lifetime).

### 4.2 Time synchronization drift

Kerberos rejects tickets when the clock skew exceeds five minutes (default). Symptoms are widespread logon failures and event **KRB_AP_ERR_SKEW**.

The PDC Emulator should sync from an authoritative external NTP source; all other DCs and members sync down the hierarchy. Check with `w32tm /query /status` and `w32tm /monitor`. Configure the PDC:

```cmd
w32tm /config /manualpeerlist:"time.nist.gov,0x8" /syncfromflags:manual /reliable:yes /update
```

### 4.3 DNS problems

Because clients locate DCs via DNS SRV records, broken DNS breaks nearly everything. Run `dcdiag /test:dns`. Confirm each DC points its own DNS client primarily at another DC (or itself) and that `_ldap._tcp.dc._msdcs.<domain>` SRV records resolve. Missing SRV records: restart the Netlogon service to re-register them, or run `nltest /dsregdns`.

### 4.4 Group Policy not applying

Use `gpresult /r` or `/h` and GPMC's Group Policy Results to see what applied and what was filtered. Common causes: security filtering excludes the object, WMI filter doesn't match, the GPO is linked to the wrong OU, block-inheritance/enforced conflicts, or `SYSVOL` replication is broken. Confirm SYSVOL health (DFS-R) and force with `gpupdate /force`.

### 4.5 Account lockouts

A single user locked out repeatedly usually has a stale credential — a mapped drive, cached mobile-mail password, scheduled task, or service running under old credentials. The PDC Emulator records the authoritative lockout. Find the source by querying **event 4740** (account locked) on the PDC and **4771/4625** to find the originating machine (the Source Network Address / Caller Computer Name field).

### 4.6 FSMO role holder failure

If a role holder is permanently lost, seize the role with `ntdsutil` (roles → connections → seize). Only seize when the original will never return; then remove the dead DC's metadata (`ntdsutil metadata cleanup`).

### 4.7 SYSVOL / DFS-R issues

Modern domains replicate SYSVOL with DFS-R. Symptoms of failure are inconsistent GPOs between DCs. Check the DFS Replication event log and run `dfsrdiag ReplicationState`. A DFS-R database in a "journal wrap" or dirty state may need a non-authoritative sync (D2) or, for a single source of truth, an authoritative sync (D4).

### 4.8 Disk, backup, and recovery

Keep system-state backups of at least one DC. Test restores. The AD Recycle Bin (enable once, irreversibly) lets you restore deleted objects with attributes intact:

```powershell
Enable-ADOptionalFeature 'Recycle Bin Feature' -Scope ForestOrConfigurationSet -Target 'corp.example.com'
Get-ADObject -Filter 'Deleted -eq $true' -IncludeDeletedObjects | Restore-ADObject
```

---

## 5. Common Cyber Security Problems

Active Directory is the number-one target in most enterprise breaches because compromising it grants control of every account and machine. The following are the attacks defenders must understand.

### 5.1 Password attacks: spraying and brute force

*Password spraying* tries one or a few common passwords against many accounts to avoid lockouts; *brute force* hammers one account. Both generate a burst of **4625** (failed logon) and, when Kerberos is used, **4771** (pre-auth failed) events. Enforce strong passwords, MFA where possible, and lockout thresholds via fine-grained password policies.

### 5.2 Kerberoasting

Any authenticated user can request a service ticket (TGS) for an account that has a Service Principal Name (SPN). The ticket is encrypted with the service account's password hash, which the attacker cracks offline. Watch for **4769** (TGS requested) with RC4 encryption (`0x17`) for many SPNs from one account. Defenses: use long random passwords or Group Managed Service Accounts (gMSAs) for service accounts, and disable RC4 in favor of AES.

### 5.3 AS-REP roasting

Accounts with "do not require Kerberos pre-authentication" set can have their AS-REP requested and cracked offline. Audit for that flag:

```powershell
Get-ADUser -Filter 'DoesNotRequirePreAuth -eq $true' -Properties DoesNotRequirePreAuth
```

### 5.4 Pass-the-Hash and Pass-the-Ticket

Attackers steal NTLM hashes or Kerberos tickets from a compromised host's memory (e.g., with Mimikatz) and reuse them to authenticate as the victim without knowing the password. Signs include lateral movement logons (**4624** type 3/9) from unexpected hosts and unusual ticket activity. Defenses: Credential Guard, LSASS protection (RunAsPPL), restricting where privileged accounts log on, and the Protected Users group.

### 5.5 Golden and Silver Tickets

A *Golden Ticket* is a forged TGT signed with the KRBTGT account hash — total forest compromise, valid until KRBTGT is rotated twice. A *Silver Ticket* forges a TGS for a single service. Detection is hard by design; look for tickets with anomalous lifetimes and TGS use without a preceding TGT request. Rotate the KRBTGT password twice (with delay) periodically and after any suspected DC compromise.

### 5.6 DCSync

An attacker with replication rights (`Replicating Directory Changes All`) can ask a DC to hand over password hashes as if it were another DC — including KRBTGT. Watch for **4662** events referencing the replication GUIDs from a non-DC account, and audit who holds replication rights.

### 5.7 Privilege escalation and delegation abuse

Misconfigured *unconstrained delegation*, *resource-based constrained delegation*, dangerous ACLs (e.g., `GenericAll` on a privileged object), and nested privileged group membership let attackers escalate. Tools like BloodHound map these attack paths. Audit delegation:

```powershell
Get-ADComputer -Filter {TrustedForDelegation -eq $true} -Properties TrustedForDelegation
```

### 5.8 Persistence

After gaining control, attackers plant persistence: adding accounts to privileged groups, AdminSDHolder abuse, malicious GPOs, skeleton keys, and DSRM account abuse. Monitor privileged group membership changes (**4728/4732/4756**) and GPO changes.

### 5.9 Legacy protocol exposure

NTLMv1, LM hashes, unsigned LDAP, and SMBv1 are frequently abused. Disable SMBv1, require LDAP signing/channel binding, and phase out NTLM in favor of Kerberos. Server 2022 improves defaults here but existing domains often carry legacy settings forward.

---

## 6. Logging: Finding Issues in the Event Logs

Effective detection depends on (1) turning the right auditing on, (2) knowing which log holds what, and (3) centralizing logs so an attacker can't just wipe a single DC.

### 6.1 Where the logs live

On a DC, Event Viewer shows several relevant logs:

- **Security** — authentication, authorization, and audit events (the primary security log).
- **Directory Service** — AD DS database, replication, and LDAP issues.
- **DNS Server** — name resolution problems.
- **System** — service, Netlogon, time, and hardware events.
- **DFS Replication** — SYSVOL replication health.
- **Applications and Services Logs → Microsoft → Windows → \*** — granular operational logs (Group Policy, PowerShell Operational, etc.).

### 6.2 Turn on the right auditing

Default auditing is minimal. Use *Advanced Audit Policy Configuration* (under Computer Configuration → Policies → Windows Settings → Security Settings) via GPO, not the legacy basic audit policy. Priorities for DCs:

- Account Logon → **Audit Kerberos Authentication Service** and **Kerberos Service Ticket Operations** (Success/Failure)
- Logon/Logoff → **Audit Logon** (Success/Failure)
- Account Management → **Audit User/Security Group/Computer Account Management** (Success)
- DS Access → **Audit Directory Service Changes** (Success)
- Detailed Tracking → **Audit Process Creation** (and enable command-line logging)

Enable command-line capture so **4688** records the full command line:

```
Computer Configuration → Administrative Templates → System → Audit Process Creation →
"Include command line in process creation events" = Enabled
```

Enable PowerShell logging (Script Block + Module) to catch attacker tooling:

```
Administrative Templates → Windows Components → Windows PowerShell →
"Turn on PowerShell Script Block Logging" = Enabled   (writes event 4104)
```

### 6.3 Increase log size and centralize

Default Security log size is far too small on a busy DC. Raise it (GPO or `wevtutil sl Security /ms:1073741824` for 1 GB) so events aren't overwritten before you can review them. Better still, ship events off-box: Windows Event Forwarding (WEF) to a collector, or an agent to a SIEM (Sentinel, Splunk, Elastic). Centralization defeats the common attacker step of clearing local logs (which itself generates **1102**).

### 6.4 Querying logs

Filter in Event Viewer via Custom Views, or from the command line:

```powershell
# Failed logons in the last day
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddDays(-1)}

# Who locked out an account (run on the PDC Emulator)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4740} |
  Select-Object TimeCreated, @{n='Account';e={$_.Properties[0].Value}}, @{n='Source';e={$_.Properties[1].Value}}

# Kerberos service ticket requests using weak RC4 (possible Kerberoasting)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4769} |
  Where-Object { $_.Message -match '0x17' }
```

```cmd
:: Legacy but handy for quick filters and remote queries
wevtutil qe Security /q:"*[System[(EventID=4625)]]" /c:20 /rd:true /f:text
```

---

## 7. Key Event ID Reference Tables

### 7.1 Authentication & logon (Security log)

| Event ID | Meaning | Why it matters |
|---|---|---|
| 4624 | Successful logon | Logon type reveals interactive (2), network (3), RDP (10). Watch for privileged logons on unexpected hosts. |
| 4625 | Failed logon | Bursts across many accounts = password spray; many on one = brute force. |
| 4634 / 4647 | Logoff / user-initiated logoff | Session correlation. |
| 4648 | Logon with explicit credentials | "Run as" / credential use — common in lateral movement. |
| 4768 | Kerberos TGT requested (AS-REQ) | Baseline of authentication; failures with pre-auth off = AS-REP roasting. |
| 4769 | Kerberos service ticket requested (TGS) | Many with RC4 (`0x17`) from one account = Kerberoasting. |
| 4771 | Kerberos pre-authentication failed | Failed password attempts via Kerberos; failure code 0x18 = bad password. |
| 4776 | NTLM credential validation | NTLM auth attempts; spikes may indicate legacy/attack traffic. |
| 4740 | Account locked out | Recorded on the PDC Emulator; source field points to the offending host. |

### 7.2 Account & group management (Security log)

| Event ID | Meaning |
|---|---|
| 4720 | User account created |
| 4722 / 4725 | Account enabled / disabled |
| 4723 / 4724 | Password change / password reset |
| 4726 | User account deleted |
| 4728 / 4729 | Member added/removed — global security group |
| 4732 / 4733 | Member added/removed — local security group |
| 4756 / 4757 | Member added/removed — universal security group |
| 4738 | User account changed |
| 4781 | Account name changed |

Additions to **Domain Admins**, **Enterprise Admins**, **Administrators**, or **Schema Admins** (4728/4732/4756) are among the highest-value alerts you can build.

### 7.3 Directory, delegation & suspicious activity

| Event ID | Meaning / detection use |
|---|---|
| 4662 | Operation on an AD object — with the replication GUIDs, indicates possible **DCSync** from a non-DC. |
| 5136 | Directory service object modified (requires DS Changes auditing). |
| 4670 | Permissions on an object changed — watch privileged objects / AdminSDHolder. |
| 4698 / 4699 | Scheduled task created/deleted — common persistence. |
| 4697 | Service installed — persistence / lateral movement. |
| 1102 | Security log cleared — classic anti-forensics; alert always. |
| 4104 | PowerShell script block logged — inspect for encoded/obfuscated commands. |
| 4688 | Process created (with command line) — hunt for suspicious tooling. |
| 7045 | New service installed (System log). |

### 7.4 Health & replication (Directory Service / System logs)

| Event ID | Log | Meaning |
|---|---|---|
| 1988 | Directory Service | Lingering object detected — replication integrity problem. |
| 2042 | Directory Service | DC has not replicated within tombstone lifetime. |
| 1311 | Directory Service | KCC could not build a replication topology (site/link issue). |
| 2013 | DFS Replication | SYSVOL / DFS-R replication issue. |
| 5774 / 5775 | System (Netlogon) | DNS SRV record registration failure. |
| 129 / 36 | System (W32Time) | Time service source/skew problems. |

---

## 8. Detection & Investigation Workflows

### 8.1 "An account keeps getting locked out"

1. On the PDC Emulator, find **4740** for the account; read the *Caller Computer Name* / *Source Network Address*.
2. On that source host, examine **4625** and **4771** to see which process or logon type is presenting bad credentials.
3. Typical culprits: cached credentials in Credential Manager, a service or scheduled task under old creds, or a mobile device with a stale mail password.

### 8.2 "Possible password spray"

1. In the Security log, aggregate **4625** and **4771** by time and count distinct target accounts.
2. A short window with many distinct accounts each failing once or twice = spray. Note the source IP (Network Information → Source Address).
3. Cross-check **4624** for any *successful* logon from that source afterward — that account may be compromised. Reset it, review its activity, and block the source.

### 8.3 "Kerberoasting hunt"

1. Query **4769** where the ticket encryption type is `0x17` (RC4).
2. Flag accounts requesting many distinct SPNs in a short time, especially from a single workstation.
3. Review which accounts have SPNs and weak passwords; migrate them to gMSAs and disable RC4.

### 8.4 "Did someone run DCSync?"

1. Look for **4662** events whose properties reference the replication control access rights (`DS-Replication-Get-Changes` GUID `1131f6aa-...` / `-All` `1131f6ad-...`).
2. If the requesting account is not a domain controller computer account, treat as a likely credential-theft / DCSync attempt.
3. Confirm no legitimate replication tool or new DC promotion explains it, then contain the account and rotate KRBTGT (twice).

### 8.5 "Privileged group changed unexpectedly"

1. Alert on **4728/4732/4756** targeting Domain/Enterprise/Schema Admins and Administrators.
2. Correlate the actor (Subject account) and the time with a change ticket. No ticket = investigate immediately.
3. Remove the unauthorized member, review that member's recent **4624/4648** logons, and audit for additional persistence (**4698**, **4697/7045**, malicious GPOs).

### 8.6 "Logs were cleared"

Any **1102** (Security log cleared) or **104** (System log cleared) that you did not initiate is a strong indicator of active intrusion. This is exactly why forwarding logs off the DC matters — the forwarded copy survives.

---

## 9. Active Directory Certificate Services (AD CS) Attacks

If you run an enterprise Certificate Authority (very common in AD environments), it is a Tier 0 asset — compromising it usually means compromising the whole forest. The **ESC1–ESC8** family of misconfigurations, popularized by SpecterOps and weaponized in tools like **Certify** and **Certipy**, lets attackers enroll certificates that impersonate any user, including Domain Admins.

The most common issues:

- **ESC1** — a certificate template allows the enrollee to *supply the Subject Alternative Name (SAN)* and permits client authentication. An attacker requests a cert "as" a domain admin and authenticates with it.
- **ESC2** — a template usable for *Any Purpose* (or no EKU), effectively a subordinate CA cert.
- **ESC3** — an *Enrollment Agent* template that lets the holder request certs on behalf of others.
- **ESC4** — dangerous *ACLs on a template* (write access) so an attacker rewrites it into an ESC1 condition.
- **ESC6** — the CA has the `EDITF_ATTRIBUTESUBJECTALTNAME2` flag set, allowing SAN injection on *any* template.
- **ESC7** — dangerous permissions on the *CA itself* (ManageCA / ManageCertificates).
- **ESC8** — **NTLM relay to the CA web enrollment (HTTP) endpoint** — relay a coerced DC's machine authentication to obtain a DC certificate.

Find and defend:

```powershell
# Enumerate CAs and templates safely from a domain-joined host
certutil -config - -ping
certutil -TCAInfo
# Attackers audit with Certipy; defenders can too:
#   certipy find -u user@corp.example.com -p '***' -dc-ip <ip> -vulnerable
```

Defenses: remove SAN-supply on client-auth templates; require CA manager approval on sensitive templates; disable `EDITF_ATTRIBUTESUBJECTALTNAME2`; enforce **Extended Protection for Authentication (EPA)** and disable NTLM/HTTP on the web-enrollment endpoint (ESC8); tighten template and CA ACLs; and monitor issuance.

Key CA event IDs (on the CA server's Security log, with CA auditing enabled via `certutil -setreg CA\AuditFilter 127`):

| Event ID | Meaning |
|---|---|
| 4886 | Certificate request received |
| 4887 | Certificate issued (approved) |
| 4888 | Certificate request denied |
| 4899 / 4900 | Certificate template / CA security setting changed |
| 4768 | Watch for TGTs obtained via certificate (PKINIT) — correlate to unexpected issuance |

---

## 10. Windows LAPS — Local Admin Password Management

Shared or identical local Administrator passwords are the fuel for lateral movement (pass-the-hash across machines). **Windows LAPS** randomizes each machine's local admin password, stores it securely, and rotates it automatically. It is now built into Windows Server 2019/2022 and current Windows client builds (April 2023 update onward) — no separate MSI like the legacy "Microsoft LAPS."

Windows LAPS can store passwords in **Active Directory** or in **Entra ID**. AD storage uses the confidential attributes `msLAPS-Password` / `msLAPS-EncryptedPassword`, and supports **password encryption** so only authorized principals can read it.

Setup (AD-backed):

```powershell
# 1. Extend the schema (run once, as Schema Admin)
Update-LapsADSchema

# 2. Grant the computers the right to write their own password on their OU
Set-LapsADComputerSelfPermission -Identity "OU=Workstations,DC=corp,DC=example,DC=com"

# 3. Configure via GPO: Computer Config > Admin Templates > System > LAPS
#    (backup directory = AD, password complexity, age, encryption)

# 4. Read a password (authorized admins only)
Get-LapsADPassword -Identity "PC01" -AsPlainText
```

Monitor LAPS in the **Applications and Services Logs → Microsoft → Windows → LAPS/Operational** log (events 10003/10004/10005 for policy processing and password update). Restrict who can read the password attribute and audit reads (event **4662** against the LAPS attributes).

---

## 11. LDAP Signing & Channel Binding Hardening

Unsigned LDAP and LDAP-over-SSL without channel binding leave AD open to NTLM relay and man-in-the-middle attacks. Microsoft's 2020 hardening guidance recommends **requiring LDAP signing** and **LDAP channel binding** on all DCs.

Before enforcing, find non-compliant clients so you don't cause an outage. With diagnostic logging turned up, DCs log:

| Event ID | Log | Meaning |
|---|---|---|
| 2886 | Directory Service | LDAP signing is *not* required (informational — you are exposed) |
| 2887 | Directory Service | Summary count of unsigned/clear-text binds received (baseline your exposure) |
| 2888 | Directory Service | Clients were rejected because signing is now required |
| 2889 | Directory Service | Identifies a specific client (IP + account) making an unsigned/clear-text bind |

Turn up logging, then hunt 2889 to find offenders:

```cmd
reg add HKLM\SYSTEM\CurrentControlSet\Services\NTDS\Diagnostics /v "16 LDAP Interface Events" /t REG_DWORD /d 2 /f
```

Enforce via GPO under *Security Options*:
- **Domain controller: LDAP server signing requirements** = *Require signing*
- **Domain controller: LDAP server channel binding token requirements** = *Always* (or *When supported* during transition)

Remediate clients (update, reconfigure to use LDAPS/signing) before flipping to *Require*.

---

## 12. Trusts & Multi-Domain/Forest Security

Trusts let users in one domain/forest access resources in another. They also create attack paths if misconfigured.

**SID history & SID filtering.** The `sIDHistory` attribute carries a user's SIDs from a migrated domain. If **SID filtering (quarantine)** is disabled on a trust, an attacker in a trusted domain can inject a privileged SID (e.g., the trusting domain's Domain Admins RID 512) into a ticket and escalate across the trust. Keep SID filtering enabled on external and (where possible) forest trusts:

```cmd
netdom trust TrustingDomain /domain:TrustedDomain /quarantine:yes    :: enable SID filtering
netdom trust /domain:TrustedDomain /EnableSIDHistory:no              :: block SID history across the trust
```

**Selective authentication.** For forest trusts, use *selective authentication* so users are not implicitly allowed to authenticate to every resource — you grant the "Allowed to authenticate" right explicitly.

**Trust hygiene.** Remove stale/unneeded trusts, prefer **one-way** and **outbound** where the direction of access permits, and remember that a forest — not a domain — is the true security boundary. Treat any trusted forest's Tier 0 as equivalent to your own.

---

## 13. Read-Only Domain Controllers (RODCs)

RODCs hold a read-only copy of AD and are designed for branch offices or lower-trust locations. They do **not** store account passwords by default; instead a **Password Replication Policy (PRP)** governs which accounts' credentials may be cached on the RODC.

Key security points:

- Keep privileged accounts (Domain Admins, etc.) in the RODC's **Denied** PRP list so their credentials are never cached at a physically less-secure site.
- If an RODC is stolen, you only need to reset the passwords of the accounts that were actually cached on it — check with `repadmin /prp view <RODC> Reveal` / the "Advanced" tab of the RODC's Password Replication Policy dialog.
- RODCs use a per-RODC **KRBTGT account** (`krbtgt_<number>`), which limits blast radius if compromised.
- Delegate local administration on the RODC without granting domain rights (Administrator Role Separation).

RODCs reduce risk at exposed sites but are not a substitute for physical security or the tiering model.

---

## 14. Hybrid Identity — Entra Connect & Entra ID

Most on-prem AD environments now synchronize to **Microsoft Entra ID** (formerly Azure AD) using **Entra Connect** (formerly Azure AD Connect). This bridges your Tier 0 identity into the cloud and creates a large, often-overlooked attack surface.

**Treat the Entra Connect server as Tier 0.** It holds credentials/permissions that can read or write directory data and, depending on sync method, effectively holds keys to both directories. Harden it like a domain controller: restricted logons, patched, no browsing/email, monitored.

Authentication methods and their risks:

- **Password Hash Sync (PHS)** — a hash-of-the-hash of on-prem passwords syncs to Entra ID. Compromise of the Connect server or the `MSOL_` sync account can enable a **DCSync** (the sync account often has replication rights) → credential theft.
- **Pass-Through Authentication (PTA)** — an on-prem agent validates passwords; a malicious agent can harvest cleartext credentials.
- **Federation (AD FS)** — the AD FS **token-signing certificate** is a golden-SAML target: stealing it lets an attacker forge tokens for any user, bypassing MFA. Protect and rotate it.
- **Seamless SSO** — uses a computer account `AZUREADSSOACC$`. Its Kerberos key must be rolled periodically; if stolen it enables silver-ticket-style forgery against Entra SSO.

Guidance: minimize privileges on the sync account and audit its use (watch **4662** replication access from it), protect AD FS signing keys, roll the `AZUREADSSOACC$` key, enable **MFA / Conditional Access** in Entra, and monitor Entra ID sign-in and audit logs (via Entra sign-in logs / Microsoft Defender for Identity / Sentinel). A compromise on either side of the sync tends to become a compromise of both.

---

## 15. Fine-Grained Password Policies & gMSAs

**Fine-grained password policies (FGPP / PSOs)** let you apply different password and lockout rules to specific groups (e.g., stricter rules for admins) rather than one domain-wide policy. They require Windows 2008+ functional level.

```powershell
New-ADFineGrainedPasswordPolicy -Name "AdminPSO" -Precedence 10 `
  -MinPasswordLength 16 -PasswordHistoryCount 24 -LockoutThreshold 5 `
  -LockoutDuration 00:30:00 -LockoutObservationWindow 00:30:00 `
  -ComplexityEnabled $true -ReversibleEncryptionEnabled $false
Add-ADFineGrainedPasswordPolicySubject "AdminPSO" -Subjects "Domain Admins","Tier0 Admins"
Get-ADUserResultantPasswordPolicy jdoe    # see which policy actually applies to a user
```

**Group Managed Service Accounts (gMSAs)** eliminate manually-managed service-account passwords. AD generates and rotates a long random password automatically; authorized hosts retrieve it. This is the primary mitigation for Kerberoasting.

```powershell
# One-time per forest: create the KDS root key (in a lab, use -EffectiveTime to skip the 10h wait)
Add-KdsRootKey -EffectiveImmediately

New-ADServiceAccount -Name "svc-sql01" -DNSHostName svc-sql01.corp.example.com `
  -PrincipalsAllowedToRetrieveManagedPassword "SQL-Servers"
# On the target server:
Install-ADServiceAccount svc-sql01
```

---

## 16. Assessment & Detection Tooling

Beyond the built-in logs, these tools baseline and hunt for AD risk:

- **PingCastle** — fast, scored health/risk report of an AD domain; great for periodic posture checks and executive-friendly output.
- **Purple Knight** (Semperis) — free AD/Entra security assessment mapping to known attack indicators.
- **BloodHound / SharpHound** — graphs attack paths ("who can reach Domain Admin"), surfacing dangerous ACLs, delegation, and nested membership. Used by both attackers and defenders — run it yourself first.
- **Microsoft Defender for Identity (MDI)** — sensor on DCs that detects reconnaissance, lateral movement, DCSync, golden tickets, and more, feeding Microsoft 365 Defender / Sentinel.
- **`dcdiag` / `repadmin` / Best Practices Analyzer** — built-in health checks (covered in section 3).
- **Microsoft Security Compliance Toolkit + LGPO** — apply and audit Microsoft's security baselines for Server 2019/2022.

Run posture tools on a schedule and track findings over time — regression is common after operational changes.

---

## 17. DNS Security

AD depends on DNS, and DNS is itself attackable.

- **Secure dynamic updates** — set AD-integrated zones to *Secure only* so only authenticated/authorized machines can register or overwrite records (prevents record hijacking).
- **Scavenging & aging** — enable to remove stale records that cause connectivity confusion and can be re-registered maliciously.
- **Zone transfers** — restrict or disable (`allow-transfer` to named servers only) so attackers can't dump your entire namespace.
- **DNS admin risk** — members of **DnsAdmins** historically could load an arbitrary DLL into the DNS service (running as SYSTEM on a DC) — effectively DC compromise. Treat DnsAdmins as privileged and audit it.
- **Known DNS server vulnerabilities** — keep DCs patched; *SIGRed* (CVE-2020-1350) was a wormable RCE in Windows DNS Server. DNS patches on DCs are high priority.
- **Monitor** the DNS Server operational log and enable **DNS analytical/audit logging** when hunting for tunneling or abuse.

---

## 18. Backup, DC Recovery & Forest Recovery Runbook

Object-level restore (AD Recycle Bin) is covered in section 4.8. This section covers larger failures.

**System-state backups.** Back up system state on at least two DCs regularly with Windows Server Backup or your enterprise product. Verify backups are younger than the tombstone lifetime (180 days) — an older backup cannot be safely restored.

**Single DC failure (other DCs healthy).** Do *not* restore from backup. Forcibly demote/clean the dead DC's metadata (`ntdsutil metadata cleanup` or `Remove-ADDomainController -ForceRemoval`), seize any FSMO roles it held (`ntdsutil roles → seize`), then build a fresh DC. Use **Install From Media (IFM)** to speed promotion over slow links: `ntdsutil "activate instance ntds" ifm "create sysvol full <path>"`.

**Authoritative vs non-authoritative restore.** A *non-authoritative* restore brings a DC back and lets normal replication update it. An *authoritative* restore (via `ntdsutil authoritative restore`) marks specific objects/subtrees as the master copy so they overwrite other DCs — used to recover accidentally deleted objects when the Recycle Bin isn't available.

**Full forest recovery (worst case — e.g., ransomware or a golden-ticket compromise across all DCs).** Follow Microsoft's *AD Forest Recovery* guide. High-level runbook:

1. **Isolate** — disconnect the environment; assume all DCs and credentials are compromised.
2. **Choose one DC per domain** to restore from a known-good backup (recover the forest root first).
3. Restore that DC, **seize all FSMO roles** onto it, and **clean up metadata** for every other DC.
4. **Raise the RID pool**, **reset the KRBTGT password twice**, reset the DSRM password, and reset trust passwords.
5. Reset credentials for all privileged accounts and any accounts that could have been cached/compromised.
6. Rebuild remaining DCs fresh (do not restore multiple DCs and let them replicate — reintroduce one at a time).
7. Re-establish DNS, SYSVOL/DFS-R, and validate with `dcdiag` / `repadmin` before reconnecting clients.

Practice this in a lab. The difference between hours and weeks of downtime is a rehearsed runbook and validated backups.

---

## 19. Legacy Credential Exposure & SIEM Queries

**Group Policy Preferences (`cpassword`).** Old GPP items (mapped drives, local users, scheduled tasks) could store credentials in SYSVOL encrypted with a *published* AES key — trivially decryptable by any authenticated user. Search SYSVOL for `cpassword` in the XML files and remove any found:

```powershell
Get-ChildItem "\\corp.example.com\SYSVOL" -Recurse -Include *.xml |
  Select-String "cpassword"
```

**Reversible encryption & LM hashes.** Ensure "Store passwords using reversible encryption" is disabled and LM hash storage is off (`NoLMHash`).

**Example detection queries (Microsoft Sentinel / KQL).** If you forward Security events to Sentinel/Log Analytics:

```kql
// Password spray: one source, many distinct accounts failing in 10 min
SecurityEvent
| where EventID == 4625
| summarize Accounts=dcount(TargetUserName), Attempts=count() by IpAddress, bin(TimeGenerated, 10m)
| where Accounts >= 10
```

```kql
// Possible Kerberoasting: RC4 service ticket requests
SecurityEvent
| where EventID == 4769 and TicketEncryptionType == "0x17"
| summarize SPNs=dcount(ServiceName) by Account, bin(TimeGenerated, 1h)
| where SPNs >= 5
```

```kql
// Security log cleared anywhere in the estate
SecurityEvent | where EventID == 1102
```

Splunk equivalents use `EventCode=4625/4769/1102` with `stats dc(...) by ...`. Whatever the SIEM, the detection logic mirrors the workflows in section 8 — the value is in centralizing the logs so these queries can run across all DCs at once.

---

## 20. Hardening Checklist

- Use the **AD Recycle Bin** and maintain tested system-state backups of DCs.
- Apply the **tiered administration model**: Tier 0 (DCs/identity) admins never log on to workstations; use separate admin accounts and **Privileged Access Workstations**.
- Put admins in the **Protected Users** group and enable **Credential Guard** + **LSASS protection (RunAsPPL)**.
- Rotate the **KRBTGT** password periodically (twice, with replication delay) and after any suspected compromise.
- Replace service-account passwords with **gMSAs**; **disable RC4** Kerberos encryption; require **AES**.
- **Disable SMBv1**, require **LDAP signing and channel binding**, and reduce/monitor **NTLM** use.
- Enforce **strong password / lockout policies** and **MFA** for privileged and remote access.
- Turn on **Advanced Audit Policy**, **command-line process auditing (4688)**, and **PowerShell script block logging (4104)**; forward everything to a **SIEM/WEF collector**.
- Regularly review **privileged group membership**, **delegation settings**, **dangerous ACLs**, and **accounts with SPNs or pre-auth disabled**.
- Keep DCs **patched**, minimize installed roles/software on DCs, and restrict inbound network access to them.
- Consider **Server 2022 Secured-core** features and **SMB encryption** for new deployments.
- Deploy **Windows LAPS** to randomize local admin passwords and kill cross-machine pass-the-hash.
- Audit and harden **AD CS** templates and CA settings (ESC1–ESC8); enable EPA and disable HTTP web enrollment.
- Enforce **LDAP signing and channel binding** (baseline with events 2887/2889 first).
- Keep **SID filtering** on trusts, use **selective authentication** on forest trusts, and remove stale trusts.
- Treat the **Entra Connect / AD FS** servers as Tier 0; protect the AD FS signing cert and roll the `AZUREADSSOACC$` key.
- Remove **GPP `cpassword`** artifacts from SYSVOL and disable reversible encryption / LM hashes.
- Run **PingCastle / BloodHound / Defender for Identity** periodically and track findings over time.
- Maintain and **rehearse a forest recovery runbook**; verify backups are within the tombstone lifetime.

---

*Applies to Windows Server 2019 and 2022 AD DS. Event IDs and cmdlets are consistent across both versions; verify auditing is enabled before relying on any Security-log event. Test all changes in a lab or with a change window before applying to production domain controllers.*

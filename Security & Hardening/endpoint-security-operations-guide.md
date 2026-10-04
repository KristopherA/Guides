> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Endpoint Security Operations

## Overview

Endpoints are where the organization's incidents start. Someone opens an attachment, plugs in
a device, installs something from a search result, or reuses a password on a
machine that then gets compromised. The security tooling around the perimeter —
pfSense with Snort, fail2ban, the Barracuda WAF, 802.1x on the network edge — is
in reasonable shape and well documented. What has not been written down is the
day-to-day operation of endpoint protection: what is actually watching each
machine, what gets checked and how often, what happens when an alert fires, and
how to take a suspect machine off the network without destroying the evidence
you will need an hour later.

This guide covers that. It is deliberately the operational layer: the deep
forensic work already exists in `Security Procedures/modules/macos-host-triage.txt`
and `Security Procedures/IR_Security_Scenarios_Guide.md`, and this document's job is to get a case to
those, correctly and with evidence intact, rather than to restate them.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Owner            | IT lead (it@example.com) — process owner `[CONFIRM: proposed default]` |
| Environment      | prod — the whole managed endpoint estate (macOS, Windows, Linux servers) |
| Location         | macOS fleet managed by Munki `[FILL IN: Munki repo host/URL]`; MDM `[FILL IN: MDM in use and console URL]`; AV/EDR console `[FILL IN: AV/EDR product and console URL]`; Graylog graylog01.example.com |
| Access           | `[FILL IN: how the AV/EDR console is reached and who has access]`; MDM console `[FILL IN]`; SSH/ARD to Macs; RDP/PowerShell remoting to Windows |
| Dependencies     | Active Directory (identity, machine accounts, 802.1x); Munki repo; MDM; the AV/EDR cloud console and its definition updates; Graylog for alerting; network access via pfSense |
| Dependents       | Every user's ability to work. Containment actions here have immediate, visible user impact — which is why the isolate procedure below is a written procedure and not a judgement call made under pressure |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                               |

## How It Works

### The fleet, and what protects each class

| Class | Management | Protection layer | Telemetry to Graylog | Notes |
|-------|-----------|------------------|----------------------|-------|
| macOS workstations | **Munki** for software and patching (`_ARCHIVE/superseded-stubs/Adding a package to munki.txt` (archived 2026-09-11)); MDM `[FILL IN: MDM in use]` for configuration profiles and FileVault escrow | Built-in: FileVault, Gatekeeper, XProtect, System Integrity Protection, application firewall. Plus `[FILL IN: AV/EDR product on macOS]` | `[FILL IN: is macOS telemetry shipped to Graylog, and how?]` | Compliance measured by `Support Procedures/macos-nist-check.sh` |
| Windows workstations | `[FILL IN: Windows workstation management — GPO, WSUS, Intune?]` | Microsoft Defender (`Get-MpComputerStatus` is already the documented check in `Windows & Mac Workstations/windows-server-2019-runbook.md`); BitLocker `[FILL IN: is BitLocker deployed?]` | `[FILL IN: Windows Event Log forwarding to Graylog?]` | `[FILL IN: size of the Windows workstation estate]` |
| Windows servers / domain controllers | GPO + patch workflow | Defender; AD-specific detections per `Active Directory/AD-Admin-Security-Guide.md` | `[FILL IN]` | DCs are the highest-value endpoints in the estate and should be treated as such, not as "servers" |
| Ubuntu servers | apt + patch workflow; hardened per `fresh-box-hardening-cheatsheet.txt` | Host hardening (Lynis), SSH hardening, fail2ban, auditd host telemetry, Trivy for package CVEs | Yes — Graylog Sidecar/Filebeat, "ORG Audit" stream | No conventional AV; the control set is hardening + telemetry + patching |
| Container hosts | Docker per service | Trivy image and config scanning; Docker Bench CIS | Yes, same pipeline | Compromise of a container host is a host compromise, not a container one |
| Network edge (all classes) | — | pfSense + Snort, 802.1x/RADIUS for network admission | Yes | Snort is the detection layer that sees what a host's own agent may miss |

The honest summary: macOS and Ubuntu are the well-covered classes, and the
Windows workstation estate is the one with the most `[FILL IN]` markers above.
Filling those in is the highest-value follow-up from this document.

### How a case reaches a person

```
AV/EDR console alert   ->  triage (this doc)  ->  contain  ->  triage modules  ->  IR guide
Graylog alert / Snort  ->
User report            ->
Phishing triage stage 3 ->
```

The last of those matters: `Security Procedures/WORKFLOW.txt` step 4 establishes
during email intake whether anyone actually opened the attachment or entered
credentials, and step 13 contains the whole chain. When the answer is "yes, on
that Mac", the case arrives here and the host becomes the subject.

## Operations (Day-2)

### Daily checks

Fifteen minutes, first thing. The point is to notice absence as much as
presence — an agent that stopped reporting is a more serious finding than a
blocked threat, because the block worked and the silence did not.

1. **AV/EDR console — open alerts.** Anything new since yesterday, anything
   still open from before. `[FILL IN: AV/EDR console URL]`
2. **AV/EDR console — agent health.** Machines not reporting in 7 days,
   machines with the agent disabled or out of date, machines with protection
   features turned off. `[CONFIRM: proposed default — 7-day threshold]`
3. **Graylog — overnight security events.** Snort alerts, fail2ban actions,
   failed-authentication spikes, and any break-glass or service account
   authentication (see `secrets-management-guide.md`).
4. **New endpoints on the network.** Anything that authenticated via 802.1x or
   appeared in DHCP that is not in the inventory.
5. **Overnight job failures** that could mask a security problem — a backup that
   did not run, a Munki run that failed fleet-wide.

Record anything that needed action. A daily check with no written output cannot
be shown to have happened, and the pattern across a month is often the finding.

### Weekly checks

1. **Definition and agent version currency** across the fleet — see Patch and
   definition currency below.
2. **Munki run health**: how many Macs checked in this week, how many failed to
   apply a pending install.
3. **Patch currency**: Macs behind on OS updates, Windows machines missing the
   last patch cycle, servers pending reboot.
4. **Encryption status**: FileVault enabled and key escrowed on every Mac;
   BitLocker where applicable. A machine whose key is not escrowed is a data-loss
   incident waiting for a lost laptop.
5. **Local administrator membership** on workstations and member servers.
   Privilege creep is normal and invisible unless someone looks —
   `Windows & Mac Workstations/windows-server-2019-runbook.md` already flags this as baseline hygiene.
6. **Exclusions review**: has anything been added to an AV exclusion list this
   week (see Exclusions policy below).
7. **Suppressed/muted alerts**: anything muted "temporarily" that is still muted.

### Monthly / quarterly

- Monthly: fleet compliance summary, feeding the monthly report in
  `vulnerability-management-process.md`.
- Quarterly: run `Support Procedures/macos-nist-check.sh` across a sample of the
  Mac fleet `[CONFIRM: proposed default]` and track PASS/FAIL/WARN counts over
  time; review the exclusions register in full; verify the isolate procedure
  still works by exercising it on a test machine.

```
sudo bash "/path/to/macos-nist-check.sh" > ~/Desktop/nist-$(hostname)-$(date +%Y%m%d).txt   # local run, sudo required or checks return SKIP
ssh user@host 'sudo bash -s' < macos-nist-check.sh > nist-$(date +%Y%m%d)-hostname.txt      # remote run over SSH, no install needed on the target
```

### Alert triage workflow

**Step 1 — Establish what the alert actually says.** Detection name, file path,
hash, process tree, the user, the machine, and the time. Then the question that
decides everything downstream: **did it block, or did it only detect?** A blocked
execution is hygiene. An allowed execution is an investigation.

**Step 2 — Establish context before acting.** Who is the user, what were they
doing, is this machine associated with a phishing report or a recent ticket, is
the same detection firing elsewhere in the fleet? A detection on three machines
in an hour is a campaign, not three alerts.

**Step 3 — Classify.**

| Class | Meaning | Action |
|-------|---------|--------|
| True positive, blocked | Protection worked | Verify no other artefacts on the host, record, close |
| True positive, allowed to run | Something executed | Isolate now. Go to step 5 |
| Suspicious / unconfirmed | Cannot tell from the console | Investigate on the host before deciding; do not close on ambiguity |
| False positive | Known-good software misdetected | Record; consider an exclusion via the policy below — never add one ad hoc |
| Informational | PUP, adware, policy violation | Remediate on the normal support queue |

**Step 4 — Severity and response time.** All proposed.

| Severity | Examples | Acknowledge within | Contain within |
|----------|----------|--------------------|----------------|
| Critical | Ransomware behaviour, confirmed C2, credential dumping, EDR agent tampered with or disabled, detection on a DC or server | 15 min `[CONFIRM: proposed default]` | Immediately — isolate first, ask later |
| High | Malware allowed to run, persistence created, lateral movement indicators, detection on a machine whose user reported a phish | 1 hour `[CONFIRM: proposed default]` | 4 hours `[CONFIRM: proposed default]` |
| Medium | Malware blocked on execution, suspicious script activity, unexpected admin tool | 4 business hours `[CONFIRM: proposed default]` | Next business day |
| Low | PUP/adware, policy violation, single blocked web request | 1 business day `[CONFIRM: proposed default]` | Normal support queue |

Out-of-hours: Critical alerts page `[FILL IN: after-hours contact and paging
method]`; everything else waits for the morning
`[CONFIRM: proposed default]`.

**Step 5 — On a Critical or an allowed execution: isolate, then preserve, then
investigate.** In that order. Isolation stops the spread; preservation protects
what you need to understand it. Both come before any attempt to clean the
machine.

**Step 6 — Hand off.** Once the host is isolated and evidence is captured, the
investigation proper happens in the existing modules:

- macOS host: `Security Procedures/modules/macos-host-triage.txt` — five phases,
  from what is running through persistence, binary verification, system logs, to
  contain and remediate.
- Either platform, by scenario: `Security Procedures/IR_Security_Scenarios_Guide.md` — scenario 1
  (suspicious process/malware), 2 (beaconing), 5 (compromised account),
  6 (ransomware), 10 (macOS persistence), 11 (memory forensics).
- Came from an email or document: `Security Procedures/START-HERE.txt`.

**Step 7 — Feed it forward.** Indicators from the case go to
`Security Procedures/modules/detection-engineering.txt`, and the resulting rule
goes to `Security Procedures/modules/detection-validation.md` to prove it fires. A case that produces no
detection has not finished.

### Isolate a host — macOS

Goal: cut network reachability while keeping the machine powered on and its
volatile state intact. **Do not shut it down** — memory, running processes and
open network connections are the most valuable evidence and they do not survive a
reboot.

Preferred method, in order:

1. **Network-level isolation.** Because Example Org runs 802.1x, the cleanest cut is at
   the network: move the machine's port or MAC to a quarantine VLAN, or reject
   its 802.1x authentication. This works even if the host is actively hostile,
   and it does not require touching the machine.
   `[FILL IN: quarantine VLAN ID and the exact steps to move a port/MAC into it]`
   Related: `Putting a switch port into Server VLAN` documents the general
   mechanism for moving a port between VLANs.
2. **EDR console isolation**, if the product supports it — usually preserves a
   management channel back to the console.
   `[FILL IN: does the AV/EDR product support host isolation, and how is it invoked?]`
3. **On the host itself**, if you have hands on it and neither of the above is
   available:

```
sudo ifconfig en0 down                                                        # wired interface down — replace en0 with the active interface
networksetup -setairportpower en1 off                                         # Wi-Fi radio off
/usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate               # confirm the application firewall state while you are here
```

Physically unplugging the cable and turning off Wi-Fi from the menu bar is
equally valid and faster. Do not disable it by removing it from AD or MDM —
that removes your management access without removing the attacker's.

Then, before anything else:

- Tell the user what has happened and that the machine is not to be used, shut
  down, or "fixed". Users switch machines off to be helpful.
- Record the time of isolation. Every later timeline depends on it.
- Do not log in with a domain admin account. Credentials used on a compromised
  host are compromised — this is how a workstation incident becomes a domain
  incident.

### Isolate a host — Windows

Same principles, same order of preference: network-level isolation first, EDR
console isolation second, host-level last.

```
Get-NetAdapter | Where-Object Status -eq 'Up'                          # identify the live adapter before disabling anything
Disable-NetAdapter -Name "<AdapterName>" -Confirm:$false               # take the interface down, machine stays powered on
Get-MpComputerStatus                                                   # Defender state, definition age, last scan — capture before remediation
```

Additional Windows-specific actions:

- **Do not** disable the computer account in AD as the containment step. It
  breaks your own access and Group Policy without stopping anything local.
- Disable the *user's* account if credential theft is suspected, and reset the
  password from a different, clean machine —
  `SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt` has the reset
  command, and `Security Procedures/IR_Security_Scenarios_Guide.md` scenario 5 has the account side.
- If the machine is a server or a DC, this is a different severity of incident
  from the first minute. Escalate immediately and do not work alone.
- Preserve the Windows event logs before anything else — they roll.

### Evidence preservation before reimaging

Reimaging destroys the answer to "how did this happen, and is it anywhere else?"
Capture first. The whole capture takes minutes; reconstructing it after a wipe
takes days and usually fails.

**Order matters — most volatile first:**

1. **Memory**, if the incident warrants it (ransomware, confirmed C2, credential
   theft). Memory is gone the instant the machine powers off.
   `Security Procedures/IR_Security_Scenarios_Guide.md` scenario 11 covers acquisition and analysis.
2. **Live state** — running processes, network connections, logged-in users.
3. **Persistence and filesystem artefacts.**
4. **Logs.**
5. **Disk image**, if the case is serious enough to warrant one.
   `[FILL IN: is full-disk imaging in scope for the organization, and with what tooling?]`

On macOS, do not improvise the capture — `Security Procedures/modules/macos-host-triage.txt`
already walks the phases in the right order (what is running and what it is
talking to; persistence; verify a specific binary; system logs; contain and
remediate). Run it and keep the output. For a quick live snapshot before you
start:

```
lsof -i -n -P | grep ESTABLISHED > ~/evidence/connections.txt        # who is it talking to, right now
ps aux > ~/evidence/processes.txt                                    # full process list with command lines
log collect --last 7d --output ~/evidence/system-logs.logarchive     # unified log archive, before it rolls
```

Also run the compliance baseline while the machine is in its found state —
it captures firewall, FileVault, SIP, Gatekeeper and update posture in one pass,
which is exactly the "was this machine configured correctly when it was
compromised" question that gets asked later:

```
sudo bash "Support Procedures/macos-nist-check.sh" > ~/evidence/nist-$(date +%Y%m%d).txt   # NIST SP 800-53 posture snapshot, native tools only
```

`Support Procedures/macos-remote-diag.sh` and
`Support Procedures/macos-desktop-support-runbook.txt` cover the non-security
diagnostic collection on the same fleet.

Handling rules for whatever you collect:

- Copy evidence **off** the host to `[FILL IN: evidence storage location]`, not
  to a synced folder — this is the same rule as the triage toolkit's "work on a
  copy in a local non-synced folder".
- Hash everything on collection and record the hashes.
- Note who collected what, when, and from which machine.
- Keep it for `[FILL IN: evidence retention period]` `[CONFIRM: proposed default]`.
- If the incident might involve member data, employment consequences, or
  anything that could become a legal matter, stop and escalate before reimaging.
  `[FILL IN: who to notify when an incident may have legal or privacy implications]`

**Only then** reimage. Rebuild from the standard image, not from a backup of the
compromised state, and rotate every credential that was used on that machine —
see `secrets-management-guide.md`.

### Patch and definition currency verification

Currency is the control that does most of the work, and it fails quietly. Check
it rather than assuming it.

macOS, per host:

```
softwareupdate -l                                                    # pending OS and security updates
/usr/sbin/managedsoftwareupdate --checkonly                          # Munki: what is pending for this machine
defaults read /Library/Preferences/ManagedInstalls LastCheckDate      # Munki: when did this Mac last check in
```

Windows, per host:

```
Get-MpComputerStatus | Select AMServiceEnabled,RealTimeProtectionEnabled,AntivirusSignatureAge,QuickScanAge   # Defender enabled, signature age, last scan
Get-HotFix | Sort-Object InstalledOn -Descending | Select -First 5                                            # last patches actually installed
```

Fleet-wide, the numbers that matter each week:

- Percentage of endpoints with definitions less than 3 days old
  `[CONFIRM: proposed default]`.
- Machines that have not checked in to Munki / MDM / the EDR console in 7 days
  `[CONFIRM: proposed default]`.
- Machines with real-time protection disabled — should be zero, and any non-zero
  value is investigated, not noted.
- Machines pending a reboot to complete a patch.

`[FILL IN: how fleet-wide currency is reported — console dashboard, export, or script]`

Server and OS patching itself is not owned here — that is
`SysAdmin Procedures/Patch_Management_Workflow.txt`, including the macOS
workstation line (user-deferred, 7 days, ongoing via MDM). This section verifies
that what that process is supposed to have done actually happened.

### Exclusions policy

Every AV/EDR exclusion is a permanent hole, and exclusions are where endpoint
protection quietly stops working. Attackers look for exclusion paths, because a
folder the agent does not inspect is the best place to keep a payload.

Rules:

- **An exclusion is never added to close an alert.** Investigate first, confirm
  the false positive, then consider an exclusion.
- **Narrowest possible scope.** A specific file hash or a fully-qualified path,
  never a whole directory tree, never a bare extension, never a wildcard that
  covers user-writable locations. `C:\Users\*` and `/Users/*` are never
  acceptable exclusions.
- **Never exclude** user profile directories, temp directories, download folders,
  or any location a non-admin can write to.
- **Every exclusion has an owner, a justification, and a review date.**
  `[CONFIRM: proposed default — 12-month review]`
- **Approval required before adding one:**
  `[FILL IN: who approves an AV/EDR exclusion]` — proposed: the process owner
  approves scoped exclusions; anything covering a directory or a server class
  needs a second approver `[CONFIRM: proposed default]`.
- **Recorded in a register**, not only in the console.
  `[FILL IN: where the exclusions register is kept]`
- Reviewed weekly for additions, quarterly in full.

A performance complaint is a legitimate reason to *investigate* an exclusion —
a database or backup path genuinely may need one — but the legitimate version is
a scoped path approved and recorded, not a broad exclusion added under pressure
and forgotten.

### Handoff into incident response

The boundary is worth stating so nothing falls through it. This document owns the
endpoint up to and including **isolation and evidence preservation**. Everything
after that belongs to the existing material:

| Situation | Goes to |
|-----------|---------|
| Suspicious process or binary on a Mac | `Security Procedures/modules/macos-host-triage.txt`, then `Security Procedures/IR_Security_Scenarios_Guide.md` scenario 1 |
| Persistence found on macOS | `Security Procedures/IR_Security_Scenarios_Guide.md` scenario 10 |
| Host beaconing to an unknown destination | `Security Procedures/IR_Security_Scenarios_Guide.md` scenario 2 |
| User account believed compromised | `Security Procedures/IR_Security_Scenarios_Guide.md` scenario 5; AD side in `Active Directory/AD-Admin-Security-Guide.md` |
| Ransomware behaviour | `Security Procedures/IR_Security_Scenarios_Guide.md` scenario 6 — isolate first, then that document, and escalate immediately |
| Memory capture needed | `Security Procedures/IR_Security_Scenarios_Guide.md` scenario 11 |
| Arrived via email or document | `Security Procedures/START-HERE.txt` / `Security Procedures/WORKFLOW.txt` |
| Sample needs detonating | `Security Procedures/modules/detonation-lab/` |
| Credentials or keys exposed on the host | `Security & Hardening/secrets-management-guide.md` |
| Indicators to turn into detections | `Security Procedures/modules/detection-engineering.txt`, then `Security Procedures/modules/detection-validation.md` |

Escalate immediately, mid-triage, without finishing the paperwork, on any of:
confirmed C2, ransomware behaviour, credential dumping, lateral movement tooling,
a server or domain controller involved, the EDR agent found disabled or tampered
with, or any indication member data has been accessed. These are the same
triggers as `Security Procedures/WORKFLOW.txt` step 14, deliberately.

`[FILL IN: incident escalation contact and out-of-hours path]`

## Troubleshooting

### Symptom: a machine stops reporting to the AV/EDR console

- Likely cause: usually mundane — machine off, off-network, or the agent needs a
  restart. Occasionally the agent was disabled, which is a Critical alert.
- Check: last check-in in the console; whether the machine appears in Munki/MDM or in Graylog at the same time. A machine reporting to Munki but not to the EDR console is a very different finding from one reporting to neither.
- Fix: reinstall or restart the agent. If the agent was deliberately disabled,
  treat it as Critical and isolate before investigating.

### Symptom: Munki has not run on a group of Macs

- Likely cause: repo unreachable, a broken manifest, or a failing package
  blocking the run.
- Check: `sudo /usr/local/munki/managedsoftwareupdate --checkonly -vv` on one affected Mac; `/Library/Managed Installs/Logs/ManagedSoftwareUpdate.log`.
- Fix: per `_ARCHIVE/superseded-stubs/Adding a package to munki.txt` (archived 2026-09-11). Treat a
  fleet-wide Munki stall as a security issue, not just a support one — it means
  patches are not landing.

### Symptom: user reports the machine is slow right after an alert

- Likely cause: a scan in progress, or the machine is actually doing something
  it should not be.
- Check: `ps aux | head -20` for CPU consumers, and `lsof -i -n -P | grep ESTABLISHED` for unexpected connections.
- Fix: do not resolve this by adding an exclusion. If a scan is the cause,
  reschedule scans; if it is not, you have found something.

### Symptom: a detection fires repeatedly on the same file across many machines

- Likely cause: either a false positive on deployed software, or a genuinely
  distributed compromise — most often the former via a Munki or GPO deployment.
- Check: is the file part of a managed deployment? Compare the hash against the package in the Munki repo.
- Fix: if deployed by us, verify the package source before excluding anything —
  a compromised deployment package is how one endpoint incident becomes a fleet
  incident. If not deployed by us, it is spreading, and that is a Critical.

### Symptom: isolated host still has network access

- Likely cause: a second interface — Wi-Fi still up after the cable was pulled,
  a USB adapter, a tethered phone, or a VPN client reconnecting.
- Check: `ifconfig -a` (macOS) or `Get-NetAdapter` (Windows) — look for every adapter, not the one you expect.
- Fix: this is why network-level quarantine is the preferred method; it does not
  depend on enumerating interfaces correctly under pressure.

### Symptom: FileVault or BitLocker key is not escrowed when a machine needs recovery

- Likely cause: enabled outside the managed workflow, or MDM enrolment failed
  after encryption was turned on.
- Check: MDM console's recovery key record for that serial.
- Fix: reissue the recovery key through MDM. Add escrow verification to the
  weekly checks — it is in the list above for this reason.

### Symptom: an exclusion nobody remembers adding

- Likely cause: added under pressure to close an alert, which is the practice the
  policy above exists to stop.
- Check: the exclusions register against the live console configuration — they drift, and the drift is the finding.
- Fix: remove it unless it has a current justification and owner. Reconcile the
  register and the console every quarter.

## Security

- Exposure: the AV/EDR console and the MDM console can both execute code on
  every endpoint. They are administrative crown jewels — MFA mandatory, access
  restricted to named administrators, and console logins monitored.
  `[FILL IN: who has console access to AV/EDR and to MDM]`
- Munki repo integrity: anything in the repo installs on every Mac. Write access
  to it is equivalent to code execution across the fleet. `[FILL IN: who can
  write to the Munki repo, and how is that access controlled]`
- Tamper protection: the agent must not be disableable by a local administrator.
  `[FILL IN: is tamper protection enabled on the AV/EDR agent?]`
- Local admin rights on workstations: `[FILL IN: current policy — do standard
  users have local admin?]` This single setting determines how most endpoint
  incidents end.
- Credentials used during response: never log in to a suspect host with a
  privileged account. See `secrets-management-guide.md`.
- Encryption: FileVault on every Mac with the key escrowed; BitLocker
  `[FILL IN]`.

## Monitoring & Alerting

- Monitored: AV/EDR detections; agent health and check-in recency; Munki/MDM
  check-in recency; definition age; Snort alerts from pfSense; fail2ban actions;
  authentication anomalies — all into Graylog (graylog01.example.com).
- Alert on absence as well as events: an endpoint that stops reporting, a Munki
  run that stops fleet-wide, an agent that goes from enabled to disabled. These
  are the failures that do not announce themselves.
- Highest-priority alerts, pages out of hours: EDR agent disabled or tampered
  with; ransomware behaviour; credential dumping; detection on a domain
  controller.
- `[FILL IN: does the AV/EDR product forward alerts into Graylog, or is the
  console the only view?]` — a second console nobody watches at 7pm is how
  alerts get missed; consolidating into Graylog is the goal.
- `[FILL IN: alert destination — email address or channel]`
- Baseline: `[FILL IN: normal weekly detection volume and fleet check-in
  percentage]`. Without a baseline, "unusual" is not actionable.

## Disaster Recovery

- Fleet-wide compromise (many endpoints at once): this is a major incident, not
  an endpoint operations task. Isolate at the network layer rather than
  host-by-host, and escalate. `[FILL IN: major incident escalation path]`
- Rebuild path for a single workstation: reimage from the standard build, re-enrol
  in MDM/Munki, restore user data from `[FILL IN: user data backup mechanism for
  workstations]`, rotate credentials used on the machine.
- `Hardware & Backup/` and `SysAdmin Procedures/Backup_DR_Runbook.txt` cover the
  restore side. An untested restore is a rumour — confirm the workstation restore
  path has been exercised. `[FILL IN: date workstation restore was last tested]`
- If the AV/EDR console or MDM is unavailable, network-level isolation via
  802.1x/VLAN still works. That redundancy is the reason it is the preferred
  isolation method above.
- Escalation if the owner is unavailable: `[FILL IN: escalation contact]`

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why / Ticket |
|------------|-------------------|--------------|
| 2026-09-11 | Created this guide over the existing macOS triage module, NIST check script and IR scenarios guide | Endpoint operations, isolation and evidence handling were undocumented; forensic depth already existed but nothing routed a case to it |
| `[FILL IN]` | `[FILL IN: record tooling changes, policy changes and significant incidents here]` | `[FILL IN]` |

## References

- `Security Procedures/modules/macos-host-triage.txt` — five-phase macOS host triage; the main handoff target
- `Security Procedures/START-HERE.txt`, `Security Procedures/WORKFLOW.txt` — phishing/document triage; step 4 identifies the affected host, step 13 contains the chain
- `Security Procedures/modules/detection-engineering.txt`, `Security Procedures/modules/detection-validation.md` — turning case indicators into proven detections
- `Security Procedures/modules/detonation-lab/` — when a sample needs executing safely
- `Security Procedures/IR_Security_Scenarios_Guide.md` — scenarios 1, 2, 5, 6, 10, 11
- `Support Procedures/macos-nist-check.sh` — NIST SP 800-53 Rev 5 posture check, native tools, runs locally or over SSH
- `Support Procedures/macos-desktop-support-runbook.txt`, `Support Procedures/macos-remote-diag.sh` — non-security diagnostics on the same fleet
- `_ARCHIVE/superseded-stubs/Adding a package to munki.txt` (archived 2026-09-11) — Munki package workflow
- `Windows & Mac Workstations/windows-server-2019-runbook.md` — `Get-MpComputerStatus`, local admin hygiene, secedit baselines
- `Active Directory/AD-Admin-Security-Guide.md` — AD attack paths that begin at an endpoint
- `SysAdmin Procedures/Patch_Management_Workflow.txt` — the patching this guide verifies
- `SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt` — account disable and password reset commands
- `SysAdmin Procedures/Backup_DR_Runbook.txt` — restore side of a rebuild
- `Security & Hardening/secrets-management-guide.md` — credential rotation after a host compromise
- `Security & Hardening/vulnerability-management-process.md` — endpoint findings and fleet currency reporting
- `[internal KB: unblocking an IP from Snort]`, `[internal KB: fail2ban unban procedure]`
- `Putting a switch port into Server VLAN` — VLAN move mechanics, basis for quarantine VLAN isolation

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Business Continuity Plan

## Overview

This plan covers how the organization keeps working when a system, a building, or a
person is unavailable for an extended period. It is about **the organization's
work continuing** — grievances, arbitration deadlines, member enquiries,
payroll, negotiations — not about the technical restoration of servers.
The technical restoration is documented elsewhere, and this plan points at
it rather than repeating it.

It exists because of a specific gap. The library has a good runbook for a
**power event** (`Hardware & Backup/power-outage-shutdown-runbook.md`) and
a good field guide for **security incidents**
(`Security Procedures/IR_Security_Scenarios_Guide.md`). Neither answers
the questions that matter to staff on the second day of an outage:

- Zimbra has been down since yesterday morning. How do members reach us?
- erp-db.example.com is unrecoverable this week and payroll runs Thursday.
- The server room is inaccessible and nobody knows for how long.
- An arbitration submission is due Friday and the file is on files.example.com.
- The sysadmin is unreachable and something has broken.

Those are continuity questions. They have organisational answers, not
technical ones, and most of them have to be decided **before** the outage.

**A candid statement of the plan's current status:** this is a first draft
written from the documented environment. Several of the things a business
continuity plan needs — maximum tolerable downtime per business function,
an out-of-band contact list, who has authority to declare an emergency —
do not exist anywhere in the library and cannot be invented by IT. They
are marked `[FILL IN]` and they need organization leadership to settle them. A plan
with those blanks is still worth having: it is the agenda for the meeting
that fills them in.

## Quick Facts

| Field            | Value                                                       |
|------------------|-------------------------------------------------------------|
| Owner            | [FILL IN: business continuity is not an IT-owned document. IT owns the technical recovery; the plan itself needs an owner in organization leadership. The sysadmin (it@example.com) owns the IT content until then] |
| Environment      | Organisation-wide — all organization systems, sites and staff          |
| Location         | This file, plus `[FILL IN: a printed copy location. A continuity plan readable only on the systems that are down is not a continuity plan]` |
| Access           | [FILL IN: who holds a copy — IT, Executive Director, [FILL IN: other roles]. Include at least one copy that does not depend on organization infrastructure] |
| Dependencies     | Out-of-band contact list; agreed business impact analysis; delegated decision authority; the technical runbooks in References |
| Dependents       | Every member-facing service; payroll; grievance and arbitration processes |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                     |

## How It Works

### Incident response, disaster recovery, and business continuity are three different things

The library currently conflates them, and the conflation causes real
problems: it leads to plans that restore servers while nobody has told the
members anything, and to incidents where the only person who can make a
decision is the one elbow-deep in a server.

| Discipline | Question it answers | Time horizon | Owner | Document |
|---|---|---|---|---|
| **Incident response** | "What is happening, is it hostile, and how do we contain it?" | Minutes to hours | IT / security | `Security Procedures/IR_Security_Scenarios_Guide.md` |
| **Disaster recovery** | "How do we get the technology back?" | Hours to days | IT | `Hardware & Backup/power-outage-shutdown-runbook.md`, `Hardware & Backup/backup-restore-testing-procedure.md`, and the per-system guides |
| **Business continuity** | "How does the organization keep serving members while the technology is gone?" | Hours to weeks | organization leadership, with IT support | **This document** |

They run **in parallel**, not in sequence. The single most common failure
in a small organisation is treating continuity as something that starts
after recovery fails. By then the members have been waiting two days and
nobody has phoned anyone.

Worked example. Zimbra is down and will be for a day:

- *Incident response* asks whether this is an attack, and isolates if so.
- *Disaster recovery* restores the mailstore
  (`Mail & Messaging/zimbra-mail-administration-guide.md`).
- *Business continuity* decides that member enquiries route to the phones,
  tells staff how to handle them, gets a notice onto the website, and
  makes sure whoever is tracking an arbitration deadline knows their
  confirmation email is not coming.

All three start in the first thirty minutes. Only one of them is about
servers.

### When this plan is activated

This plan is invoked when an outage is **extended** or **organisationally
significant**, not for routine faults. Proposed triggers:

| Trigger | Rationale |
|---|---|
| Any tier 1 system down, or expected to be down, for more than `[CONFIRM: proposed default — 4 hours during business hours]` | Beyond this, staff need to be told what to do instead of waiting |
| Any outage expected to span a working day or more | Manual workarounds become necessary, not optional |
| Loss of the server room or the building, for any duration | Physical loss changes every assumption in the technical runbooks |
| Confirmed ransomware or a compromise affecting multiple systems | Recovery will be measured in days and will involve people outside IT |
| A statutory or contractual deadline is at risk — arbitration, grievance filing, payroll | The deadline does not move because the systems are down |
| Key-person unavailability during any of the above | The plan's delegation provisions are the point |

**Who can activate it:** [FILL IN: name the role, not the person, and name
a deputy. Someone must be able to say "we are in continuity mode" without
waiting for a meeting. In practice this is usually the Executive Director
or equivalent, with IT able to activate for technical events and inform
immediately after.]

### Roles during an extended outage

Small organisations do not have separate people for each of these. One
person may hold several. What matters is that each role is **named in
advance** and has a **named alternate**, because the realistic scenario
includes somebody being unreachable.

| Role | Responsibility | Primary | Alternate |
|---|---|---|---|
| Continuity lead | Declares activation; decides priorities; owns the decision to invoke workarounds | [FILL IN] | [FILL IN] |
| Technical lead | Directs recovery; owns the technical runbooks | [FILL IN] | [FILL IN — this blank is itself a finding; see key-person scenario] |
| Communications lead | Staff notifications, member-facing messaging, website notice | [FILL IN] | [FILL IN] |
| Member services lead | Runs manual workarounds for member-facing functions | [FILL IN] | [FILL IN] |
| Finance / payroll lead | Owns payroll continuity decisions and any manual pay process | [FILL IN] | [FILL IN] |
| Scribe | Keeps the timeline: what happened, when, who was told | [FILL IN] | [FILL IN] |

The scribe role looks optional and is not. Everything afterwards — the
report to leadership, the insurance claim, the fixed procedure, the
measured recovery time — comes from a contemporaneous timeline that nobody
remembers to keep unless it is somebody's job.

---

# BUSINESS IMPACT ANALYSIS

## How to read this table

This is the core of the plan and it is currently empty. That is
deliberate, and the emptiness is itself the finding.

**These figures must be set by organization leadership, not by IT.** Maximum
tolerable downtime is a statement about the organization's obligations to its
members — how long can grievance intake stop before a deadline is missed,
how long can payroll be late — and IT is not in a position to answer it.
What IT contributes is the *capability* side: what the current
infrastructure can actually deliver, measured rather than assumed.

The two must then be compared. Where the tolerable downtime is shorter
than the achievable recovery time, that is a gap requiring either
investment or an accepted risk, recorded as such.

Definitions, because they are routinely mixed up:

- **MTD (maximum tolerable downtime)** — how long the *business function*
  can be unavailable before the consequences become unacceptable. A
  business number.
- **RPO (recovery point objective)** — how much data loss is acceptable,
  expressed as time. Answered by backup frequency.
- **RTO (recovery time objective)** — how quickly IT commits to restoring
  the service. Must be shorter than MTD, and must be *achievable* —
  measured by `Hardware & Backup/backup-restore-testing-procedure.md`, not
  guessed.

> **Do not copy the RPO/RTO table from
> `SysAdmin Procedures/Backup_DR_Runbook.txt`.** That file is unfilled
> template boilerplate throughout — `example.com`, `Jane Smith`,
> `10.0.0.10`, hosts called `web01` and `fileserver`. Its RPO and RTO
> figures are the template's illustrative defaults. They were never agreed
> with anyone in the organization and they describe no real system. **The absence of real
> RPO/RTO figures is one of the most significant findings of the library
> review, and papering over it with template numbers would be worse than
> leaving it blank.**

## The analysis

| System | Business function it serves | Max tolerable downtime | RPO | RTO | Manual workaround |
|---|---|---|---|---|---|
| Dynamics GP | [FILL IN: financials, accounts payable, general ledger — confirm scope] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| DMS / erp-db.example.com | [FILL IN: member records, dues, grievance tracking — confirm what DMS holds and which staff depend on it daily] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Payroll (GP + DMS) | [FILL IN: staff payroll, and whether any member-facing payment depends on it] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Zimbra — mail.example.com | Staff and member correspondence; the record of commitments and deadlines | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| files.example.com (Synology) | [FILL IN: departmental shares — grievance and arbitration files, negotiations material] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| AD — auth2.example.com / auth4.example.com | Staff authentication; file, mail and workstation access depend on it | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| sign.example.com | [FILL IN: document signing — which processes require it, and are any legally time-bound?] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| db01.example.com / db02.example.com | [FILL IN: which applications and reports depend on this MySQL pair] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| prod01.example.com | [FILL IN: role] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| forums.example.com | [FILL IN: member discussion — who uses it and how time-sensitive is it?] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| app-db01.example.com | [FILL IN: application and business function] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| app-db02.example.com | [FILL IN: application and business function] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| graylog01.example.com | Centralised logging — an IT dependency, not a member-facing one | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Internet connectivity | Everything externally facing, plus remote staff | [FILL IN] | n/a | [FILL IN] | [FILL IN] |
| Telephony | [FILL IN: what the phone system is and who provides it — see `Telephony & Conferencing/`. Phones are the fallback for most other outages, so their own resilience matters more than usual] | [FILL IN] | n/a | [FILL IN] | [FILL IN] |
| Staff workstations | Individual productivity | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Server room / building access | Physical prerequisite for most recovery | [FILL IN] | n/a | [FILL IN] | [FILL IN] |

### Filling this in

Work function-first, not system-first. Ask each department: *"If this
stopped working at 9am Monday, when does it start causing real harm, and
what is the first thing that gets missed?"* The answers are usually more
specific and more useful than a number — "we'd be fine until Wednesday,
but the arbitration bundle is due Friday and it has to be assembled from
the arbitration share" tells you exactly what to protect.

Then map functions to systems. A function usually depends on several
systems, and the MTD of the function sets the target for every system
under it.

[CONFIRM: proposed default — the business impact analysis is reviewed
annually and after any significant change to the organization's systems or obligations.]

---

# SCENARIO PLAYBOOKS

Each playbook assumes the technical runbooks are being followed in
parallel by IT. What is written here is the organisational half.

---

## Scenario 1 — Extended power loss

**Technical runbook:** `Hardware & Backup/power-outage-shutdown-runbook.md`
— follow it for the shutdown decision, the shutdown order, and the
bring-up order. Do not duplicate that work here.

**What makes it a continuity event:** the shutdown runbook ends when
everything is safely off. Continuity starts there, because at that point
The organization has no systems and an unknown restoration time.

**First 30 minutes**

- [ ] IT executes the shutdown decision per the runbook. That decision is
      time-critical and is **not** delegated upward — a controlled shutdown
      cannot wait for a meeting.
- [ ] Continuity lead is informed by phone, not email. Mail may already be
      down and will be down shortly regardless.
- [ ] Establish the expected duration from [FILL IN: utility provider /
      building management], and treat it as unreliable.

**First 2 hours**

- [ ] Notify all staff through the out-of-band channel: systems are down,
      expected duration, what to do meanwhile. See Communications below.
- [ ] Decide whether staff stay, go home, or work from elsewhere. This
      depends on whether **internet and phones** survive the outage
      independently — [FILL IN: are the phone system and the internet
      circuit on the UPS or on separate power? If the phones die with the
      servers, the fallback channel for every other scenario dies too.]
- [ ] Identify deadlines falling within the expected outage window and
      decide, per deadline, whether to seek an extension now. Asking early
      is straightforward; asking after the deadline is not.
- [ ] Put a notice on [FILL IN: the public website or a status page — is
      it hosted externally, and can it be updated when organization infrastructure
      is down? If the website is self-hosted it is unavailable in exactly
      the situation it is needed.]

**Beyond a day**

- [ ] Relocate staff to [FILL IN: alternate work location — a second
      office, a rented space, or home working]. Home working requires
      internet and VPN, which requires the server room — check the
      dependency before promising it.
- [ ] Invoke the manual workarounds below for member-facing functions.
- [ ] Re-assess twice daily and communicate on a fixed schedule even when
      there is no news. "No change, next update at 4pm" is a message.

**On restoration:** IT follows the bring-up order and dependency gates in
the power runbook. Do **not** tell staff systems are available until the
gates have passed — a premature all-clear produces a second outage and
destroys confidence in the next notification.

---

## Scenario 2 — Loss of the server room

This is extended power loss without a restoration time, plus the loss of
the hardware. Fire, flood, prolonged building inaccessibility, structural
damage, or an extended cooling failure.

**What changes:** every technical runbook in the library assumes the
hardware exists and can be reached. Here it cannot. Recovery is
**rebuild plus restore**, and the constraint is the offsite backup copy,
the replacement hardware, and somewhere to put it.

**Immediate**

- [ ] Confirm nobody is at risk. Everything below is subordinate to that.
- [ ] Do not attempt entry to a room that facilities or emergency services
      have not released.
- [ ] Continuity lead activates this plan formally and informs the
      [FILL IN: board / executive] per the escalation path.
- [ ] Notify [FILL IN: insurer] — most policies have prompt-notification
      requirements, and the claim will need the scribe's timeline.

**Assess what survives**

| Asset | Question | Where the answer lives |
|---|---|---|
| Offsite backup media | Are the offsite backup disks in the partner-site safe intact and retrievable? | `Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md`; [FILL IN: who has safe access and what the retrieval lead time is] |
| Onsite backup media | Almost certainly lost with the room — it was in the room | — |
| Synology snapshots | Lost with the NAS. Snapshots are on the same volume as the data | `Hardware & Backup/synology-nas-administration-guide.md` |
| Cloud/SaaS services | Unaffected. [FILL IN: which organization services are hosted externally? Microsoft 365 is present in the library — confirm what runs there, because that is what keeps working] | `Microsoft 365/m365-entra-admin-guide.md` |
| Configuration records | Can the environment be rebuilt from documentation? Today: only partly | This library — see the warning below |

> **The honest assessment.** The organization could not currently rebuild its
> environment from documentation alone. `SysAdmin Procedures/Inventory_Asset_Reference.txt`
> — the intended source of truth for hosts, VLANs, DNS and certificates —
> is unfilled boilerplate, and most of the new guides carry `[FILL IN]`
> markers precisely where a rebuild would need the answer. **Filling in the
> asset inventory is the single highest-value continuity investment
> available**, and it costs nothing but time.

**Rebuild sequence** — follow recovery prioritisation below. The realistic
shape:

1. Restore communications first: mail and phones, however temporarily.
   [FILL IN: is there a path to stand up mail quickly elsewhere — a
   temporary forward, a hosted service? Decide before it is needed.]
2. Obtain replacement hardware or hosting. [FILL IN: is there a vendor
   relationship for emergency hardware? What is the realistic lead time?
   This number usually dominates the whole recovery.]
3. Restore AD, then storage, then databases, then applications — the
   dependency order in the power runbook's bring-up table, which does not
   change because the hardware is new.
4. Restore data from the offsite Retrospect media.

**Continuity meanwhile:** assume weeks, not days. Every member-facing
function runs on the manual workarounds below for the duration.

---

## Scenario 3 — Loss of internet connectivity

Internal systems are healthy; the connection to the outside world is not.

**Immediate triage** — establish which of three cases this is, because
they have different owners and different durations:

```
ping -c 4 [FILL IN: ISP gateway IP]        # is the circuit up to the provider's edge?
ping -c 4 1.1.1.1                          # routing works, DNS aside
nslookup example.com 1.1.1.1                 # external resolution
nslookup files.example.com [FILL IN: internal DNS IP]   # internal resolution unaffected?
```

| Case | Symptom | Owner |
|---|---|---|
| Circuit down | No path to the provider edge | [FILL IN: ISP, account number, support number, contracted response time] |
| Firewall/router failure | Circuit up, nothing passes | IT — `Networking Guide/` |
| DNS failure | Everything up, names do not resolve | IT — `Networking Guide/dns-dhcp-administration-guide.md` |

**Continuity impact**

| Still works | Does not work |
|---|---|
| Internal file shares, internal applications, staff logins | Inbound and outbound mail |
| Phones, **if** the phone system is not VoIP over the same circuit — [FILL IN: confirm] | Remote/VPN staff access |
| Anything already on a workstation | Member access to any organization-hosted web service |

**Actions**

- [ ] Confirm whether mail is queuing or bouncing. **Queuing is survivable;
      bouncing loses member correspondence.** [FILL IN: does the organization have a
      backup MX, and what is its queue-and-forward behaviour during an
      extended outage?] This is the single most important question in this
      scenario and it deserves an answer before the next outage.
- [ ] Route member contact to phones; brief reception on the message.
- [ ] Tether or use a mobile hotspot for the small number of externally
      dependent tasks that genuinely cannot wait. [CONFIRM: proposed
      default — keep at least one mobile hotspot or a documented tethering
      arrangement available for continuity use, and test it annually.]
- [ ] For an outage beyond `[CONFIRM: proposed default — 4 hours]`,
      consider a secondary connection. [FILL IN: is there a second circuit
      or a failover path? If not, that is a single point of failure worth
      pricing.]
- [ ] Remote staff: tell them explicitly. Otherwise they spend the day
      assuming their own connection is broken.

---

## Scenario 4 — Loss of mail (mail.example.com / Zimbra)

**Technical runbook:** `Mail & Messaging/zimbra-mail-administration-guide.md`
(12-step outage triage). Restore detail:
`Hardware & Backup/backup-restore-testing-procedure.md`.

Mail deserves its own playbook because it is both a critical service and
**the channel every other plan assumes for notification.** When mail is
the thing that is down, the notification plan must not use it.

**First 30 minutes**

- [ ] Establish whether mail is *down* or *undelivered*. Users cannot tell
      the difference and will report both identically.
- [ ] Establish whether inbound mail is **queuing or bouncing**. Queued
      mail arrives late; bounced mail is gone, and a bounced grievance
      submission is a real harm, not an inconvenience.
- [ ] Notify staff **by phone or in person**. Not by email.

**Continuity actions**

- [ ] Publish an alternative contact route for members: the main phone
      number, and [FILL IN: is there an alternative externally hosted
      address — a Microsoft 365 mailbox, for example — that could receive
      member mail during a Zimbra outage? Standing one up in advance costs
      little and is worth far more during an outage than during a meeting].
- [ ] Brief reception and member services on what to say and how to record
      enquiries that would normally arrive by email. Paper or a shared
      document on files.example.com — which is still up in this scenario.
- [ ] Identify deadlines depending on email confirmation. Contact the
      counterparty by phone and confirm in writing later. **A deadline
      missed because our mail was down is still a missed deadline.**
- [ ] Warn staff not to switch to personal email for organization business. It is
      a privacy and records-management problem that outlasts the outage by
      years. If there is genuinely no alternative, the continuity lead
      authorises it explicitly and it is logged.

**On restoration**

- [ ] Confirm queued mail has been delivered, not discarded.
- [ ] Confirm outbound mail is being accepted by recipients — a restored
      mail server that fails DKIM or SPF will be silently spam-filtered.
      See `Mail & Messaging/email-authentication-spf-dkim-dmarc.md`.
- [ ] Tell staff mail is back, and tell them explicitly whether anything
      was lost. They need to know whether to follow up.

---

## Scenario 5 — Loss of DMS / Dynamics GP (erp-db.example.com)

These are the most business-critical systems the organization runs: payroll and member
data. An extended outage here is the scenario most likely to cause
material harm, and it is the one with the fewest workarounds.

**Technical runbooks:** `Dynamics GP & DMS/dynamics-gp-2018-admin-guide.md`,
`Databases/mysql-administration-guide.md`, and the restore testing
procedure.

**Immediate**

- [ ] Determine the scope precisely. The distinctions matter and have
      different answers:
      - Database down vs. application/web tier down — the DMS web tier has
        its own documented failure mode (the DMS web tier Apache
        crash runbook). An
        Apache crash is not a data problem and is fixed in minutes.
      - Data intact vs. data corrupt — corruption means a restore and an
        RPO decision.
      - Connectivity vs. data — an ODBC/TLS failure looks like a dead
        database. See the connector and `require_secure_transport` notes in
        the MySQL guide.
- [ ] Notify the finance/payroll lead and the member services lead
      immediately, regardless of expected duration. They may need to start
      a manual process whose lead time is longer than the outage.

**The payroll question**

This needs deciding in advance, not during the outage:

- [ ] [FILL IN: when is payroll run, and what is the latest point at which
      a run can start and still pay staff on time?] That deadline, minus
      the measured restore time for GP, is the real decision point.
- [ ] [FILL IN: is there a manual or externally-assisted payroll path —
      the bank's own portal, a payroll bureau, or repeating the previous
      period's run and reconciling afterwards? Who authorises it?]
- [ ] [FILL IN: what does the organization's bank require, and what is the cut-off for
      a manual submission?]

**Member data continuity**

- [ ] [FILL IN: is there any current export of member data — a report, an
      extract, a read-only copy — that staff could consult read-only
      during an outage? A daily read-only extract to files.example.com would be
      a cheap and disproportionately useful continuity control.]
- [ ] During an outage, member enquiries requiring DMS lookup are logged
      for callback rather than answered from memory. Record them
      consistently so nothing is lost when the system returns.
- [ ] Grievance and arbitration deadlines tracked in DMS: [FILL IN: is
      there a secondary record of deadlines — a calendar, a spreadsheet,
      paper? If the only record of a statutory deadline is inside DMS, that
      is a serious single point of failure and it is cheap to fix.]

**On restoration**

- [ ] Re-enter everything captured manually during the outage, and
      reconcile. Assign this explicitly to someone — it is invisible work
      that otherwise does not happen.
- [ ] Verify the data against the last known-good reference point before
      re-opening the system to users.

---

## Scenario 6 — Ransomware

**Do not improvise this and do not duplicate the response here.**

**Technical response:** `Security Procedures/IR_Security_Scenarios_Guide.md`
section 6, "Ransomware Response" — containment in the first 60 seconds,
evidence preservation, strain identification. Section 5 covers the
compromised account path, and `Security & Hardening/endpoint-security-operations-guide.md`
covers the endpoint side.

> [FILL IN: the organization does not currently have a dedicated ransomware playbook —
> the IR scenarios guide is a field reference of commands, not an
> organisational playbook covering legal notification, member data breach
> obligations, insurance, and the ransom decision. That gap should be
> closed, and when it is, this section should point at it instead.]

**What this plan adds — the organisational half:**

- [ ] **Assume an extended outage from the outset.** Ransomware recovery is
      measured in days to weeks. Activate continuity immediately rather
      than waiting to see how recovery goes.
- [ ] **Recovery depends on backups the attacker could not reach.** For
      the organization that means the offsite Retrospect media in the partner-site safe, and
      locked/immutable Synology snapshots if enabled ([FILL IN: snapshot
      locking status is flagged as unconfirmed in the NAS guide]). Onsite
      backups reachable from a compromised network must be assumed
      compromised until proven otherwise.
- [ ] **Do not restore into the compromised environment.** Rebuilding
      clean and restoring data into it is slower and is the only approach
      that does not reinfect.
- [ ] **Member and staff data breach obligations are a legal question, not
      an IT one.** [FILL IN: who provides the organization's legal advice, and what are
      the privacy-commissioner and member-notification obligations for a
      breach involving member or payroll data? Establish this before an
      incident — the notification clock starts at discovery.]
- [ ] **The ransom decision is not IT's.** [FILL IN: who decides, and has
      the position been agreed in advance? Deciding under pressure, on the
      day, is how organisations make decisions they regret.]
- [ ] **Notify the insurer early.** [FILL IN: does the organization carry cyber
      insurance? Many policies require notification before recovery steps
      are taken and may direct the response.]
- [ ] **Communications will be scrutinised.** Members and possibly media
      will ask. Say only what is known. See Communications below.

---

## Scenario 7 — Key-person unavailability

**This scenario is the most likely of all of them**, and unlike the others
it has no technical mitigation at all.

The organization's systems are administered by one person. Illness, leave, family
emergency, a departure, or simply being on a plane makes every other
playbook in this document harder, and makes a few of them impossible as
written.

**The current state, stated plainly.** Almost every guide in this library
names the same single sysadmin as owner, and the "escalation if the owner is
unavailable" line in each of them is `[FILL IN]`. There is currently no
documented alternate for any IT system. That is the finding.

**What must exist before it is needed** — none of it can be arranged
during the event:

| Need | Status | Action |
|---|---|---|
| Credential access | [FILL IN: is there break-glass access to 1Password? Who holds it, and has it been tested?] | `Security & Hardening/secrets-management-guide.md` — an emergency access procedure is the highest-priority item in this table |
| A named technical alternate | Absent | [FILL IN: an internal second person, or a retained external provider with an agreed response time?] |
| Physical access | [FILL IN: who else can enter the server room and the offsite-media safe?] | Physical keys and codes, held where |
| Vendor and provider contacts | Scattered across documents | The out-of-band contact list below |
| Documentation good enough to follow | Partial — this library, with `[FILL IN]` markers marking exactly where an alternate would be stuck | Filling in the markers is the mitigation |
| Domain, DNS and certificate control | [FILL IN: where is the organization's domain registered, who is the registrant contact, and who else can access it? A lapsed domain with no reachable contact is an organisation-level emergency] | |

**During the event**

- [ ] Continuity lead confirms expected duration, and whether the person
      can be reached for advice even if not to work.
- [ ] Freeze non-essential change. Do not patch, upgrade, or rearrange
      anything during a period when nobody can fix the consequences.
      Security-critical patching is the exception and is a judgement call.
- [ ] Escalate to the external provider if one is retained. [FILL IN.]
- [ ] Keep a written log of everything done in the person's absence.
      Undocumented changes made by a stand-in are the second incident.

**Standing mitigations** — cheap, and worth doing regardless:

- [CONFIRM: proposed default — a documented break-glass credential
  procedure, tested annually, with the test recorded.]
- [CONFIRM: proposed default — at least one other person walks the power
  outage runbook and the restore procedures once a year, as the
  non-owner tester described in
  `Hardware & Backup/backup-restore-testing-procedure.md`. That exercise
  simultaneously tests the documentation and the succession plan.]
- [CONFIRM: proposed default — a retained relationship with an external IT
  provider for emergency cover, with an agreed response time, even if it
  is used for nothing else.]

---

# COMMUNICATION PLAN

## The core problem

**Email is the normal channel, and email is frequently the thing that is
down.** Any communication plan routed through mail.example.com fails in a large
share of the scenarios above. The power outage runbook already flags this
as its critical gap. It remains unresolved.

The requirement is an **out-of-band channel** that depends on nothing the organization
operates: personal mobile numbers, SMS, a phone tree, or an externally
hosted status page.

## Notification order

Notify in this order. Do not wait for one to complete before starting the
next — but do not start broad staff communications before the continuity
lead knows, or they will hear about it from someone else.

| # | Who | When | Channel (primary → fallback) | Content |
|---|---|---|---|---|
| 1 | Technical lead | Immediately on detection | Phone → SMS | What is down, what is known |
| 2 | Continuity lead | Within 15 minutes of a suspected extended outage | Phone → SMS | Scope, expected duration, recommendation to activate |
| 3 | Department leads | Within 30 minutes | Phone / SMS → in person | What is unavailable, which workarounds to start |
| 4 | All staff | Within 1 hour | [FILL IN: SMS list / phone tree / in person] | What is down, what to do, when the next update comes |
| 5 | Members (if member-facing) | As soon as the message is accurate | [FILL IN: website notice / phone greeting / social media — which of these can be updated when organization systems are down?] | How to reach the organization meanwhile |
| 6 | [FILL IN: board / executive] | Per the escalation threshold | Phone | Impact, expected duration, decisions needed |
| 7 | External parties with pending deadlines | As soon as a deadline is at risk | Phone, confirmed in writing later | Specific, per deadline |
| 8 | All staff — resolution | When verified, not when hoped | Same channel as #4 | Back up, anything lost, what to check |

**Update on a fixed schedule even with no news.** `[CONFIRM: proposed
default — every 2 hours during business hours for an active incident, and
at the start and end of each day for a multi-day event.]` Silence is read
as either "nothing is happening" or "it is worse than they are saying",
and both are corrosive.

## Message templates

Keep them short and concrete. Staff need three things: is it down, what do
I do now, when will I hear more.

```
INITIAL (staff)
  [System] is unavailable as of [time]. We are working on it.
  Meanwhile: [the one thing they should do differently].
  Next update by [time].
  Do not reply to this message — [mail] is affected.

UPDATE (staff)
  [System] is still unavailable. [What we now know, in one sentence.]
  Current estimate: [time, or "not yet known" — do not invent one].
  Next update by [time].

RESOLUTION (staff)
  [System] is back as of [time]. Please check [the specific thing to check].
  [Anything lost, stated plainly — or "no data was lost".]
  Report anything still not working to [contact].

MEMBER-FACING
  [Organization]'s [service] is temporarily unavailable. To reach us, please call
  [number] between [hours]. We expect to restore service by [time / "as
  soon as possible" — never a guess].
```

**Never promise a restoration time you do not have.** "We will update you
by 2pm" is always better than a guess that turns out wrong, and it costs
nothing to keep.

## Out-of-band contact list

**This list must exist in printed form**, held by the continuity lead and
the technical lead, and stored somewhere reachable when the building is
not. A contact list on files.example.com is useless in most of these scenarios.

It holds personal contact details, so it is confidential and is
distributed only to the roles that need it. [FILL IN: confirm with staff
that their personal numbers may be held for this purpose — consent matters
and is quick to obtain in advance.]

### Internal

| Role | Name | Work phone | Mobile | Personal email | Notes |
|---|---|---|---|---|---|
| Continuity lead | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Authority to activate this plan |
| Continuity lead (alternate) | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| Technical lead | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| Technical lead (alternate) | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Currently absent — see scenario 7 |
| Communications lead | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| Member services lead | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| Finance / payroll lead | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| [FILL IN: each department lead] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |

### External

| Contact | Purpose | Account / reference | Phone | Hours / response time |
|---|---|---|---|---|
| [FILL IN: ISP] | Circuit outage | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: utility provider] | Power outage and restoration estimate | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: building management / facilities] | Building access, cooling, generator | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: electrician] | Electrical faults | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: APC / Schneider support] | UPS faults, battery modules | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: hardware vendor / reseller] | Emergency replacement hardware | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: external IT provider, if retained] | Emergency cover — scenario 7 | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: telephony provider] | Phone system — the fallback channel | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: domain registrar] | Organization domain and DNS control | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: insurer] | Property and cyber claims | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: legal counsel] | Breach obligations, member notification | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: bank] | Manual payroll submission | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: offsite-media safe access] | Offsite backup media retrieval | [FILL IN] | [FILL IN] | [FILL IN] |

---

# MANUAL WORKAROUNDS FOR MEMBER-FACING SERVICES

The test of a continuity plan is whether a member who calls during an
outage gets help. These are the fallbacks. Each needs confirming with the
department that would actually operate it — a workaround IT invented and
nobody has practised is not a workaround.

| Function | Normal system | Manual workaround | Prepared in advance? |
|---|---|---|---|
| Member enquiry by phone | DMS lookup | Log the enquiry on a paper or offline form, call back once systems return. Answer only what is known without lookup | [FILL IN: is there a standard offline enquiry form? Create one — it takes an hour and removes all improvisation on the day] |
| Member enquiry by email | Zimbra | Phone line; [FILL IN: alternate externally hosted address] | [FILL IN] |
| Grievance intake | [FILL IN: DMS? email? a form?] | Paper intake with a strict log; enter into DMS on restoration | [FILL IN: a printed grievance intake form held offline] |
| Arbitration / deadline tracking | [FILL IN: DMS? a calendar?] | [FILL IN: a secondary offline record of upcoming deadlines. If the only record lives in DMS, an extended DMS outage puts statutory deadlines at risk] | [FILL IN — high priority] |
| Document signing | sign.example.com | Wet signature, scan later; or defer if not urgent | [FILL IN: is a wet signature acceptable for the documents normally signed here?] |
| Access to grievance / arbitration files | files.example.com | [FILL IN: are critical current files also held locally on a workstation, or is the share the only copy? For an in-progress arbitration bundle this matters a great deal] | [FILL IN] |
| Payroll | Dynamics GP | [FILL IN: manual bank submission or bureau — see scenario 5] | [FILL IN] |
| Member communications / notices | [FILL IN: website, forums, mail] | Phone greeting; [FILL IN: externally hosted status page or social media] | [FILL IN] |
| Staff internal coordination | Mail, shares | In-person standups; SMS group | Cheap, and works |

Two principles that apply to all of them:

1. **Capture consistently, and capture everything.** Manual records have
   to be re-entered. Inconsistent capture makes reconciliation slow and
   error-prone, and reconciliation is where errors reach member records.
2. **Assign the re-entry before the outage ends.** Name the person while
   the outage is running. Otherwise the paper pile sits for a week.

---

# RECOVERY PRIORITISATION

## The order, and why

Recovery order is set by **dependency first, business impact second**.
Restoring a high-value application before the thing it depends on wastes
the effort and produces confusing failures. The technical dependency order
is already established in the bring-up table of
`Hardware & Backup/power-outage-shutdown-runbook.md`, and it does not
change because the cause is different.

| # | Layer | Why it is here | Gate before continuing |
|---|---|---|---|
| 0 | People and premises safety | Nothing else matters first | Site is safe and accessible |
| 1 | Network — switching, firewall, internet | Nothing else can be reached without it | Gateway responds; DNS resolves |
| 2 | AD — auth2.example.com, auth4.example.com | Nearly everything authenticates against it, including the NAS domain join | A test domain login succeeds |
| 3 | Storage — files.example.com | Shares and any storage-backed guests | Volumes healthy; shares reachable |
| 4 | Hypervisors — Proxmox | Hosts the guests | Cluster quorate; storages active |
| 5 | Databases — erp-db.example.com, db01.example.com, db02.example.com | Applications need them first | Engines accept connections; startup log clean, no unexplained crash recovery |
| 6 | Mail — mail.example.com | Both a business function and the notification channel | Send and receive both verified |
| 7 | Business applications — Dynamics GP, DMS, sign.example.com, prod01 | The member-facing work | A real user journey works end to end |
| 8 | Supporting services — forums, app-db01, app-db02, graylog01 | Important, not urgent | — |
| 9 | Workstations and user access | Users return to normal working | Login and shares work |

**Where business impact overrides the order:** within a layer. If both
DMS and sign.example.com are down at layer 7 and payroll runs tomorrow, DMS
goes first. Between layers, dependency wins — bringing DMS up before its
database is not a priority decision, it is a mistake.

**Mail is deliberately early**, before the business applications. It is
the channel through which everything else is coordinated and through which
members are told what is happening. An organisation with mail and no
applications can communicate; the reverse cannot.

## Partial recovery

An extended outage rarely restores everything at once, and partial
availability is its own hazard.

- **Be explicit about what is back.** "Mail is back, DMS is not" is a
  usable message. "Systems are coming back" produces a wave of support
  calls from people whose system is not.
- **Do not re-open a system to users until its verification gate has
  passed.** A system that is up but wrong is worse than one that is down,
  because it puts bad data in front of people making decisions.
- **Sequence the re-entry of manually captured work** so that it does not
  collide with a system still stabilising.

---

# PLAN TESTING

**An untested plan is a document, not a capability.** The same logic that
makes an untested backup a rumour applies here: a plan nobody has walked
through contains steps that do not work, contacts that have moved on, and
assumptions nobody has noticed.

## Tabletop exercises

A tabletop is a facilitated discussion, not a technical drill. Nothing is
shut down. The value is in discovering, calmly, which decisions nobody can
make and which contacts nobody has.

**Format** — 90 minutes, `[CONFIRM: proposed default — annually, with a
shorter 45-minute review after any significant change to systems or
staffing]`:

| Time | Activity |
|---|---|
| 0:00-0:10 | Facilitator sets the scenario. Participants do not see it in advance |
| 0:10-0:40 | **First 30 minutes** — each role says what they would actually do, in order. The facilitator writes down every "I'd need to ask…" and "I don't know where that is" |
| 0:40-1:00 | **Inject a complication.** The technical lead is unreachable. The outage is ransomware, not hardware. The deadline is tomorrow, not next week |
| 1:00-1:20 | **Day two** — what does the organisation look like tomorrow morning if this is not fixed? |
| 1:20-1:30 | Findings and actions, with owners and dates |

**Participants:** continuity lead, technical lead, communications lead,
member services lead, finance lead, and at least one person who would
actually answer the phones. `[CONFIRM: proposed default — include a
department representative who is not in IT management, because they ask
the questions the plan's authors have stopped seeing.]`

**Scenarios, rotating** — use a different one each time:

1. Extended power loss, restoration unknown, on a Thursday before payroll.
2. Ransomware discovered on a Monday morning; several shares are encrypted.
3. Zimbra down for 24 hours with an arbitration deadline on Friday.
4. The technical lead is unreachable and files.example.com has a failed volume.
5. Building inaccessible for two weeks.

**Rules that make it useful:**

- **No fixing during the exercise.** Write the gap down and move on.
  Solving one problem in the room consumes the whole session.
- **"I would call the sysadmin" is a finding, not an answer.** Push past it.
- **Every action gets an owner and a date**, or the exercise was theatre.
- **The scenario is not the point.** What the discussion reveals about
  authority, contacts and workarounds is.

## Other testing

| Test | What it proves | Cadence |
|---|---|---|
| Tabletop exercise | Decisions, roles, communications | `[CONFIRM: proposed default — annually]` |
| Out-of-band contact test | The phone tree / SMS list actually reaches people | `[CONFIRM: proposed default — semi-annually; a no-warning test at a sensible hour]` |
| Restore testing | Backups can be recovered, and how long it takes | `Hardware & Backup/backup-restore-testing-procedure.md` |
| Power shutdown walk-through | The shutdown order works and how long it takes | `Hardware & Backup/power-outage-shutdown-runbook.md` (annual) |
| Non-owner recovery test | The documentation is sufficient without its author | `[CONFIRM: proposed default — annually, and it is the strongest single mitigation for scenario 7]` |

Record every exercise in Decisions & History below, including what it
found. **An exercise that found nothing was run wrong.**

---

## Operations (Day-2)

### Plan maintenance

A continuity plan decays faster than technical documentation, because it
depends on people and phone numbers rather than on configuration.

**Review triggers — any of these, not just the calendar:**

- [ ] Annually, at minimum. `[CONFIRM: proposed default — reviewed each
      [FILL IN: month], alongside the business impact analysis.]`
- [ ] After any staff change affecting a named role in this plan. A
      contact list with a departed employee in it is worse than none,
      because it creates false confidence.
- [ ] After any significant system change — a new business-critical
      system, a migration to or from hosted services, a change of ISP or
      telephony provider.
- [ ] After any real incident. The incident is free testing; use it.
- [ ] After any tabletop exercise.
- [ ] After any change to the organization's obligations that changes the business
      impact analysis.

**Review checklist:**

- [ ] Every name and number in the contact lists verified by contacting
      them, not by assuming.
- [ ] Role assignments and alternates still correct, and the alternates
      know they are alternates.
- [ ] Business impact analysis still reflects what the business says.
- [ ] Measured RTOs from restore testing still meet the agreed targets.
      Where they do not, the gap is recorded and escalated rather than
      ignored.
- [ ] Scenario playbooks reflect the current environment.
- [ ] The printed copies are current. **Replace them; do not assume.**
- [ ] `[FILL IN]` count has gone down since the last review. If it has
      not, the review did not happen.

### After any activation

- [ ] Debrief within `[CONFIRM: proposed default — five working days]`,
      while people still remember.
- [ ] Record the actual timeline from the scribe's log — especially the
      real restoration times, which are the best data available.
- [ ] Update this plan and the technical runbooks with what was wrong.
- [ ] Report to [FILL IN: board / executive]: what happened, impact,
      duration, what is being changed.

## Troubleshooting

Failure modes of the plan itself. These are the ways continuity plans go
wrong in practice.

### Symptom: nobody is sure whether the plan is active

- Cause: activation authority was never assigned, or the threshold is
  vague.
- Fix now: the most senior person available declares it and informs
  upward. **An unnecessary activation costs a meeting; a missed one costs
  a day.**
- Fix later: fill in the activation authority and deputy above.

### Symptom: staff found out from a member, not from the organization

- Cause: the notification chain started with email, which was down, or
  waited for certainty before saying anything.
- Fix: send the initial message before the cause is known. "X is down, we
  are investigating, next update at [time]" is complete and correct.

### Symptom: the contact list did not work

- Cause: numbers were stale, or the list was stored on an unavailable
  system.
- Fix now: work from personal knowledge and physical proximity.
- Fix later: the semi-annual contact test exists for exactly this, and the
  printed copy requirement exists for the storage half.

### Symptom: recovery is technically complete but the business is still stopped

- Cause: manually captured work has not been re-entered, or staff were not
  told the system is usable, or a dependent process is still waiting.
- Fix: the recovery is not finished at the technical gate. It is finished
  when the manual backlog is cleared and the people doing the work say so.

### Symptom: the same finding appears in every exercise

- Cause: actions are raised without owners or dates.
- Fix: an exercise finding with no owner is not a finding. Assign it in
  the room.

## Security

- **This plan contains personal contact details.** Treat it as
  confidential and distribute it by role. The printed copies need a
  controlled location, and superseded copies are destroyed rather than
  left in a drawer.
- **No credentials in this document, ever** — including in the out-of-band
  contact list. Break-glass credential access is a separate, controlled
  procedure: `Security & Hardening/secrets-management-guide.md`.
- **Continuity workarounds must not create security incidents.** Personal
  email for organization business, unencrypted member data on a personal device,
  or a temporary share with open permissions all outlast the outage. Any
  such measure is authorised explicitly by the continuity lead, logged,
  and reversed on restoration.
- **A disaster is a social-engineering opportunity.** During a publicly
  visible outage, expect calls claiming to be vendors, staff who cannot
  log in, or IT support. Identity verification requirements do not relax
  because systems are down — they matter more. See
  `Security & Hardening/helpdesk-social-engineering-awareness_1.txt`.
- Member data handled manually during an outage remains subject to the
  same privacy obligations. Paper records are collected and securely
  destroyed after re-entry.

## Monitoring & Alerting

Continuity depends on knowing an outage has started, which is a monitoring
question. The current state is a gap: per
`SysAdmin Procedures/monitoring-alerting-guide.md`, only Graylog is
documented, and `graylog01.example.com` is itself part of the estate that can go
down.

- [FILL IN: is there any external monitoring — an uptime check from
  outside the organization's network — for the member-facing services? Internal
  monitoring cannot tell you that the organization is unreachable from the internet,
  which is precisely what a member experiences.]
- [FILL IN: how does the technical lead find out about an out-of-hours
  outage today? If the answer is "a staff member notices in the morning",
  that is a continuity finding, not a monitoring one.]
- [CONFIRM: proposed default — an external uptime check on mail.example.com and
  the main website, alerting to a mobile number, independent of the organization's
  infrastructure.] It is the cheapest continuity control available and it
  works when everything else does not.

## Disaster Recovery

The technical recovery procedures this plan depends on:

- `Hardware & Backup/power-outage-shutdown-runbook.md` — shutdown
  decision, shutdown order, bring-up order and dependency gates
- `Hardware & Backup/backup-restore-testing-procedure.md` — proves the
  backups work and produces the measured restore times this plan's RTOs
  must be based on
- `Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md` —
  file and system restore from Retrospect, including the offsite media set
- `Linux & Servers/lxc_backup_restore_proxmox91.txt` and
  `Linux & Servers/proxmox-cluster-administration-guide.md`
- `Databases/mysql-administration-guide.md` — including point-in-time
  recovery
- `Hardware & Backup/synology-nas-administration-guide.md`
- `Mail & Messaging/zimbra-mail-administration-guide.md`
- `Dynamics GP & DMS/dynamics-gp-2018-admin-guide.md`

**The standing gap:** RTO and RPO targets do not exist. Filling in the
business impact analysis above, and comparing it against measured restore
times, is the work that turns this library's recovery procedures into
commitments the organization can actually make to its members.

## Decisions & History (ADR-lite)

| Date | Decision / Change | Why / Ticket |
|---|---|---|
| 2026-09-11 | This plan created | Library review found a power-outage runbook and an IR field guide, but nothing covering extended loss of a system, staff communication, or working without IT |
| 2026-09-11 | Recorded that `SysAdmin Procedures/Backup_DR_Runbook.txt` RPO/RTO figures are template boilerplate and must not be quoted as organization targets | The figures were never agreed and describe no real system |
| [FILL IN] | Business impact analysis agreed with organization leadership | [FILL IN] |
| [FILL IN] | Activation authority and role alternates assigned | [FILL IN] |
| [FILL IN] | Out-of-band contact list established and printed | [FILL IN] |
| [FILL IN] | First tabletop exercise held | [FILL IN] |

## References

- `Hardware & Backup/power-outage-shutdown-runbook.md`
- `Hardware & Backup/backup-restore-testing-procedure.md`
- `Hardware & Backup/Retrospect Restore/Retrospect Restore Procedure.md`
- `Hardware & Backup/synology-nas-administration-guide.md`
- `Linux & Servers/lxc_backup_restore_proxmox91.txt`
- `Linux & Servers/proxmox-cluster-administration-guide.md`
- `Databases/mysql-administration-guide.md`
- `Mail & Messaging/zimbra-mail-administration-guide.md`
- `Mail & Messaging/email-authentication-spf-dkim-dmarc.md`
- `Dynamics GP & DMS/dynamics-gp-2018-admin-guide.md`
- `Security Procedures/IR_Security_Scenarios_Guide.md` — incident response,
  including the ransomware containment steps referenced in scenario 6
- `Security & Hardening/endpoint-security-operations-guide.md`
- `Security & Hardening/secrets-management-guide.md` — break-glass
  credential access, the key dependency of scenario 7
- `Security & Hardening/helpdesk-social-engineering-awareness_1.txt`
- `SysAdmin Procedures/monitoring-alerting-guide.md`
- `SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt` — the
  offboarding half matters for scenario 7 succession
- `SysAdmin Procedures/Inventory_Asset_Reference.txt` — **unfilled
  boilerplate.** Completing it is the highest-value continuity investment
  available, because it is what a rebuild would be done from
- `SysAdmin Procedures/Backup_DR_Runbook.txt` — **unfilled boilerplate.**
  Useful structure; its hosts, addresses, retention periods and RPO/RTO
  figures are template defaults and describe no part of the real environment

---

# APPENDIX — FIRST 30 MINUTES CARD

Print this. Keep a copy with the continuity lead, a copy with the
technical lead, and a copy in the server room next to the power outage
checklist. It is the only part of this plan anyone will read on the day.

```
================================================================
  BUSINESS CONTINUITY — FIRST 30 MINUTES         rev 2026-09-11
  Full plan: SysAdmin Procedures/business-continuity-plan.md
  Continuity lead: [FILL IN]          [FILL IN: mobile]
  Technical lead:  [FILL IN]          [FILL IN: mobile]
  Alternate:       [FILL IN]          [FILL IN: mobile]
================================================================

MINUTE 0-5   ASSESS
  [ ] Is anyone at risk? Safety first, always.
  [ ] WHAT is down?  (one system / several / the site / the network)
  [ ] SINCE when?
  [ ] Any sign this is hostile? -> also open
      Security Procedures/IR_Security_Scenarios_Guide.md
  [ ] Do NOT start fixing before someone has been told.

MINUTE 5-15  NOTIFY  (phone/SMS — assume email is down)
  [ ] Technical lead
  [ ] Continuity lead — recommend activate / not activate
  [ ] Department leads for anything member-facing

  ACTIVATE THIS PLAN IF:
    - a tier 1 system is down > [CONFIRM: 4 hours in business hours]
    - the outage will span a working day
    - the server room or building is lost
    - ransomware or multi-system compromise
    - a grievance / arbitration / payroll deadline is at risk
    - the technical lead is unavailable during any of the above
  WHEN IN DOUBT, ACTIVATE. Standing down is cheap.

MINUTE 15-30  STABILISE AND COMMUNICATE
  [ ] Start the scribe log: time, what happened, who was told.
  [ ] Staff message — do not wait for certainty:
        "[System] is down since [time]. We are working on it.
         Meanwhile [do this]. Next update by [time]."
  [ ] Member-facing? Route enquiries to the phones and brief
      reception on what to say.
  [ ] List deadlines falling inside the likely outage window.
      Phone the counterparty EARLY if one is at risk.
  [ ] Start the manual workarounds — do not wait to see if
      recovery is quick.
  [ ] Set the next update time and keep it.

RECOVERY ORDER  (dependency first — do not skip ahead)
  Safety -> Network -> AD -> Storage -> Hypervisors ->
  Databases -> MAIL -> Business apps -> Supporting -> Workstations
  Business impact decides order WITHIN a layer, never between.

DO NOT
  - Do not notify staff by email when email is the problem
  - Do not promise a restoration time you do not have
  - Do not tell people a system is back before it is verified
  - Do not restore into a compromised environment (ransomware)
  - Do not let the only person who can decide be the one
    fixing the servers
  - Do not skip the scribe log — everything afterwards needs it

TECHNICAL RUNBOOKS
  Power        Hardware & Backup/power-outage-shutdown-runbook.md
  Restores     Hardware & Backup/backup-restore-testing-procedure.md
               Hardware & Backup/Retrospect Restore/
                 Retrospect Restore Procedure.md
  Security     Security Procedures/IR_Security_Scenarios_Guide.md
  Mail         Mail & Messaging/zimbra-mail-administration-guide.md
  NAS          Hardware & Backup/synology-nas-administration-guide.md
  Databases    Databases/mysql-administration-guide.md
================================================================
```

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

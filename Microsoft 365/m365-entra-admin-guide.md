> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Microsoft 365 / Entra ID Administration

Supersedes `_ARCHIVE/superseded-stubs/O365 security policy updates.txt` (archived 2026-09-11) (a 91-byte note pointing at
**Home > Org settings > Baseline security mode**). That navigation path is preserved in the
"Security defaults vs. custom policy" section below. The old stub should be treated as retired.

---

## Overview

Microsoft 365 provides the organization's cloud productivity stack — Office desktop/web apps, OneDrive,
SharePoint, Teams — and Entra ID (formerly Azure AD) is the identity directory behind it.
Identity at Example Org starts on-premises in Active Directory (`example.com`, domain controllers
`auth2.example.com` and `auth4.example.com`), so Entra ID is a hybrid directory: accounts are mastered in
on-prem AD and synchronised up. Primary mail for staff is Zimbra on `mail.example.com`, not Exchange
Online — see `Mail & Messaging/zimbra-mail-administration-guide.md`. Confirm what Exchange
Online is actually used for here before relying on the Exchange section:
[FILL IN: which mailboxes, if any, live in Exchange Online vs. Zimbra — full migration, coexistence, or Office apps only].

Audience: the sysadmin and anyone covering for them. Everything in this doc assumes Global
Administrator or an equivalent scoped role.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Owner            | IT lead (it@example.com), IT                                 |
| Environment      | prod (single tenant)                                                   |
| Location         | Microsoft cloud — admin portals listed under Access                     |
| Access           | https://admin.microsoft.com (M365 admin centre), https://entra.microsoft.com (Entra ID), https://security.microsoft.com (Defender), https://compliance.microsoft.com (Purview). PowerShell: `Microsoft.Graph` and `ExchangeOnlineManagement` modules |
| Dependencies     | On-prem AD (`auth2.example.com`, `auth4.example.com`); the directory sync server [FILL IN: hostname running Entra Connect / cloud sync agent]; public DNS for `example.com`; internet egress through pfSense |
| Dependents       | Office application activation, OneDrive/SharePoint/Teams, any SaaS app federated to Entra ID ([FILL IN: list of apps using Entra SSO]), conditional access enforcement |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                                |

Tenant identifiers — do not guess these, read them from **Entra admin centre > Overview**:

| Item                      | Value                                        |
|---------------------------|----------------------------------------------|
| Tenant name               | [FILL IN: tenant display name]               |
| Tenant ID (GUID)          | [FILL IN: tenant ID from Entra > Overview]   |
| Default/onmicrosoft domain| [FILL IN: e.g. something.onmicrosoft.com]    |
| Verified custom domains   | [FILL IN: verified domains, e.g. example.com and any others] |
| Licence subscriptions     | [FILL IN: subscription names and seat counts from Billing > Your products] |

## How It Works

Accounts originate in on-premises Active Directory. A directory sync agent on
[FILL IN: sync server hostname] reads the AD user objects and writes matching cloud objects into
Entra ID. `UserPrincipalName` is the anchor users log in with — the same attribute already used
for LDAP binds (see `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11)), so a user's sign-in name is consistent between
on-prem and cloud. Sync direction is one-way (AD → cloud) for object attributes; whether password
hash sync, pass-through auth, or federation is in use determines where the password is actually
checked: [FILL IN: sign-in method — PHS, PTA, or AD FS federation — from Entra Connect config].

  on-prem AD (auth2 / auth4.example.com)
      -> directory sync agent on [FILL IN: sync host]   # every 30 min by default
          -> Entra ID tenant
              -> M365 workloads (Office, OneDrive, SharePoint, Teams)
              -> conditional access + MFA evaluated at sign-in

Licences are assigned in the cloud only — a synced user object exists in Entra ID as soon as sync
runs, but has no service access until a licence is attached, ideally by group membership rather
than per user.

Configuration lives in three places and it matters which: on-prem AD and Entra Connect (identity
attributes, sync scope, sign-in method), the Entra admin centre (roles, conditional access,
authentication methods), and the M365 admin centre (licences, org settings, service config).
Audit data lives in the unified audit log in Purview and in the Entra sign-in/audit logs.

---

## Setup / Installation

The tenant already exists; this section covers what must be true for it to keep working, not a
from-scratch build.

- Directory sync is installed on [FILL IN: sync server hostname and OS version]. It is a
  single point of failure for user provisioning; if it is down, new hires and disables do not
  reach the cloud.
- Domain `example.com` is verified in the tenant via a DNS TXT record. If the record is ever removed
  the domain falls out of verification and sign-ins break:
  [FILL IN: the MS domain-verification TXT record value currently published for example.com].
- PowerShell modules used for admin work:

      Install-Module Microsoft.Graph -Scope CurrentUser            # Entra/user/group/licence admin
      Install-Module ExchangeOnlineManagement -Scope CurrentUser   # Exchange Online cmdlets
      Connect-MgGraph -Scopes "User.ReadWrite.All","Directory.ReadWrite.All"   # interactive sign-in, consents scopes
      Connect-ExchangeOnline -UserPrincipalName admin@example.com  # opens modern-auth prompt

## Configuration

### Admin roles and who holds them

Least privilege applies: a person should hold the narrowest role that lets them do the job, and
Global Administrator should be rare. Read the live list before trusting this table —
**Entra admin centre > Roles and administrators > (role) > Assignments**.

| Role                        | Purpose                                              | Held by                                  |
|-----------------------------|------------------------------------------------------|------------------------------------------|
| Global Administrator        | Full tenant control. Keep the count in single digits | [FILL IN: current GA assignees]          |
| Privileged Role Administrator | Can grant roles to others — as dangerous as GA     | [FILL IN: assignees]                     |
| User Administrator          | Create/modify users, reset non-admin passwords       | [FILL IN: assignees]                     |
| Exchange Administrator      | Mailbox and mail-flow admin                          | [FILL IN: assignees]                     |
| Security Administrator      | Defender/Entra security config                       | [FILL IN: assignees]                     |
| Helpdesk Administrator      | Password resets for non-admin users                  | [FILL IN: assignees]                     |
| Global Reader               | Read-only everything — use for audits and handover   | [FILL IN: assignees]                     |
| Break-glass accounts        | Emergency access only, see Security section          | [FILL IN: the two break-glass UPNs]      |

Rules for this environment:
- Admin roles are assigned to the admin's own separate admin account, never to their day-to-day
  mailbox-bearing account. [FILL IN: confirm whether separate admin accounts exist, and their naming convention].
- Every role assignment gets reviewed at least annually; record the review date in the
  Decisions & History table at the bottom of this file.
- If Entra ID P2 is licensed, use Privileged Identity Management so roles are eligible rather than
  permanently active: [FILL IN: whether Entra ID P2 / PIM is licensed in this tenant].

### Licensing model and assignment

Licence inventory — read from **M365 admin centre > Billing > Your products**, do not estimate:

| Subscription / SKU | Seats purchased | Seats assigned | Assigned to whom / what for |
|--------------------|-----------------|----------------|-----------------------------|
| [FILL IN: SKU name] | [FILL IN: seats purchased] | [FILL IN: seats assigned] | [FILL IN: staff group] |
| [FILL IN: SKU name] | [FILL IN: seats purchased] | [FILL IN: seats assigned] | [FILL IN: e.g. shared/kiosk staff] |

Assign licences by group, not by clicking individual users. Group-based licensing means a new hire
lands in the right AD group, syncs up, and gets licensed without a second manual step.

    New-MgGroup -DisplayName "M365-Licence-E3" -MailEnabled:$false -SecurityEnabled -MailNickname "m365licencee3"   # creates a cloud security group to hold a licence
    Get-MgSubscribedSku | Select-Object SkuPartNumber, ConsumedUnits, @{n='Total';e={$_.PrepaidUnits.Enabled}}       # seat usage per SKU — run before buying more
    Get-MgUser -Filter "assignedLicenses/`$count eq 0 and userType eq 'Member'" -ConsistencyLevel eventual -CountVariable c   # users with no licence

Licence groups in use: [FILL IN: the actual group names used for licence assignment, and whether they are on-prem AD groups or cloud-only].

Usage-location must be set on a user before a licence will attach. For Example Org that is `CA`.

    Update-MgUser -UserId user@example.com -UsageLocation CA    # required before any licence assignment

### Security defaults vs. custom policy

The retired stub recorded the path **Home > Org settings > Baseline security mode**, which is
where the tenant-wide baseline toggle lives in the M365 admin centre. Microsoft's model is
either/or:

- **Security defaults** — a single on/off switch. Forces MFA registration for everyone, blocks
  legacy authentication, requires MFA for admins. No exclusions, no customisation. Correct choice
  for a tenant with no Entra ID P1.
- **Custom conditional access** — requires Entra ID P1 or higher. Lets you exclude break-glass
  accounts, scope by app, location, device state, and risk. Security defaults must be turned off
  before CA policies take effect.

Current state: [FILL IN: whether security defaults are enabled or conditional access is in use — check Entra > Overview > Properties > Manage security defaults].

The failure mode to avoid: turning security defaults off intending to build CA policies, then not
finishing. That leaves the tenant with neither. If you disable security defaults, have at least
the baseline CA policy set below in report-only mode the same day.

### MFA and conditional access policy set

Authentication methods should be phishing-resistant where possible. Preference order at the organization:
Microsoft Authenticator number matching or FIDO2 > Authenticator push > TOTP app > SMS (last
resort, and only where nothing else works). SMS and voice are the two methods worth actively
retiring — [FILL IN: current enabled authentication methods from Entra > Authentication methods].

Baseline conditional access policy set. Build each one in **report-only** first, review the sign-in
log impact for a week, then enable. Every policy excludes the break-glass accounts.

| # | Policy | Assignment | Grant / control |
|---|--------|------------|-----------------|
| CA01 | Require MFA for administrators | All users in directory roles | Require MFA |
| CA02 | Require MFA for all users | All users, exclude break-glass | Require MFA |
| CA03 | Block legacy authentication | All users, exclude break-glass | Block — targets clients that cannot do modern auth |
| CA04 | Require compliant or hybrid-joined device for [FILL IN: which apps] | [FILL IN: scope] | Require device compliance |
| CA05 | Block sign-in from outside Canada | All users, exclude break-glass and [FILL IN: any staff who travel] | Block — named location "Canada" |
| CA06 | Require MFA for risky sign-ins | All users (needs Entra ID P2) | Require MFA on medium/high risk |

Actual deployed policies and their names: [FILL IN: export of current CA policies — Entra > Protection > Conditional Access].

Before enabling any blocking policy, confirm the break-glass exclusion is present on it. A CA
policy with no exclusion is the standard way tenants lock themselves out.

### Exchange Online mailbox administration

Only relevant to the extent Exchange Online is actually used here — see the Overview caveat.

    Connect-ExchangeOnline -UserPrincipalName admin@example.com        # connect
    Get-Mailbox -ResultSize Unlimited | Select DisplayName,PrimarySmtpAddress,RecipientTypeDetails   # inventory of all mailboxes
    Get-MailboxStatistics user@example.com | Select DisplayName,TotalItemSize,ItemCount                    # size check before a quota complaint
    Add-MailboxPermission -Identity shared@example.com -User user@example.com -AccessRights FullAccess -InheritanceType All   # grant full access to a shared mailbox
    Add-RecipientPermission -Identity shared@example.com -Trustee user@example.com -AccessRights SendAs        # allow send-as from that mailbox
    Set-Mailbox user@example.com -LitigationHoldEnabled $true                # preserves all content — check retention policy before using
    Get-MessageTrace -SenderAddress user@example.com -StartDate (Get-Date).AddDays(-2) -EndDate (Get-Date)  # where did a message go, last 48h
    New-Mailbox -Shared -Name "Reception" -DisplayName "Reception" -Alias reception   # create a shared mailbox (no licence needed)

Shared-mailbox delegation in Zimbra works differently — see
`Mail & Messaging/zimbra-mail-administration-guide.md` and the `Share Mailboxes Zimbra/` folder.
Do not carry Exchange habits across; the Zimbra model is grants on the mailbox, not permissions
objects.

---

## Operations (Day-2)

### User lifecycle in a hybrid AD context

The rule: **create and disable in on-prem AD, never in the cloud.** A cloud-only change to a synced
object either gets overwritten on the next sync cycle or leaves the two directories disagreeing.

**Onboarding**
1. Create the user in AD on `auth2.example.com` in the correct OU, with `UserPrincipalName` set to
   `firstname.lastname@example.com` ([FILL IN: confirm the actual UPN naming convention in use]).
2. Add to the AD groups that drive licence assignment and access.
3. Force a sync rather than waiting the default 30 minutes, from the sync server:

       Start-ADSyncSyncCycle -PolicyType Delta      # pushes changes to Entra ID now (Entra Connect sync only)

4. Confirm the object arrived and set usage location:

       Get-MgUser -UserId firstname.lastname@example.com | Select DisplayName,UserPrincipalName,OnPremisesSyncEnabled   # OnPremisesSyncEnabled should be True
       Update-MgUser -UserId firstname.lastname@example.com -UsageLocation CA    # required before licensing

5. Confirm the licence arrived by group membership; assign directly only as an exception.
6. Walk the user through MFA registration at https://aka.ms/mfasetup.

Cross-reference `SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt` for the
non-M365 parts of onboarding.

**Offboarding**
1. Disable the AD account and reset its password — this is the step that actually stops access.
2. Revoke existing cloud sessions, because a disabled AD account can still have valid tokens for
   up to an hour:

       Revoke-MgUserSignInSession -UserId user@example.com    # invalidates refresh tokens immediately

3. Convert the mailbox to shared if mail needs to stay reachable ([FILL IN: whether the departing user's mail is in Zimbra or Exchange Online — the procedure differs]).
4. Transfer OneDrive ownership to the manager before the retention window expires
   ([FILL IN: OneDrive retention period configured for deleted users]).
5. Remove the licence only after mail and files are handled — removing it first starts deletion
   timers.
6. Delete the AD object after [FILL IN: the retention period agreed for disabled accounts].

**Name changes**: change in AD, let it sync, then confirm the cloud UPN and primary SMTP updated.
Aliases for the old address should be kept: [FILL IN: how long old addresses are retained as aliases].

### Audit log access

Two separate logs, and people mix them up:

- **Entra sign-in logs** — who signed in, from where, with what MFA result, and which CA policy
  applied. Entra admin centre > Monitoring > Sign-in logs. Retention on the free tier is short
  ([FILL IN: retention — 7 days on free, 30 days with P1/P2 — confirm which applies]).
- **Unified audit log** — what was *done* (mailbox access, file downloads, admin config changes).
  Purview compliance portal > Audit. Must be enabled before it records anything:
  [FILL IN: confirm unified audit log is turned on — Purview > Audit > "Start recording user and admin activity"].

      Search-UnifiedAuditLog -StartDate (Get-Date).AddDays(-7) -EndDate (Get-Date) -RecordType AzureActiveDirectory   # admin/directory changes, last week
      Search-UnifiedAuditLog -StartDate (Get-Date).AddDays(-7) -EndDate (Get-Date) -Operations "Add member to role."  # who was granted an admin role
      Get-MgAuditLogSignIn -Top 50 -Filter "status/errorCode ne 0"   # recent failed sign-ins

For an investigation, pull both and line them up by timestamp. If there is an on-prem component,
also pull the DC security log — see `Active Directory/AD-Admin-Security-Guide.md` section on
detection workflows.

Long-term retention beyond the portal defaults requires exporting. [FILL IN: whether M365 audit data is forwarded anywhere — e.g. graylog01.example.com — and how].

### Routine checks

| Check | Where | Frequency |
|-------|-------|-----------|
| Directory sync healthy, last sync recent | Entra > Health > Connect Sync, or `Get-ADSyncScheduler` on the sync host | Weekly |
| Licence seats vs. assigned | Billing > Your products | Monthly |
| Admin role assignments unchanged | Entra > Roles and administrators | Monthly |
| Break-glass accounts tested | See procedure below | Quarterly |
| CA policies still as documented | Entra > Conditional Access | Quarterly |
| Message centre / service health advisories | admin.microsoft.com > Health | Weekly |

---

## Troubleshooting

### Symptom: new user exists in AD but not in M365
- Likely cause: sync has not run, the OU is out of sync scope, or the object failed validation.
- Check:  `Get-ADSyncScheduler` on the sync host    # confirms sync is enabled and when it last ran
- Check:  Entra portal > Health > Connect Sync > errors   # shows per-object sync failures
- Fix:    `Start-ADSyncSyncCycle -PolicyType Delta`    # forces an immediate delta sync
- If the object still does not appear, the OU is probably excluded from the sync scope — check the
  Entra Connect configuration wizard's domain/OU filtering page.

### Symptom: user cannot sign in, "your account is blocked"
- Likely cause: a conditional access policy is blocking, or the AD account is disabled.
- Check:  Entra > Sign-in logs > find the user's failed attempt > Conditional Access tab   # names the exact policy that blocked
- Fix:    correct the underlying condition (location, device compliance) rather than excluding
  the user from the policy. Exclusions accumulate and hollow out the policy.

### Symptom: licence will not assign, "usage location not set"
- Fix:    `Update-MgUser -UserId user@example.com -UsageLocation CA`   # sets the required attribute

### Symptom: MFA prompt loop, user can never complete sign-in
- Likely cause: broken or stale registered method, or time skew on the authenticator device.
- Check:  Entra > user > Authentication methods   # lists registered methods
- Fix:    require re-registration, then have the user re-enrol at https://aka.ms/mfasetup

      Update-MgUserAuthenticationMethod ...    # [FILL IN: confirm the exact cmdlet/graph call used for the require-re-register action in the current module version]

### Symptom: changes made in the cloud keep reverting
- Likely cause: the attribute is sourced from on-prem AD and sync overwrites it.
- Fix:    make the change in AD and let it sync. This is expected behaviour, not a fault.

### Symptom: legacy application cannot authenticate after MFA enforcement
- Likely cause: the app uses basic auth, which is blocked by CA03 / security defaults.
- Check:  Entra > Sign-in logs, filter Client app = "Other clients"   # legacy auth attempts show here
- Fix:    move the app to modern auth, or if it is a service account, [FILL IN: the approved
  exception mechanism — e.g. an app registration with a certificate, or a scoped CA exclusion].
  Do not disable the legacy-auth block tenant-wide.

---

## Security

- **Exposure**: internet-facing by nature. There is no network perimeter in front of Entra ID;
  conditional access *is* the perimeter.
- **Auth**: `UserPrincipalName` from on-prem AD, password verified per the configured sign-in
  method, plus MFA. Base DN and bind details for the on-prem side are in `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11).
- **Certificates**: no tenant-side cert to manage unless AD FS federation is in use. If it is,
  the token-signing certificate rotation is a hard dependency:
  [FILL IN: whether AD FS is in use and, if so, the token-signing cert expiry].
- **Secrets**: break-glass credentials are stored offline (see below). App registration secrets and
  certificates are inventoried at [FILL IN: where app registration credentials are recorded, and their expiry dates]. Never in this doc.
- **Hardening notes**: block legacy auth; disable user consent to third-party apps unless reviewed;
  restrict who can register applications; disable self-service tenant joins.
  [FILL IN: current values for Entra > User settings — app registration, user consent, guest access].

### Break-glass account procedure

Two emergency-access accounts exist so that a misconfigured conditional access policy, an expired
federation certificate, or a lost admin phone cannot lock IT out of the tenant.

Requirements for each:
- Cloud-only account on the `.onmicrosoft.com` domain, **not** synced from AD, so an on-prem
  outage does not affect it.
- Permanently assigned Global Administrator (not PIM-eligible — PIM activation can itself require
  MFA that may be the thing that is broken).
- Excluded from every conditional access policy, explicitly and individually.
- Long random password, [FILL IN: password length/complexity standard used], plus a
  phishing-resistant second factor that does not depend on a personal phone:
  [FILL IN: second factor in use for break-glass — e.g. FIDO2 key held where].
- Password and recovery details sealed and stored offline in two physical locations:
  [FILL IN: the two storage locations for break-glass credentials].
- Sign-in alerting on both accounts so any use is noticed immediately:
  [FILL IN: where break-glass sign-in alerts are sent].

Accounts: [FILL IN: the two break-glass UPNs].

**Quarterly test** (do not skip — an untested break-glass account is a rumour):
1. Retrieve one envelope, note who opened it and when.
2. Sign in from a clean browser profile. Confirm no CA policy blocks the sign-in.
3. Confirm the account still holds Global Administrator.
4. Rotate the password, reseal, restore to storage, log the test in Decisions & History.
5. Confirm the sign-in alert fired.

**When to use**: only when normal admin access is impossible. Every use is an incident and gets
written up, with the password rotated afterwards regardless of outcome.

## Monitoring & Alerting

- Entra sign-in logs reviewed for impossible-travel and repeated-failure patterns.
  [FILL IN: whether Identity Protection is licensed — risk detections need P2].
- Alerts configured: [FILL IN: which alert policies exist in Purview / Defender and where they email].
- Service health advisories arrive via the M365 message centre;
  [FILL IN: whether message centre digests are emailed, and to whom].
- Baseline "normal": [FILL IN: typical daily sign-in volume and failure rate, once observed].
- Consider forwarding sign-in and audit data to `graylog01.example.com` for retention beyond the
  portal default — [FILL IN: whether this integration exists].

## Disaster Recovery

- **RTO/RPO**: Microsoft runs the service; the organization's recovery scope is configuration and identity, not
  infrastructure. [FILL IN: agreed RTO/RPO for tenant configuration recovery].
- **Deleted users**: soft-deleted objects are recoverable for 30 days.

      Get-MgDirectoryDeletedItemAsUser        # lists recoverable deleted users
      Restore-MgDirectoryDeletedItem -DirectoryObjectId <id>   # restores one within the 30-day window

- **Deleted groups/sites**: SharePoint and Teams have their own recycle bin stages before permanent
  deletion. [FILL IN: SharePoint retention/deletion settings configured for this tenant].
- **Configuration backup**: export conditional access policies and role assignments to a file kept
  outside the tenant, so a bad change can be reversed:

      Get-MgIdentityConditionalAccessPolicy | ConvertTo-Json -Depth 10 > ca-policies-backup.json   # run before any CA change

  [FILL IN: where these exports are stored and how often they are taken].
- **Tenant recovery scenarios**:
  - *Locked out of all admin accounts* — use break-glass. If break-glass also fails, the only
    path is a Microsoft support ticket with tenant ownership proof, which is slow (days, not
    hours). Contact path: [FILL IN: Microsoft support arrangement — partner/CSP or direct, and account number].
  - *Directory sync server lost* — cloud objects persist and users keep working; new provisioning
    stops. Rebuild the sync agent on a new host and re-run the wizard against the same tenant.
    Do not enable a second sync server against the same tenant simultaneously.
  - *Accidental mass deletion via sync* — Entra Connect halts sync if deletions exceed its
    threshold. Do not simply raise the threshold to push it through; find out why the objects
    disappeared from AD first.
  - *Tenant compromise* — rotate all admin credentials, revoke all refresh tokens
    (`Revoke-MgUserSignInSession` per user), audit app registrations and consent grants for
    attacker-created ones, then review the unified audit log for the blast radius. See
    `Security Procedures/IR_Security_Scenarios_Guide.md`.
- **Escalation**: primary IT lead; if unavailable, [FILL IN: secondary contact and after-hours escalation path].

## Decisions & History (ADR-lite)

| Date       | Decision / Change                                           | Why / Ticket |
|------------|-------------------------------------------------------------|--------------|
| 2026-01-21 | Note recorded re: Org settings > Baseline security mode      | Original `_ARCHIVE/superseded-stubs/O365 security policy updates.txt` (archived 2026-09-11) stub |
| 2026-09-11 | Stub superseded by this guide                                | Documentation consolidation |
| [FILL IN: date]  | [FILL IN: when security defaults were enabled/disabled]      | [FILL IN: why / ticket]    |
| [FILL IN: date]  | [FILL IN: when hybrid sync was established]                  | [FILL IN: why / ticket]    |

## References

- `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11) — LDAP bind details for the on-prem directory
- `Active Directory/AD-Admin-Security-Guide.md` — section 14 covers hybrid identity and Entra Connect
- `Mail & Messaging/zimbra-mail-administration-guide.md` — primary mail platform
- `Mail & Messaging/email-authentication-spf-dkim-dmarc.md` — SPF/DKIM/DMARC, including any M365 sending
- `SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt`
- `_ARCHIVE/superseded-stubs/O365 security policy updates.txt` (archived 2026-09-11) — retired stub, superseded by this file
- Microsoft: Entra ID documentation, conditional access templates, emergency-access account guidance

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

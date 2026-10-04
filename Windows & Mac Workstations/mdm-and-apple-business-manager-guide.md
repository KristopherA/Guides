> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# MDM and Apple Business Manager

## Current State — Read This First

**Nothing in this documentation library establishes that the organization operates an MDM
or holds an Apple Business Manager account.** That is a statement about the
evidence, not a claim that one does not exist. The evidence is:

| Source | What it says | What it proves |
|--------|--------------|----------------|
| `SysAdmin Procedures/Inventory_Asset_Reference.txt` §4 | Has an "MDM Enrolled" column in the workstation table | Nothing — the table has no rows filled in |
| `SysAdmin Procedures/Inventory_Asset_Reference.txt` §8 | Lists "MDM (Jamf/Mosyle) — https://example.jamfcloud.com" | Nothing reliable — that whole file is unfilled boilerplate (the same section carries `example.com`, `10.0.0.10`, and `Tenant ID: xxxxxxxxx`). Treat the Jamf URL as a lead to check, not a fact |
| `SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt` | "Enroll in MDM (if using Jamf/Mosyle/Kandji)" | The checklist author was not sure either |
| `Support Procedures/macos-desktop-support-runbook.txt` §18 | A full Jamf/Mosyle troubleshooting section | Generic vendor procedure, not evidence of a tenant |
| `Security & Hardening/endpoint-security-operations-guide.md` | Repeatedly `[FILL IN: MDM in use and console URL]` | The same open question, recorded elsewhere |

**[FILL IN: does the organization have an MDM? Name the product, the tenant/console URL, the
licence count, the renewal date, and who holds the administrator account. If the
answer is no, write "none" here explicitly so the next person does not have to
repeat this search.]**

**[FILL IN: does the organization have an Apple Business Manager (or the older Apple School
Manager / DEP) account? If yes, record the organisation name as it appears in
ABM, the Managed Apple ID that holds the Administrator role, and the customer
numbers registered under Settings → Enrollment Information. If no, say so.]**

Answer both of those before using the rest of this document, because the
document forks on them:

- **If an MDM exists** — sections marked **[Operating]** apply. Verify the
  configuration described actually matches, and correct this document where it
  does not.
- **If no MDM exists** — sections marked **[Evaluation]** apply. They set out
  what would be gained, what it costs, and the order to do it in. Everything
  else here is still worth reading, because it describes the gaps the organization currently
  has.

The two facts that are established, and which everything below builds on:

- The Mac fleet is managed by **Munki** — see
  `Windows & Mac Workstations/munki-server-administration-guide.md`.
- Workstations are **bound to Active Directory** (`auth2.example.com`,
  `auth4.example.com`) — see `Active Directory/AD-Admin-Security-Guide.md`.

Munki and AD binding between them cover software delivery and identity. They do
not cover configuration enforcement, device identity, or remote wipe. That gap
is what this document is about.

## Overview

This guide covers device management beyond software delivery: enrolment,
supervision, configuration enforcement, device identity, and the ability to
lock or erase a machine the organization no longer physically controls.

Munki already answers "what software is on this Mac." It does not answer "is
FileVault on and where is the recovery key," "does this Mac lock its screen,"
"can this Mac be erased remotely if it is stolen," or "is this Mac one of ours
at all." Those questions are MDM's job. On the Windows side the equivalent
questions are answered partly by Group Policy today and would be answered by
Entra ID join and Intune if the organization went that way — see
`Microsoft 365/m365-entra-admin-guide.md`.

Audience is the sysadmin and anyone covering for them. Tier 3 depth: this is a
system with security and compliance implications, and the lifecycle sections
are meant to be followed step by step.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Owner            | IT lead (it@example.com), IT Ops                            |
| Environment      | prod — the whole managed workstation fleet, macOS and Windows 11       |
| Location         | MDM console `[FILL IN: product and console URL, or "none — no MDM in use"]`; Apple Business Manager https://business.apple.com `[FILL IN: is there an account? organisation name?]`; Entra ID https://entra.microsoft.com |
| Access           | MDM console `[FILL IN: how it is reached and who has access]`; ABM requires a Managed Apple ID with the Administrator or Device Enrollment Manager role `[FILL IN: which Managed Apple IDs exist and what roles they hold]`; Entra/Intune via the M365 tenant |
| Dependencies     | Internet egress through pfSense to Apple's MDM push endpoints and to the MDM vendor's cloud; DNS; an APNs certificate (expires annually — see Security); Active Directory for identity; the reseller or Apple sales channel for ABM device assignment |
| Dependents       | FileVault recovery key escrow; configuration profile enforcement (screen lock, firewall, update deferral, restrictions); remote lock and wipe; zero-touch provisioning; Wi-Fi/802.1x certificate delivery if that is done by profile; the onboarding and offboarding checklists |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                               |

Facts that must be read off the live systems rather than assumed:

| Item | Value |
|------|-------|
| MDM product and version | `[FILL IN]` |
| MDM console URL | `[FILL IN]` |
| Licence count / seats and renewal date | `[FILL IN]` |
| APNs certificate Apple ID and expiry | `[FILL IN: the Apple ID used to generate the APNs cert — losing it means re-enrolling every device]` |
| ABM organisation name and organisation ID | `[FILL IN: read from business.apple.com → Settings → Enrollment Information. Do not guess this]` |
| ABM customer numbers (Apple direct / reseller) | `[FILL IN]` |
| Reseller of record for Apple hardware | `[FILL IN: who the organization buys Macs from, and their Apple reseller ID]` |
| Number of Macs in the fleet | `[FILL IN]` |
| Number of Macs currently enrolled / supervised | `[FILL IN]` |
| Windows devices: AD-joined / Entra-joined / hybrid | `[FILL IN: current join state of the Windows 11 fleet]` |
| Intune licensed and in use? | `[FILL IN: does the M365 licensing include Intune, and is any device enrolled?]` |

## How It Works

### Munki and MDM are not substitutes for each other

This is the distinction the rest of the document rests on, so it is worth
being precise about.

| Capability | Munki | MDM |
|------------|-------|-----|
| Install, update and remove applications | **Yes — this is what it is for** | Possible, but clumsy for anything but App Store and bootstrap packages |
| Self-service software catalogue for users | Yes (Managed Software Center) | Varies by product, usually worse |
| Enforce a setting the user cannot change | No | **Yes — configuration profiles** |
| Escrow a FileVault recovery key | No | **Yes** |
| Prove a device is the organization's before macOS is installed | No | **Yes, via ABM/ADE** |
| Lock or erase a device remotely | No | **Yes** |
| Survive an erase and reinstall | No — the client must be rebuilt | **Yes, if the device is in ABM** |
| Work when the device has never been touched by IT | No | **Yes, with ADE — the device enrols itself at Setup Assistant** |
| Report compliance state to a console | Only with an add-on (MunkiReport/Sal) | **Yes, natively** |

Munki is a **pull** system over plain HTTP: the client decides to check in,
downloads what a manifest tells it to, and installs it. It has no channel back,
no way to make the machine do something now, and no way to stop a user undoing
its work. An administrator who removes munkitools from a Mac has removed Munki
from that Mac.

MDM is a **push-capable** system built into macOS itself. The MDM server sends
a push notification via Apple's servers; the device wakes and asks the server
what it wants. The management payload lives in the OS, not in an installed
application, so a user cannot uninstall it — and on a supervised device, cannot
remove it at all.

**The practical summary:** Munki answers "what is installed." MDM answers "what
is enforced, whose device is this, and can I erase it." Running Munki without
an MDM is a defensible position for software delivery and a poor one for
encryption key escrow and for lost hardware. Running MDM without Munki means
doing software packaging in a tool that is worse at it. Most Mac-managing
organisations that use Munki run both, deliberately.

### Apple Business Manager and Automated Device Enrolment

Apple Business Manager (ABM) is Apple's free portal at
https://business.apple.com. Two things matter here.

**1. Automated Device Enrolment (ADE).** A Mac whose serial number sits in ABM
and is assigned to an MDM server contacts Apple during Setup Assistant, is told
"you belong to this organisation, enrol with this MDM," and does so before a
user account exists. This is what "zero-touch" means: the machine is unboxed at
the user's desk, joins Wi-Fi, and configures itself.

**2. Supervision.** A Mac enrolled through ADE is *supervised*. Supervision is
not a setting you turn on later; it is a property of how the device enrolled.
What it buys:

| Property | Manual enrolment | ADE enrolment (supervised) |
|----------|------------------|----------------------------|
| User can remove the MDM profile | Yes — System Settings → Privacy & Security → Profiles → remove | **No** |
| Enrolment survives erase-and-reinstall | No | **Yes — Setup Assistant re-enrols it** |
| Enrolment survives a DFU restore via Apple Configurator | No | **Yes** |
| Activation Lock can be managed/bypassed by the organisation | No | **Yes**, if Activation Lock is enabled via MDM |
| Restrictions that require supervision (e.g. certain payloads) | Unavailable | Available |
| Device is identifiably the organization's to Apple and to a repair depot | No | **Yes** |

**This is the capability manual enrolment can never provide.** A manually
enrolled Mac that is stolen, wiped, and set up by the thief is simply gone: it
has no relationship to the organization any more. A supervised Mac that is stolen and wiped
re-enrols itself into the organization's MDM on first network contact.

**The purchase linkage is the part people miss.** A device only appears in ABM
if Apple knows the organization bought it. That requires one of:

- **Purchased directly from Apple** using the organization's Apple Customer Number,
  registered in ABM under Settings → Enrollment Information. Devices appear
  automatically.
- **Purchased from an Apple Authorized Reseller** who is enrolled in the
  programme, using the organization's ABM Organisation ID and the reseller's Reseller ID,
  both registered in ABM. The reseller submits the serials. **This has to be
  arranged with the reseller before the purchase**, and it has to be repeated
  on every purchase order — it is not a switch that stays on.
- **Added manually with Apple Configurator** (Mac or iPhone app). This works
  for existing devices and for devices bought outside the channel, but it
  requires physical access and puts the device into a **30-day provisional
  period** during which a user can release it from ABM. After 30 days it is
  permanent. It also requires erasing the Mac.

Consequence: **a Mac bought from a retail store, a broker, or on a personal
credit card cannot be made supervised without a wipe**, and may never carry the
same assurance. Purchasing channel is a device-management decision, not just a
procurement one. `[FILL IN: where does the organization actually buy Mac hardware, and is
that channel registered in ABM?]`

ABM also provides **Managed Apple IDs** (organisation-owned Apple accounts,
federatable to Entra ID) and **Apps and Books** (volume app licences owned by
the organisation rather than by an individual's Apple ID). Both are worth
having and neither is the reason to set ABM up; ADE is.

### Device identity: three overlapping directories

An organization-owned Mac can hold up to three separate identities, and confusing them causes
most enrolment tickets:

```
Hardware serial  ->  Apple Business Manager   # "Apple says this device is the organization's"
Device record    ->  MDM                      # "the organization's MDM manages this device"
Computer object  ->  Active Directory         # "this device has a machine account in example.com"
```

They are independent. A Mac can be AD-bound and not in MDM (probably the
current state), in MDM and not AD-bound, or in ABM but assigned to no MDM
server at all (a common and silent failure — the device shows in ABM and still
enrols nowhere). Retirement means removing all three, and the offboarding
checklist currently only removes the AD computer object.

### The Windows side

Windows 11 at the organization is AD-bound today. The cloud equivalents are:

| Model | What it means | Applies here? |
|-------|---------------|-----------------|
| **AD domain join** (current) | Machine account in `example.com`, policy by GPO, no cloud device identity | Yes — current state |
| **Hybrid Entra join** | AD-joined *and* registered in Entra ID; GPO still applies, device becomes visible to Conditional Access | `[FILL IN: is Entra Connect configured to sync device objects? See the M365 guide]` |
| **Entra join** (cloud-native) | No AD machine account; policy by Intune | `[FILL IN: any devices in this state?]` |
| **Intune enrolment** | MDM for Windows: configuration profiles, compliance policy, BitLocker key escrow, remote wipe | `[FILL IN: licensed? in use?]` |

Hybrid join is the low-risk step, because it changes nothing about how the
machine is managed and makes devices visible for Conditional Access — which is
what lets "only from a managed device" become an enforceable condition rather
than an aspiration. See `Microsoft 365/m365-entra-admin-guide.md` for the
tenant-side configuration and for what Entra Connect is currently syncing.

**BitLocker key escrow is the Windows analogue of FileVault escrow and the same
argument applies**: a key that exists only on the machine it decrypts is not a
recovery key. AD-joined machines can escrow to AD; Entra-joined machines escrow
to Entra ID. `[FILL IN: are BitLocker recovery keys currently escrowed to AD,
and has a recovery actually been tested?]`

**Parallels VMs.** Windows 11 running under Parallels on Apple silicon (see
`Windows & Mac Workstations/windows-11-desktop-runbook.md` §12) is a real
Windows install and can be domain-joined, Entra-joined and Intune-enrolled like
any other. Points specific to these VMs:

- The VM and its host Mac are **two devices**, and both need managing. Managing
  the Mac does nothing for the VM.
- Networking mode matters: Parallels defaults to Shared (NAT). Domain join,
  device-based network policy, and anything that needs the VM to be its own LAN
  device require **Bridged** networking.
- The VM's device identifiers are synthetic and change if the VM is recreated
  or cloned. **Never clone a managed VM** — the clone inherits the machine
  account and the two fight over it. Build from a fresh install, or sysprep.
- Windows Hello and biometrics do not pass through, so any Intune compliance or
  Conditional Access policy requiring a biometric or TPM-backed Windows Hello
  will fail on these VMs by design. Plan the policy exceptions before enforcing.
- Parallels snapshots are not backup and will happily restore a machine to a
  pre-policy state.
- `[FILL IN: how many Parallels Windows VMs exist, who has them, and are they
  domain-joined?]`

## Operations (Day-2)

### Device lifecycle

The lifecycle below ties into
`SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt`. Where a step is
missing from that checklist today, it is called out — those are the gaps to
close when the checklist is next revised.

**1 — Procurement**

- Buy through a channel registered in ABM (see above). If the purchase does not
  go through that channel, the device will not be supervised and cannot be made
  so without a wipe.
- Record on the purchase order: the ABM Organisation ID, or the Apple Customer
  Number, so the reseller can submit the serials. `[FILL IN: the exact
  identifiers to quote — read them from ABM, do not guess]`
- Capture the serial number at receipt and add it to the asset inventory.
- Proposed control: **no Mac enters service without appearing in ABM first**
  `[CONFIRM: proposed default]`.

**2 — ABM assignment** *[Operating]*

- Confirm the serial has appeared: ABM → Devices, search the serial.
- Assign it to the MDM server: ABM → Devices → select → Edit Device Management
  → assign to `[FILL IN: MDM server name as it appears in ABM]`.
- **Assignment is the step that is silently skipped.** A device sitting in ABM
  unassigned looks fine in the portal and enrols nowhere.
- Set the enrolment profile options on the MDM side: supervision on, enrolment
  non-removable, which Setup Assistant panes to skip.
- Proposed defaults for the ADE profile `[CONFIRM: proposed default]`:
  supervised, MDM removal disallowed, await device configuration on, skip Apple
  ID / Siri / Screen Time / Analytics / Touch ID marketing panes, do not skip
  FileVault (so encryption is prompted at first login) or Terms.

**3 — Enrolment and provisioning**

For an ADE device the enrolment happens by itself at Setup Assistant. The
remaining work is to confirm it happened and to layer on the things MDM does
not do:

```
profiles status -type enrollment                  # expect: Enrolled via DEP: Yes / MDM enrollment: Yes
profiles list -all                                # every profile now installed
sudo profiles renew -type enrollment              # force a re-check against ABM if it did not enrol
```

Then the existing onboarding steps: set the hostname, bind to AD, set the Munki
`ClientIdentifier`, enable FileVault, and run the compliance check.

```
sudo scutil --set ComputerName "[FILL IN: naming convention]"        # display name
sudo scutil --set HostName "[FILL IN: fqdn]"                          # fqdn
sudo scutil --set LocalHostName "[FILL IN: bonjour name]"             # bonjour name
sudo dsconfigad -show                                                  # confirm AD bind state
sudo defaults write /Library/Preferences/ManagedInstalls ClientIdentifier "[FILL IN: manifest name]"
sudo managedsoftwareupdate -vvv                                        # first Munki run
sudo fdesetup status                                                    # expect FileVault is On
sudo bash "Support Procedures/macos-nist-check.sh"                      # baseline compliance snapshot
```

Order matters in one place: **bind to AD before enabling FileVault**, so the
user's AD account gets a SecureToken and can unlock the disk. A Mac where only
the local admin can unlock FileVault is a support incident waiting for a
password reset.

**4 — Deployment to the user**

- Confirm the user can log in with AD credentials, that FileVault is on, and
  that the recovery key is visible in the MDM console **before** the machine
  leaves IT's hands. An unescrowed key discovered later is unrecoverable.
- Record in the asset inventory: serial, asset tag, hostname, assigned user,
  AD joined, MDM enrolled, FileVault state.

**5 — Reassignment between users**

Reassignment is not a soft handover. Proposed standard: **wipe between users**
`[CONFIRM: proposed default]`. The reasons are not paranoia — the outgoing
user's keychain, cached credentials, FileVault unlock token, browser sessions
and local files all persist otherwise, and an organization that holds member
data on staff machines.

```
sudo fdesetup list                                 # who currently holds FileVault unlock tokens
sudo fdesetup remove -user [FILL IN: outgoing user] # revoke the outgoing user's unlock token
```

Then Erase All Content and Settings (System Settings → General → Transfer or
Reset), which on a supervised device re-enrols at Setup Assistant. Update the
asset inventory and the MDM device record's assigned user.

**6 — Wipe and retirement**

All three identities must be removed, in this order:

```
# 1. From the MDM console: issue Erase Device, and WAIT for it to report complete
# 2. Confirm the erase happened before doing anything else
sudo profiles status -type enrollment              # on a device you still hold
# 3. Remove the AD computer object
Remove-ADComputer -Identity "[FILL IN: computer name]" -Confirm:$false
# 4. Release from ABM only if the device is leaving the organization permanently
#    ABM -> Devices -> select -> Release. THIS IS IRREVERSIBLE.
```

**Do not release a device from ABM for a wipe-and-reissue.** Release is for
sale, donation, or e-waste only. A released device cannot be re-added except
via Apple Configurator, and only if it was not bought in the channel. Releasing
a Mac that is merely being reassigned throws away its supervision permanently.

The current offboarding checklist (Step 3) does `sudo profiles remove -type
enrollment -forced` then wipes. On a supervised device that command fails by
design, and it should — the intended flow is to wipe *through* MDM, not to
unenrol first. `[FILL IN: update the offboarding checklist to match whichever
model the organization ends up with.]`

Retirement for disposal additionally requires attesting that the drive is
unrecoverable. On Apple silicon and T2 Macs, Erase All Content and Settings
destroys the hardware encryption key, which is sufficient. On older Intel Macs
without a T2, it is not — those need a full-disk erase or physical destruction.

### Configuration profile management *[Operating]*

Profiles are the enforcement mechanism. A profile is a signed plist of
payloads; delivered by MDM it is installed at the device level and, on a
supervised device, cannot be removed by the user.

Rules that keep a profile estate manageable:

- **One payload concern per profile.** A single monolithic profile means any
  change re-pushes everything and one bad payload blocks the rest.
- **Scope by group, not by device**, mirroring the Munki manifest strategy.
- **Test on one machine first.** A profile that locks out login or breaks
  network reaches the whole fleet in minutes and is hard to undo remotely.
- Keep profile sources in version control or exported from the console. A
  profile that exists only in a web UI is not backed up.

Proposed baselines. Every threshold below is a proposal, not a discovered fact,
and each should be walked through against how staff actually work before being
enforced.

| Payload | Proposed setting | Why | Status |
|---------|------------------|-----|--------|
| FileVault | Require enablement at login; escrow the personal recovery key to MDM; defer no more than 2 logins | An unencrypted laptop is a data breach on loss. Escrow is the whole point — see below | `[CONFIRM: proposed default]` |
| Screen lock | Screensaver at 10 minutes; password required immediately; 5-second grace | NIST AC-11, and the check script tests exactly this | `[CONFIRM: proposed default]` |
| Login window | Show name and password fields, not the user list; disable guest; disable automatic login | Does not advertise valid usernames; check script tests it | `[CONFIRM: proposed default]` |
| Firewall | Application firewall on, stealth mode on, block-all off | NIST SC-7; block-all off because it breaks printing and file sharing discovery | `[CONFIRM: proposed default]` |
| Software update | Automatic check and download on; install security responses and system files automatically; defer major macOS upgrades 60 days, minor updates 7 days | Deferral stops a surprise major release breaking a line-of-business app; 60/7 is the common enterprise pairing | `[CONFIRM: proposed default]` |
| Restrictions | Block removal of the MDM profile; block Erase All Content and Settings by the user; disallow iCloud Drive for Desktop and Documents sync | The last one matters for member data — iCloud sync is an uncontrolled copy off the file server | `[CONFIRM: proposed default]` |
| Gatekeeper / privacy | Gatekeeper enforced (App Store and identified developers); block user override | Check script tests Gatekeeper state | `[CONFIRM: proposed default]` |
| Password policy | Inherit from AD rather than setting a second, conflicting policy in MDM | Two policies on one machine produce lockouts nobody can explain | `[CONFIRM: proposed default]` |
| PPPC / TCC | Pre-approve Full Disk Access and Screen Recording for the AV/EDR agent and the remote support tool | Otherwise every machine prompts the user, and some will decline | `[CONFIRM: proposed default]` |
| Certificates | Deliver the internal CA and any 802.1x client certificate by profile | Removes the manual keychain step in onboarding | `[CONFIRM: proposed default]` |

Verify what is actually applied on a machine:

```
profiles list -all                                 # every installed profile
profiles show -type enrollment                     # the enrolment profile specifically
sudo profiles -P                                   # verbose payload dump, device and user level
defaults read /Library/Preferences/com.apple.SoftwareUpdate   # confirm update payload landed
```

### FileVault recovery key escrow

This deserves its own section because it is the single highest-value thing MDM
does at the organization, and the current state is unknown.

A FileVault personal recovery key that exists only on the encrypted machine is
not a recovery key; it is a decoration. The situations it is needed for —
forgotten password with no other SecureToken holder, a departed employee's
machine, a failed OS upgrade leaving the disk locked, a machine recovered from
a theft — are all situations where the machine cannot be logged into. Escrow
means the key is held somewhere else: the MDM console, keyed to the serial
number.

```
sudo fdesetup status                               # is FileVault on at all
sudo fdesetup list                                 # which users hold unlock tokens
sudo fdesetup validaterecovery                     # prompts for a key, answers true/false — tests the escrowed copy
sudo fdesetup changerecovery -personal             # issue a new personal key; MDM captures it on next check-in
```

`sudo fdesetup validaterecovery` is the verification step. It confirms the key
held in escrow actually unlocks that machine, without a reboot. **A recovery
key that has never been validated is a rumour.** Proposed control: validate the
escrowed key on a random sample of 5 Macs per quarter `[CONFIRM: proposed
default]`.

If there is no MDM, the key has to live somewhere deliberate — the password
manager, one entry per machine keyed by serial — and that placement must be a
recorded procedure rather than a habit. See
`Security & Hardening/secrets-management-guide.md`. This is a worse answer than
escrow (it is manual, so it will drift, and it depends on the person enabling
FileVault remembering to do it) but it is far better than the key existing
nowhere. `[FILL IN: where do FileVault recovery keys currently live for the
organization's Mac fleet?]`

The parallel finding on Windows: BitLocker keys should escrow to AD. The
endpoint security guide already flags "FileVault or BitLocker key is not
escrowed when a machine needs recovery" as a known failure mode.

### Lost or stolen device procedure

Act in this order. The first two steps do not depend on having an MDM.

**1 — Contain the identity, immediately.** The device is a credential store
before it is hardware.

```
Disable-ADAccount -Identity "[FILL IN: username]"          # AD account, per the offboarding checklist Step 1
Set-ADAccountPassword -Identity "[FILL IN: username]" -Reset   # invalidates cached credentials
Remove-ADGroupMember -Identity "VPN-Users" -Members "[FILL IN: username]" -Confirm:$false
```

Revoke M365 sessions in Entra ID, revoke any 802.1x or Wi-Fi certificate issued
to the device, and remove its MAC from any reservation or allow-list. See
`Active Directory/AD-Admin-Security-Guide.md` and
`Microsoft 365/m365-entra-admin-guide.md`.

**2 — Record the facts while they are fresh.** Serial, asset tag, assigned
user, when and where it was last seen, whether FileVault was on, whether the
device was powered on or asleep when lost, and — the question that decides
step 4 — **what was on it.**

**3 — Remote lock.** *[Operating]* From the MDM console, issue Device Lock with
a 6-digit PIN and a message with an IT contact number. Record the PIN in the
password manager immediately; it is needed to unlock the device if it comes
back and it is shown once.

Lock is reversible and preserves data. **Always lock before considering wipe.**

**4 — The wipe decision.** This is a judgement call and should be made by a
person, not a reflex.

Wipe is **irreversible** and destroys any data that exists only on that device.

| Consider | Wipe now | Lock and wait |
|----------|----------|---------------|
| FileVault was on and the device was powered off | Less urgent — the disk is already cryptographically inaccessible | Lock; the encryption is doing the work |
| FileVault was off, or the device was awake and unlocked | **Wipe** | — |
| The device holds member data with no other copy | Confirm the backup first | Lock while confirming |
| Device is merely misplaced in the building | — | Lock; do not wipe |
| Theft with any indication of targeting | **Wipe** | — |
| The user says "my only copy of X is on it" | Verify that claim against the file server and backup before wiping | Lock, verify, then decide |

The trap is the last row. A user under stress will say data is unique to the
device; it usually is not, because it is usually on `files.example.com` or in
OneDrive. But sometimes it is, and a wipe issued in the first ten minutes has
destroyed it. **Check the backup before wiping. Locking costs nothing and buys
the time to check.** See
`Hardware & Backup/synology-nas-administration-guide.md` for snapshot recovery
and `SysAdmin Procedures/Backup_DR_Runbook.txt`.

A wipe command sits queued until the device next reaches the network. It is not
a guarantee — a device that never comes online never receives it. That is why
step 1 (identity containment) comes first and is not optional.

**5 — Aftermath.** Record the serial as lost in ABM (**do not release it** —
keeping it in ABM is what re-captures it if it is ever wiped and set up again),
update the asset inventory, and if member or personal data was on an
unencrypted device, treat it as a privacy incident and escalate per
`Security Procedures/IR_Security_Scenarios_Guide.md`. `[FILL IN: who at the organization is
notified for a suspected privacy breach, and what is the reporting obligation
under applicable privacy legislation? This is a legal question, not an IT one — get the answer
from whoever owns privacy compliance and record it here.]`

Proposed control: a lost-device report goes to IT within 24 hours of discovery,
and identity containment happens within 1 hour of that report
`[CONFIRM: proposed default]`.

### Compliance reporting with the NIST check script

`Support Procedures/macos-nist-check.sh` is the measurement instrument the organization
already has. It runs against NIST SP 800-53 Rev 5 using only native tools, so
it needs nothing installed on the target, and it works whether or not an MDM
exists.

```
sudo bash "Support Procedures/macos-nist-check.sh"                        # run locally
ssh [FILL IN: admin account]@<mac> 'sudo bash -s' < macos-nist-check.sh    # run remotely over SSH
ssh [FILL IN: admin account]@<mac> 'sudo bash -s' < macos-nist-check.sh > "nist-$(date +%Y%m%d)-<mac>.txt"
```

The controls it tests map almost exactly onto the profile baselines proposed
above, which is the useful part — the script tells you whether the profiles are
doing what you think:

| Control | What it checks | Enforced by which payload |
|---------|----------------|---------------------------|
| AC-2 | Guest account, auto-login, root, login window style | Login window |
| AC-7 | Failed logon attempt limit | Password policy (AD) |
| AC-11 | Screensaver idle time, password required | Screen lock |
| AC-17 | SSH / remote access exposure | Restrictions |
| AC-20 | Internet Sharing, Remote Apple Events | Restrictions |
| CM-6 | SIP, Gatekeeper, automatic software updates | Gatekeeper, software update |
| CM-7 | Unnecessary services — file sharing, **printer sharing**, AirDrop, content caching | Restrictions |
| IA-5(1) | Password complexity via MDM profile — **explicitly reports whether a profile is present** | Password policy |
| MP-5 / SC-28 | FileVault | FileVault |
| SC-7 | Application firewall, stealth mode | Firewall |
| SI-2 | macOS version currency | Software update |
| SI-3 | XProtect | (built in) |
| CP-9 | Time Machine | `[FILL IN: is Time Machine used at the organization, or is user data expected to live on files.example.com?]` |
| PE-3 / PS | **MDM enrolment presence** | — |

Two rows do double duty. **IA-5(1)** reports whether a configuration profile is
setting password policy at all, and **PE-3/PS** reports whether the device is
enrolled in MDM. Running the script across the fleet therefore answers the
question at the top of this document empirically — if every Mac reports no
enrolment, there is no MDM.

Proposed cadence: run across the fleet quarterly, and on every machine at build
time as part of onboarding `[CONFIRM: proposed default]`. Keep the raw output
files; the diff between quarters is where configuration drift shows up. Without
an MDM the script is the *only* compliance evidence the organization has, which makes the
cadence more important, not less.

### If there is no MDM: evaluation and onboarding path *[Evaluation]*

The order below is deliberate. Each step is useful on its own and none of them
is wasted if the next one is deferred.

**Step 0 — Establish the facts.** Run the NIST script across the Mac fleet and
count the PE-3/PS results. Check whether anything exists at
`https://business.apple.com` under any organization-associated Apple ID. Ask the Apple
reseller whether the organization has an ABM Organisation ID on file — they will know, and
it is one email. Cost: a few hours.

**Step 1 — Create Apple Business Manager, regardless of the MDM decision.** ABM
is free. It takes a business verification (D-U-N-S number and a callback to a
named authority within the organisation) and a couple of days. Having it costs
nothing and creates the option; not having it means every Mac bought in the
meantime is permanently outside the supervision path. `[FILL IN: the organization's D-U-N-S
number and the person who will act as the verification contact]`

**Step 2 — Register the purchase channel.** Give the reseller the organization's ABM
Organisation ID and get their Reseller ID into ABM. From that point, new Macs
land in ABM automatically. This is the step that stops the problem getting
worse while the rest is decided.

**Step 3 — Evaluate MDM products.** Criteria that matter for the organization specifically:

| Criterion | Why it matters here |
|-----------|---------------------|
| Coexists cleanly with Munki | Munki is staying; the MDM must not duplicate or fight software delivery |
| FileVault personal key escrow and validation | The primary reason to do this at all |
| ADE / supervision support | Table stakes, but confirm the workflow |
| AD or Entra ID integration | The fleet is AD-bound; a second identity source is a cost |
| Cost per device per year, and the minimum seat count | `[FILL IN: fleet size drives this]` |
| Windows support | Whether one console covers both, or Intune handles Windows separately |
| Can it be operated by one person | The realistic constraint |
| Data residency | Member data context — `[FILL IN: does the organization have a data residency requirement? Ask whoever owns privacy compliance]` |

Candidates to price: Jamf Pro, Jamf Now, Mosyle, Kandji, Addigy, and the
open-source MicroMDM/NanoMDM path. `[FILL IN: get quotes; record the shortlist
and the decision in the Decisions table below]`. Note that the boilerplate
inventory file names Jamf — worth confirming whether that reflects a real past
evaluation or is template text.

**Step 4 — Pilot.** Enrol 3–5 Macs, IT-owned first. Push exactly one profile —
screen lock — and confirm it applies and that the NIST script's AC-11 check
flips to PASS. That single loop proves the whole chain works.

**Step 5 — FileVault escrow across the fleet.** The first real win. Reissue
recovery keys on enrolled machines with `fdesetup changerecovery -personal`, and
validate a sample.

**Step 6 — Roll out the remaining baselines**, one payload at a time, with the
NIST script as the before/after measurement.

**Step 7 — Update the checklists.** Onboarding and offboarding both need real
MDM steps rather than the current conditional "if using Jamf/Mosyle/Kandji".

**If the decision is not to adopt an MDM**, that is a legitimate outcome for a
small fleet — but it must be a decision, recorded below, with the accepted risks
named: no remote wipe, no key escrow, no enforced configuration, and no way to
re-establish control of a stolen Mac. And ABM should still be created (Step 1),
because it costs nothing and keeps the door open.

## Troubleshooting

### Symptom: device does not enrol at Setup Assistant

- Likely causes, in order of probability:
  1. **The serial is in ABM but not assigned to an MDM server.** The most common
     cause and invisible unless you look at the device record in ABM.
  2. The serial never reached ABM — the purchase did not go through the
     registered channel.
  3. No network at Setup Assistant, or the network is a captive portal.
  4. Apple's activation endpoints are blocked at pfSense.
  5. The device was previously released from ABM.
- Check:
  ```
  # In ABM: Devices -> search the serial -> confirm "Assigned to: <MDM server>"
  profiles status -type enrollment                 # on the device, post-setup
  sudo profiles renew -type enrollment             # force a re-check with Apple
  nc -zv albert.apple.com 443                      # device activation endpoint reachable?
  nc -zv gdmf.apple.com 443                        # Apple device management lookup
  nc -zv [FILL IN: MDM server hostname] 443        # the MDM itself reachable?
  ```
- Fix: assign the device in ABM, then erase and re-run Setup Assistant — the
  check happens once, at setup. `sudo profiles renew -type enrollment` can pick
  it up post-setup on some macOS versions but is not reliable; the erase is.
- **Network requirement**: Apple push notification service needs outbound TCP
  **5223** (with 443 fallback) to `17.0.0.0/8`, plus 443 to Apple's activation
  hosts. A firewall that permits 443 but not 5223 produces devices that enrol
  and then never respond to a command — which presents as "MDM is broken" rather
  than "a port is blocked." `[FILL IN: confirm pfSense permits TCP 5223 outbound
  from the workstation VLANs to 17.0.0.0/8. See the IPAM reference.]`

### Symptom: profile is not applying

- Likely causes:
  - The device has not checked in. Profiles are pushed; a sleeping or off-network
    device has not received it.
  - Scope: the device is not in the group the profile targets.
  - Two profiles set the same key. Last one wins, unpredictably. The most common
    real cause once scoping is ruled out.
  - The payload requires supervision and the device is manually enrolled.
  - A user-level payload is being delivered to a device with no user logged in.
- Check:
  ```
  profiles list -all                               # is the profile installed at all?
  sudo profiles -P                                 # full payload dump — look for a duplicate key
  log show --predicate 'subsystem == "com.apple.ManagedClient"' --last 2h --style compact
  log show --predicate 'process == "mdmclient"' --last 2h
  ```
- Fix: send a blank push from the console to wake the device, confirm scoping,
  and resolve duplicate keys by consolidating into one profile. Then re-run the
  NIST script to confirm the setting actually took effect — profile installed
  and setting enforced are two different claims.

### Symptom: device is in ABM but the assignment is missing

- Likely causes: released from ABM at some point (irreversible); assigned to a
  different MDM server; the reseller submitted it against the wrong organisation
  ID; or the 30-day provisional period from an Apple Configurator addition
  expired with the device released.
- Check: ABM → Devices → search the serial. The device record shows the
  assignment, the order date, and the source (Apple, reseller, or Configurator).
- Fix: reassign if it is present and unassigned. If it was released, the only
  route back is Apple Configurator with a physical wipe — and if the device was
  bought outside the registered channel, that route gives only the 30-day
  provisional state. Contact the reseller if the device should have been
  submitted and was not; they can usually resubmit.

### Symptom: supervision lost after a wipe

- **This should not happen on an ADE device, and if it has, one of these is true:**
  - The device was never actually supervised — it was manually enrolled, and
    manual enrolment does not survive an erase. Check whether it ever reported
    `Enrolled via DEP: Yes`.
  - The device was released from ABM before or during the wipe. Irreversible.
  - Setup Assistant ran with no network, so the ABM check never happened and was
    skipped. This one is recoverable.
  - The erase was done from Recovery or by DFU restore to a macOS version older
    than the ADE record expects.
- Check:
  ```
  profiles status -type enrollment                 # "Enrolled via DEP: No" on a supposedly supervised Mac is the finding
  sudo profiles show -type enrollment
  ```
- Fix: if it is still in ABM and assigned, erase again **on a working network**
  and let Setup Assistant re-check. If it has been released, it is gone —
  record it, and treat it as the reason releases require a second person to
  approve `[CONFIRM: proposed default]`.

### Symptom: FileVault key is not in the console

- Likely causes: FileVault was enabled by the user or by hand rather than
  through the profile; the escrow payload was not scoped to that device; the key
  was reissued locally after enrolment; or the device has not checked in since
  encryption completed.
- Check:
  ```
  sudo fdesetup status                             # encrypted?
  sudo fdesetup haspersonalrecoverykey             # does a personal key exist on this machine?
  sudo fdesetup validaterecovery                   # does the key you hold actually work?
  ```
- Fix: `sudo fdesetup changerecovery -personal` to issue a new key, then force a
  check-in and confirm it appears in the console. Then validate it. Add escrow
  verification to the build checklist so this is caught at provisioning rather
  than during a recovery.

### Symptom: a Mac is in AD but not in MDM (or vice versa)

- Not a fault in itself — it is the expected state at the organization today. It becomes a
  problem at retirement, when only one of the three records gets cleaned up.
- Check: reconcile three lists — AD computer objects, MDM device records, and
  the asset inventory. Anything in one and not the others is a finding.
  ```
  Get-ADComputer -Filter * -Properties LastLogonDate | Sort LastLogonDate | ft Name,LastLogonDate
  # ^ stale computer objects; cross-reference against the MDM device list and the inventory
  ```
- Fix: reconcile, and make the three-way cleanup an explicit step in the
  offboarding checklist. Proposed cadence: reconcile quarterly
  `[CONFIRM: proposed default]`.

## Security

- **Exposure**: the MDM console can execute arbitrary code as root on every
  managed Mac, and can erase them. It is the highest-value administrative
  surface in the fleet — comparable to Munki repo write access (which is the
  same power by a slower route) and to Domain Admin.
  `[FILL IN: who has MDM console access, at what role, and is MFA enforced?]`
- **ABM access**: an ABM Administrator can release every device from the
  organisation, permanently destroying supervision fleet-wide. There must be at
  least two Administrator accounts (so access is not lost with one person) and
  no more than necessary. `[FILL IN: current ABM role assignments]`
- **APNs certificate**: the MDM's push certificate expires **annually** and is
  tied to the Apple ID that created it. If it lapses, every device stops
  responding to commands. If it is renewed with a *different* Apple ID, every
  device must be re-enrolled — for supervised devices that means wiping the
  fleet. Consequences: use a role-based Managed Apple ID nobody will lose,
  record which one, and calendar the renewal 30 days out.
  `[FILL IN: APNs Apple ID and expiry date]` `[CONFIRM: proposed default — a
  calendar reminder 30 days before expiry, owned by the sysadmin]`
- **Secrets**: MDM console credentials, the ABM Managed Apple ID, the APNs
  Apple ID, FileVault recovery keys, and remote-lock PINs all live in the
  password manager. None of them belong in this document. See
  `Security & Hardening/secrets-management-guide.md`.
- **Certificates**: profiles are signed. If the organization delivers an internal CA or
  802.1x client certificates by profile, their lifecycle belongs in
  `Security & Hardening/certificate-pki-lifecycle-guide.md`.
- **Separation from AD**: MDM can enforce a local password policy. Do not — let
  AD own password policy and let MDM own device configuration. Two authorities
  over one credential produces lockouts nobody can diagnose.
- **The risk of not having any of this**, stated plainly so the decision is
  informed: no remote wipe on a lost machine, no escrowed encryption keys, no
  enforced screen lock or firewall, no way to prove a device belongs to the organization, and
  no reliable way to keep a stolen Mac from becoming someone else's Mac.

## Monitoring & Alerting

- Worth alerting on, whichever model is in place `[CONFIRM: proposed defaults]`:
  - A Mac that has not checked in to MDM (or to Munki) for **7 days** — matches
    the threshold already used in
    `Security & Hardening/endpoint-security-operations-guide.md`.
  - A device where FileVault is off, or on with no escrowed key.
  - A profile that fails to install on more than **3** devices — that is a bad
    profile, not three bad Macs.
  - Any device released from ABM.
  - APNs certificate within 30 days of expiry.
  - A new device enrolling that is not in the asset inventory.
- What normal looks like: every Mac checks in daily; FileVault on everywhere
  with a validated escrowed key; the NIST script's quarterly run shows no new
  FAILs relative to the previous quarter.
- `[FILL IN: can MDM or Munki check-in data be shipped to graylog01.example.com so
  absence alerting sits with the rest of the monitoring? See
  SysAdmin Procedures/monitoring-alerting-guide.md]`
- Absence is the signal that matters. A device that stops reporting is either
  off, gone, or unmanaged, and all three want investigating.

## Disaster Recovery

- **Losing the MDM console** means losing the ability to push configuration, wipe
  devices, and read escrowed FileVault keys. Devices keep working and keep their
  current profiles; nothing changes for users until something needs to change.
  The urgent loss is key escrow access.
- **Keep an export of every configuration profile outside the console**, so the
  estate can be rebuilt rather than re-derived. `[FILL IN: where profile exports
  are kept, and is that location backed up by Retrospect?]`
- **Losing ABM access** is worse than losing the MDM, because ABM is the root of
  supervision. Two administrators, and the Managed Apple IDs recorded in the
  password manager, are the mitigation.
- **Rebuild order** after losing the MDM: recover or recreate ABM access →
  stand up the MDM → generate a new APNs certificate **with the same Apple ID**
  → reassign devices in ABM to the new server → re-import profiles → devices
  re-enrol on next erase or on `profiles renew`.
- **The unpleasant truth about APNs**: if the certificate is renewed under a
  different Apple ID, every supervised device must be erased to re-enrol. Losing
  that Apple ID is a fleet-wide reimaging event. This is why it gets its own
  vault entry and its own calendar reminder.
- Escalation if the owner is unavailable: `[FILL IN: name and contact]`

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why / Ticket |
|------------|-------------------|--------------|
| 2026-09-11 | Guide created. Records that MDM and ABM presence at the organization is **unconfirmed**, and structures the content to work either way | The library had no device-management doc; the only MDM references were boilerplate or conditional |
| `[FILL IN]` | `[FILL IN: does the organization have an MDM — yes or no, and which]` | `[FILL IN]` |
| `[FILL IN]` | `[FILL IN: does the organization have Apple Business Manager — yes or no]` | `[FILL IN]` |
| `[FILL IN]` | `[FILL IN: if an MDM was evaluated and rejected, record that and the accepted risks — a recorded "no" is worth as much as a "yes"]` | `[FILL IN]` |
| `[FILL IN]` | `[FILL IN: where FileVault recovery keys are held today]` | `[FILL IN]` |

## References

- `Windows & Mac Workstations/munki-server-administration-guide.md` — software
  delivery; the complement to everything here
- `Windows & Mac Workstations/windows-11-desktop-runbook.md` — §11 Arm64, §12
  Parallels on Apple silicon
- `Windows & Mac Workstations/parallels-shared-folders-and-drive-mapping.md`
- `Support Procedures/macos-desktop-support-runbook.txt` — §12 FileVault, §18
  MDM troubleshooting, §22 Erase All Content and Settings, §23 Apple
  Configurator and DFU restore
- `Support Procedures/macos-nist-check.sh` — the compliance measurement
- `SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt` — the lifecycle
  this guide plugs into
- `SysAdmin Procedures/Inventory_Asset_Reference.txt` — has the "MDM Enrolled"
  column; currently unfilled boilerplate
- `Security & Hardening/endpoint-security-operations-guide.md` — fleet security
  operations, isolation, the same open MDM questions
- `Security & Hardening/secrets-management-guide.md` — where MDM, ABM, APNs and
  recovery-key secrets belong
- `Security & Hardening/certificate-pki-lifecycle-guide.md`
- `Active Directory/AD-Admin-Security-Guide.md` — AD binding, computer objects,
  account containment
- `Microsoft 365/m365-entra-admin-guide.md` — Entra ID join, Conditional Access,
  Intune considerations
- `Networking Guide/ipam-vlan-topology-reference.md` — the firewall policy
  matrix that must permit APNs egress
- Upstream: Apple Platform Deployment guide; Apple Business Manager User Guide;
  Apple's "Use MDM to deploy devices" documentation

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

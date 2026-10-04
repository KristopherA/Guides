> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Secrets Management

## Overview

A secret is anything that grants access and is not supposed to be known. The organization
holds a lot of them — AD service accounts, database credentials, SSH private
keys, API keys for the DNS provider and the off-site backup, network device
enable passwords, certificate private keys, and the break-glass accounts that
exist precisely for the day nothing else works. They are currently spread across
a vault, several documents, a few config files, and some people's memory.

This document defines where each class of secret belongs, how to share one
safely, how often each class is rotated, what to do when a secret is exposed,
and how offboarding triggers rotation. It also records two findings in this
documentation library itself that need remediation, because a guide that
describes good practice while the folder next to it holds credentials in the
clear is not much use.

Audience is IT Ops. Everything here applies to secrets Example Org administers; member
and staff personal passwords are a separate concern.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Owner            | IT lead (it@example.com) — process owner `[CONFIRM: proposed default]` |
| Environment      | prod, staging, lab — all organization-administered secrets                     |
| Location         | `[FILL IN: password manager / vault in use]` — `SysAdmin Procedures/Inventory_Asset_Reference.txt` Section 7 records service account passwords as living in "1Password / IT vault"; confirm whether that is the sanctioned store and record the canonical name and URL here |
| Access           | `[FILL IN: how the vault is accessed — URL, app, SSO or standalone]`; MFA `[FILL IN: is MFA enforced on the vault?]` |
| Dependencies     | Active Directory (for the accounts many secrets belong to); MFA provider `[FILL IN]`; the vault's own recovery mechanism |
| Dependents       | Effectively every documented procedure in this library that begins with "log in as" — backups, scanning, deployment, monitoring, LDAP-integrated applications, network device administration |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                               |

## How It Works

The model is simple and worth stating because most secret mishandling comes from
not having stated it:

```
Secrets live in the vault.            # one place, access-controlled, audited
Systems get them by reference.        # a config points at a path, not a value
People get them by grant.             # shared through the vault, time-limited
Nothing else ever holds one.          # not a doc, not a ticket, not a chat, not a repo
```

Three things follow from that. First, **the vault is the only source of truth**;
a password written anywhere else is a copy that will not be rotated and will
outlive its usefulness. Second, **every secret has an owner and a class**, and
the class determines rotation cadence and blast radius. Third, **a secret that
has been somewhere it should not be is compromised**, regardless of whether
anyone believes it was seen. That last rule is what makes the incident response
section short: there is no assessment step, there is a rotation step.

Where secrets are consumed by systems rather than people, the reference
mechanism matters:

```
Linux services   # environment file, mode 0600, owned by the service account — not in the unit file, which is world-readable
Windows services # gMSA where the service supports it, so there is no password to manage at all
Containers       # secret file or environment injected at run time — never baked into an image layer
Scripts          # read from a protected file at run time; never a literal in the script body
```

`[FILL IN: which of these mechanisms are actually in use at Example Org today]`

## What counts as a secret

If knowing it lets someone act as us, it is a secret. In practice that includes
things people do not instinctively treat as secret: a bind DN plus its password,
an API key in a URL, a Wi-Fi PSK, a recovery key, a session token pasted into a
ticket, a database connection string, a `.pem` file attached to an email.

It also includes **recovery material**: FileVault recovery keys, MFA recovery
codes, the vault's own break-glass credentials. These are secrets that grant
access by design, which makes them higher value, not lower.

Not secrets, though they are often confused for them: usernames on their own,
hostnames, IP addresses, certificate public halves, and bind DNs in isolation.
These are sensitive-ish — they help an attacker — but they are not credentials.
Treat them as internal information, not as vault items. The distinction matters
because classifying everything as a secret leads to classifying nothing.

## The classes Example Org holds, and where each lives

| Class | Examples at the organization | Stored in | Notes |
|-------|-----------------|-----------|-------|
| AD service accounts | `svc-backup`, `svc-monitoring`, `svc-deploy` (from the asset inventory); the LDAP bind account `CN=servicesadmin,CN=Users,DC=example,DC=com`; Nessus scan accounts | `[FILL IN: password manager / vault in use]` | Inventory Section 7 is the register of what exists; the vault holds the values. Prefer gMSA where the consuming service supports it |
| API keys / tokens | DNS provider API key, off-site backup (S3/B2) keys, M365 app credentials, VirusTotal API key used by the triage toolkit, Graylog API tokens used by Sidecar | `[FILL IN]` | Scope each key to the minimum permission; never embed in a URL |
| SSH private keys | Admin keys to the Ubuntu fleet, deployment keys, the Nessus Linux scan key | On the workstation/host that uses them, passphrase-protected; passphrases in `[FILL IN]` | Keys are per-person, never shared. `AllowGroups sshusers` and `PasswordAuthentication no` are already enforced in `sshd hardening conf.txt`, which makes the key the only way in |
| Database credentials | MySQL application and admin accounts | `[FILL IN]`; injected into apps by reference | Never in a repo, never in a wiki page |
| Certificate private keys | Wildcard `*.example.com` key/PFX, LDAPS and RADIUS server keys, MySQL server key | Generated on the consuming host where possible; exportable copies in `[FILL IN]` | Governed jointly with `certificate-pki-lifecycle-guide.md` |
| Network device credentials | pfSense admin, switch enable/privileged-exec passwords, wireless controller admin, Barracuda WAF admin | `[FILL IN]` | These are the ones most likely to be in someone's notes rather than the vault — audit for that specifically |
| RADIUS / 802.1x shared secrets | Shared secret between each NAS (switch, AP, controller) and the RADIUS server | `[FILL IN]` | Per-device secrets, not one shared everywhere `[CONFIRM: proposed default]` |
| Break-glass accounts | Emergency AD account, local admin on critical hosts, vault recovery, appliance root accounts | Sealed, offline — see Break-glass below | Deliberately not in daily-use storage |
| Recovery material | FileVault recovery keys, MFA recovery codes, BitLocker keys if used | FileVault keys escrowed to MDM; others in `[FILL IN]` | The offboarding and onboarding checklists already reference escrow |
| Wi-Fi / pre-shared keys | Guest network PSK, any PSK-based SSID | `[FILL IN]` | Rotate on staff departure if widely known |

`[FILL IN: any class Example Org holds that is not listed above]`

## Sharing rules

### How to share a secret with a colleague

1. Check whether they need the secret or need the access. Often the right answer
   is to add their own account to a group, not to hand over a shared credential.
   A shared credential is an audit trail you have just erased.
2. Share through the vault's own sharing mechanism, to a named person, not to a
   group or a link anyone can open.
   `[FILL IN: the vault's sharing feature and how to use it]`
3. Set an expiry on the share. `[CONFIRM: proposed default — 7 days]`
4. Record that you shared it, with whom and why, in the ticket.
5. If the secret is one that cannot be scoped per person — a device enable
   password, a shared appliance admin — rotate it when the person no longer
   needs it, not merely revoke the share.

If the vault cannot share to that person (an external party, a vendor), use a
one-time-view link from `[FILL IN: approved one-time-secret tool, if any]` with
the shortest viable expiry, and send the link and any passphrase over two
different channels. Then rotate as soon as the work is done.

### Never, under any circumstances

- **Email.** Mail is stored, backed up, indexed, forwarded, and read on phones.
  A secret sent by email is a secret in the backup set forever.
- **Chat.** Same, plus retention policies nobody controls and clients that sync
  to personal devices.
- **A ticket body or ticket comment.** Tickets are widely readable, exported,
  and long-lived. This includes "temporary" credentials in an onboarding ticket.
- **Plain text in a repository**, including private ones, including in history.
  Git history is forever; deleting the line does not remove it.
- **A documentation file**, including this library. See Findings below.
- **A config file in a world-readable location**, or a systemd unit file
  (`systemctl cat` shows it to everyone).
- **A screenshot.** It is still plain text, it is just harder to grep.
- **Command-line arguments**, which land in shell history and in the process
  table where any local user can read them with `ps`.

If a secret has been through any of these, it is exposed — go to Secret exposure
response below and rotate. Do not weigh up how likely it is that anyone saw it.

## Findings in this library — remediate

Two items in this documentation library need attention. Both were identified on
2026-09-11 and neither is a hypothetical.

**1. `Security & Hardening/ssh keys.rtf`**

Reviewed 2026-09-11: the file contains working notes on `ssh-add`, the macOS
Keychain, and `ssh-add -A` behaviour. **It does not appear to contain private key
material.** The problem is the filename and the format — a file called
"ssh keys.rtf" in a shared documentation folder invites both the assumption that
keys are kept there and the habit of putting them there. It is also an RTF,
which grep-based secret scanning handles poorly.

Remediation `[CONFIRM: proposed default]`:
- Re-verify the full contents before acting, then rename to something accurate,
  e.g. `ssh-agent-keychain-notes.md`, and convert to Markdown.
- Fold the content into a Linux/SSH access guide rather than leaving it loose.
- Re-scan the whole library for key material (see the audit step below) rather
  than assuming this was the only candidate.

**2. `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11)**

This file contains LDAP connection details in plain text including the bind
account distinguished name:

```
CN=servicesadmin,CN=Users,DC=example,DC=com      # bind DN for LDAP-integrated applications
```

The password is not in the file, which is the one thing that went right. But
naming the privileged bind account in a shared document hands an attacker the
exact target for a password-spray, and the file's existence strongly suggests
the password is written down somewhere adjacent.

Remediation `[CONFIRM: proposed default]`:
- Confirm where the `servicesadmin` password currently lives and move it to the
  vault if it is not already there.
- Rotate that password, since its age and distribution are unknown.
  `[FILL IN: which applications bind with servicesadmin — every one of them must
  be updated in the same window or LDAP auth breaks for that app]`
- Reduce `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11) to the connection parameters applications genuinely need
  (host, port 636, base DN, the UPN attribute) and replace the bind DN line with
  a pointer to the vault entry.
- Confirm the account is least-privilege — a read-only bind account, not a
  Domain Admin. `[FILL IN: current privilege level of servicesadmin]`

**Library-wide audit.** Do this once, then make it routine:

```
trivy fs --scanners secret "~/Documents/Guides and Manuals"   # secret scan across the whole library
grep -rInE "BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY" "~/Documents/Guides and Manuals"   # any embedded private key
grep -rInE "password[[:space:]]*[:=]|passwd[[:space:]]*[:=]|api[_-]?key|secret[[:space:]]*[:=]" "~/Documents/Guides and Manuals"   # credential-shaped lines, expect false positives
```

Anything found is treated as exposed: rotate first, clean the file second.
Trivy is already installed per `fresh-box-hardening-cheatsheet.txt`, and
Trufflehog/Gitleaks for repositories are covered in
`Security Procedures/IR_Security_Scenarios_Guide.md` scenario 4.

## Operations (Day-2)

### Rotation schedule

Rotation exists to bound the damage from an exposure nobody noticed. The cadence
should reflect blast radius, not tradition. All values proposed.

| Class | Rotate every | Also rotate on | Notes |
|-------|--------------|----------------|-------|
| AD service accounts | 365 days `[CONFIRM: proposed default]` | Offboarding of anyone who held it; suspected exposure | Long interval is acceptable only because these are long, random, and vault-held. gMSA accounts rotate automatically and need no entry here |
| Privileged / admin AD accounts | 180 days `[CONFIRM: proposed default]` | Offboarding; any endpoint compromise where the account was used | |
| API keys / tokens | 180 days `[CONFIRM: proposed default]` | Offboarding of the creator; vendor breach notification | Prefer keys that can be rotated without downtime — create the new key, cut over, then revoke the old |
| SSH private keys (per person) | 365 days `[CONFIRM: proposed default]` | Departure; workstation compromise or loss | Rotation means new keypair and removing the old public key from every `authorized_keys` |
| SSH deployment / automation keys | 180 days `[CONFIRM: proposed default]` | Any change to who can reach the automation host | |
| Database credentials | 180 days `[CONFIRM: proposed default]` | Offboarding; application handover | Coordinate with the app owner — this is the class most likely to cause an outage when rotated carelessly |
| Certificate private keys | At each certificate renewal — never reuse a key | Suspected compromise | Governed by `certificate-pki-lifecycle-guide.md` |
| Network device credentials (admin / enable) | 180 days `[CONFIRM: proposed default]` | Departure of anyone with device access; contractor engagement ending | The class most likely to be shared and least likely to be rotated |
| RADIUS shared secrets | 365 days `[CONFIRM: proposed default]` | Device replacement; suspected compromise | Rotate per device to avoid a fleet-wide outage |
| Break-glass accounts | 180 days `[CONFIRM: proposed default]` | Every use, without exception | See Break-glass below |
| Wi-Fi PSKs | 365 days `[CONFIRM: proposed default]` | Departure of staff who knew it | 802.1x SSIDs have no PSK and so no entry here |
| Vault master / recovery | `[FILL IN: vault recovery rotation policy]` | Departure of anyone holding it | |

Rotation is scheduled work, not ad-hoc. Put the due dates somewhere that nags:
`[FILL IN: where rotation due dates are tracked — calendar, ticket queue, or
vault expiry field]`

### Rotation procedure

The generic shape. The failure mode is always the same — the credential is
changed in one place and not the other, and something breaks at 3am.

**Step 1 — Find every consumer first.** Before changing anything, list every
system, script, cron job, config file and integration that uses this credential.
This is the step people skip and it is the whole job.

```
grep -rIn "<username>" /etc /opt /org-scripts 2>/dev/null      # config references on a host
crontab -l ; ls -la /etc/cron.d/                                # scheduled jobs that authenticate
systemctl list-units --type=service --state=running             # services that may hold it
```

For an AD service account, also check the account's recent logons in Graylog to
find consumers nobody documented. That is usually where the surprises are.

**Step 2 — Prefer add-then-remove over change-in-place.** Where the platform
allows two credentials to be valid at once — a second API key, a second SSH
public key, a second DB user — create the new one, migrate consumers, then
revoke the old. This turns a hard cutover into a reversible one and is worth the
extra ten minutes every time.

**Step 3 — Generate a new secret properly.**

```
openssl rand -base64 32                                         # random secret where a service will consume it
ssh-keygen -t ed25519 -C "<person>@example.com $(date +%F)" -f ~/.ssh/id_ed25519_new   # new SSH keypair, with a passphrase
```

Never reuse a secret across systems, and never derive the new one from the old
one by incrementing something.

**Step 4 — Store it in the vault before deploying it.** If the deployment fails
halfway and the new secret only existed in your terminal scrollback, the outage
is longer than it needed to be.

**Step 5 — Deploy to every consumer** identified in step 1, in a maintenance
window if any consumer is production. Follow change management —
`SysAdmin Procedures/Change_Management_Log.txt`.

**Step 6 — Verify.** Not "the service started" — actually exercise the
credential path.

```
systemctl status <service> --no-pager                                    # service is up
journalctl -u <service> --since "10 min ago" | grep -iE "auth|denied|fail"   # nothing is failing quietly
ldapsearch -H ldaps://auth2.example.com:636 -D "<bind DN>" -W -b "<base>" -s base   # LDAP bind actually works with the new credential
```

Then wait one full cycle of anything scheduled — a nightly backup that
authenticates will not fail until tonight. A rotation is not complete until the
next scheduled run using it has succeeded. `[CONFIRM: proposed default — hold the
old credential for 24 hours before revoking]`

**Step 7 — Revoke the old secret.** This is a step, not an afterthought. An old
credential left valid means the rotation achieved nothing.

**Step 8 — Record it.** Update the vault entry's rotation date and the owner
field, and update Section 7 of `SysAdmin Procedures/Inventory_Asset_Reference.txt`.

### Offboarding-triggered rotation

`SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt` already handles the
departing person's *own* access well — AD password reset within one hour,
account disabled, sessions killed, equipment recovered, and a deferred cleanup
pass at 30–90 days. That covers the accounts that belong to them.

What it does not fully close is the **shared** credentials they knew. Disabling
someone's account does nothing about a switch enable password they memorised.
Add this as an explicit step in the offboarding checklist
`[CONFIRM: proposed default]`:

Within 24 hours of departure `[CONFIRM: proposed default]`, rotate:

- Every network device admin and enable password they had access to.
- Every shared appliance admin account (pfSense, Barracuda WAF, wireless
  controller, Nessus console, Graylog admin, MDM console).
- Any service account whose password they personally knew — check the "Owner"
  column in Inventory Section 7, and the vault's own sharing history for items
  shared with them.
- Any API key they created or held.
- Wi-Fi PSKs on any PSK-based SSID they knew.

Within 7 days `[CONFIRM: proposed default]`:

- Remove their SSH public key from every `authorized_keys` on every host.
  `sshd hardening conf.txt` sets `AllowGroups sshusers`, so removing them from
  that AD group is the fast containment; key removal is the durable fix and both
  should happen.
- Reassign ownership of every vault item, service account and scheduled task
  they owned. The offboarding checklist's Step 2 already asks about service
  accounts they managed — the answer must produce vault ownership changes, not
  just a note.
- Revoke vault access and confirm the revocation.

Escalate the whole set to **immediate** rather than 24 hours if the departure was
involuntary or contentious. That judgement belongs to the process owner and does
not need anyone's approval to exercise.

### Break-glass accounts

Break-glass accounts exist for the case where normal authentication is
unavailable — AD is down, the vault is unreachable, MFA is broken, or an
administrator account has been compromised. They are the accounts that must work
when nothing else does, which makes them both essential and the most dangerous
credentials Example Org holds.

Which accounts these are: `[FILL IN: the list of break-glass accounts —
emergency AD account, local admin on critical hosts, vault recovery, appliance
root accounts]`

Handling rules:

- **Stored offline and sealed.** A printed or written copy in a sealed, signed,
  tamper-evident envelope in `[FILL IN: physical location — safe or lockbox]`,
  with access controlled by `[FILL IN: who can open it]`. Storing break-glass
  credentials only in the vault defeats the purpose, because "the vault is
  unavailable" is one of the scenarios they exist for.
- **Two-person control** where feasible: `[CONFIRM: proposed default — two
  people required to open the envelope, recorded at the time]`.
- **Excluded from routine use.** If a break-glass account is used for
  convenience, it stops being break-glass.
- **Monitored.** Any authentication by a break-glass account generates a
  high-priority alert in Graylog, day or night. This is the highest-value single
  alert in the environment, because there is no benign reason for it to fire
  unnoticed. `[FILL IN: Graylog alert destination for break-glass authentication]`
- **MFA exclusions documented.** Break-glass accounts often must be excluded
  from Conditional Access or MFA to work during an outage. That exclusion is a
  deliberate, recorded risk, not an oversight.
  `[FILL IN: which break-glass accounts are excluded from MFA/Conditional Access]`

**Seal check** — quarterly `[CONFIRM: proposed default]`:

1. Confirm the envelope is present, sealed, and the signature across the seal is
   intact and matches the record.
2. Confirm the sealed-on date, and that the contents list matches the register.
3. Do **not** open it to verify the password unless the seal is already broken,
   the contents are out of date, or a scheduled rotation is due.
4. Record the check: date, who checked, seal intact yes/no.
5. If the seal is broken or missing, treat it as a use of the account: rotate
   immediately, investigate who opened it and why, and re-seal.

**After any use**, planned or not: rotate the credential, re-seal, record what it
was used for, and — if the use was unplanned — open an incident. An unexplained
break-glass authentication is a compromise until proven otherwise.

**Test annually** `[CONFIRM: proposed default]` that each break-glass credential
actually works. An untested emergency credential is a rumour; discovering during
an outage that the emergency account was disabled by a policy change six months
ago is a bad day made much worse. Rotate and re-seal after every test.

### Secret exposure — incident response

Triggers: a secret found in a repository, a ticket, an email, a chat, a log, a
screenshot, a documentation file, or a file shared externally; a workstation or
server compromise on a machine holding secrets; a vendor breach notification; a
departing person who may have retained material.

There is no assessment phase. If it was exposed, it is compromised.

**Step 1 — Contain the exposure, but do not destroy evidence.** Revoke or
disable the credential first where you can do so without losing the record of
what happened. For a repository exposure, revoke the key *before* rewriting
history — purging the commit does not un-leak the secret and destroys the
timeline.

**Step 2 — Rotate,** following the rotation procedure above. Skip step 2's
graceful add-then-remove if the exposure is active and severe; accept the
outage.

**Step 3 — Determine the blast radius.** What did the secret grant? Where else
was the same or a similar secret used? Reused credentials are the usual reason a
small exposure becomes a large one.

**Step 4 — Look for use.** This is the part that turns an exposure into an
incident or clears it.

```
# Graylog: authentications by the exposed account, all sources, since the earliest possible exposure date
# look for: unfamiliar source IPs, out-of-hours activity, geographies we do not operate in,
#           successful auth from a host that has no business using this account
```

`[FILL IN: Graylog stream/query for authentication events by account]`

For AD accounts, also check for new logon locations and any privilege change on
the account. For API keys, check the provider's own audit log — most keep one,
and it is often the only place misuse is visible.

**Step 5 — If there is any evidence of use, this is an incident.** Stop here and
go to `Security Procedures/IR_Security_Scenarios_Guide.md` — scenario 4 (Exposed Credentials /
Secrets in Code) for the credential itself, scenario 5 (Compromised User
Account) if it is an account, and the endpoint isolation procedure in
`endpoint-security-operations-guide.md` if a host is implicated.

**Step 6 — Clean up the exposure.** Purge from git history, delete the file,
redact the ticket, clear the chat message — after revocation, never instead of
it.

**Step 7 — Fix the cause.** A secret in a repository means there is no pre-commit
scanning; scenario 4 of the IR guide covers installing a Gitleaks hook. A secret
in a ticket means the intake process invites it. A secret in a documentation file
means the library needs the periodic scan described above. Rotating without
fixing the cause guarantees a repeat.

**Step 8 — Record it** in Decisions & History and in the change log.

## Troubleshooting

### Symptom: a service fails to authenticate immediately after a rotation

- Likely cause: a consumer was missed in step 1, or a config caches the old value.
- Check: `journalctl -u <service> --since "30 min ago" | grep -iE "auth|denied|credential"` — the error usually names the account.
- Fix: update the missed consumer and restart it. If the old credential is still
  valid, this is recoverable; if you already revoked it, this is why step 7 comes
  last.

### Symptom: a scheduled job fails the night after a rotation, though everything looked fine

- Likely cause: cron jobs and backups authenticate on their own schedule and do
  not fail until they run.
- Check: the job's log for the first run after the rotation — `/var/log/backup-linux.log` and the other paths in Inventory Section 7.
- Fix: update the job's credential reference. Then adopt the 24-hour hold before
  revoking the old secret, which exists to catch exactly this.

### Symptom: LDAP-integrated applications all stop authenticating at once

- Likely cause: the shared bind account password changed and only some
  applications were updated.
- Check: `ldapsearch -H ldaps://auth2.example.com:636 -D "<bind DN>" -W -b "<base>" -s base` — proves whether the credential itself is good, separating a credential problem from an application problem.
- Fix: update every application that binds. This is why the bind account's
  consumer list needs to be written down before the first rotation, not during
  it. Related: `Security & Hardening/LDAP Slow Down troubleshooting.txt`.

### Symptom: nobody can reach the vault

- Likely cause: vault outage, SSO/IdP outage, or expired MFA enrolment.
- Check: `[FILL IN: vault status page or health check]`
- Fix: this is the break-glass scenario. Use the sealed material, then treat it
  as a use: rotate and re-seal. Do not respond by making a local copy of vault
  contents "just in case" — that is how secrets end up in documents.

### Symptom: a break-glass authentication alert fires and nobody claims it

- Likely cause: either a legitimate use nobody recorded, or a compromise.
- Check: seal integrity, the physical access log, and the source of the authentication in Graylog.
- Fix: treat as compromise until someone accounts for it. Rotate, re-seal,
  investigate.

### Symptom: a secret is needed urgently and the only known copy is in an email

- Likely cause: the historical practice this document is written to end.
- Check: n/a.
- Fix: use it if the outage demands it, then rotate it the same day, store the
  new value in the vault, and delete the email. The rotation is not optional —
  the secret's confidentiality is already gone.

## Security

- Exposure: the vault is the highest-value system Example Org operates. Compromise of it
  is compromise of everything.
- Vault access: MFA mandatory `[CONFIRM: proposed default]`; access reviewed
  quarterly `[CONFIRM: proposed default]`; `[FILL IN: who currently has vault
  access, and at what level]`.
- Least privilege: every service account gets the minimum rights its job
  requires. The `servicesadmin` bind account in particular should be read-only.
- Audit: the vault's own access log should be reviewed
  `[CONFIRM: proposed default — monthly]`, and ideally shipped to Graylog.
  `[FILL IN: does the vault support log export to Graylog?]`
- Never store the vault's own recovery material in the vault.
- Secrets in this document: none, deliberately, and none should ever be added.
  This document describes where secrets live; it does not hold any.

## Monitoring & Alerting

- Alert immediately: any break-glass account authentication; any authentication
  by a service account from an unexpected source IP; vault access from an
  unfamiliar location; failed-authentication spikes on service accounts
  (password spray against a bind DN that is written down in a shared file is the
  specific risk the `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11) finding creates).
- Alert on schedule: secrets past their rotation due date; vault items with no
  owner; vault items not accessed in 12 months (probably orphaned, possibly
  still live).
- Existing signal: fail2ban and Snort already alert into Graylog; service
  account authentication patterns belong in the same place.
- `[FILL IN: Graylog stream and alert destination for secret-related events]`
- Baseline: `[FILL IN: normal authentication pattern per service account — which
  sources, what frequency]`. Without a baseline, "unexpected source" is not
  something anyone can act on.

## Disaster Recovery

- Vault unavailable: break-glass procedure above. RTO for restoring vault access:
  4 hours `[CONFIRM: proposed default]`.
- Vault data loss: `[FILL IN: vault backup/export mechanism and where the export
  is stored]`. An encrypted export held offline is the only defence against
  losing every credential at once, and it must be tested — an untested export is
  a rumour.
- Test the restore annually `[CONFIRM: proposed default]` and record the date.
- If the process owner is unavailable, break-glass access and vault recovery must
  still be exercisable. `[FILL IN: second person with break-glass and vault
  recovery access]` — a single point of failure here is unacceptable and is
  probably the most urgent item in this document.

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why / Ticket |
|------------|-------------------|--------------|
| 2026-09-11 | Created this guide; recorded two findings in the documentation library (`ssh keys.rtf`, `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11)) | No single document defined storage, sharing, rotation or exposure response |
| `[FILL IN]` | `[FILL IN: record rotations, exposures and policy changes here]` | `[FILL IN]` |

## References

- `SysAdmin Procedures/Inventory_Asset_Reference.txt` — Section 7, service account and scheduled task register
- `SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt` — access revocation on departure; this guide adds the shared-credential rotation step
- `SysAdmin Procedures/Change_Management_Log.txt` — change record for rotations
- `Security & Hardening/sshd hardening conf.txt` — key-only SSH, `AllowGroups sshusers`
- `Security & Hardening/ssh keys.rtf` — finding 1 above; ssh-agent/Keychain notes pending rename
- `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11) — finding 2 above; bind DN in plain text
- `Active Directory/AD-Admin-Security-Guide.md` — AD attack paths that begin with a credential
- `Security & Hardening/certificate-pki-lifecycle-guide.md` — certificate private keys
- `Security & Hardening/vulnerability-management-process.md` — scan credentials and their privilege
- `Security & Hardening/endpoint-security-operations-guide.md` — host compromise implicating stored secrets
- `Security Procedures/IR_Security_Scenarios_Guide.md` — scenario 4 (Exposed Credentials / Secrets in Code), scenario 5 (Compromised User Account)
- `Security & Hardening/fresh-box-hardening-cheatsheet.txt` — Trivy, including `--scanners secret`

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Zimbra Mail Server Administration — mail.example.com

---

## Overview

Zimbra is the organization's mail platform. It runs on `mail.example.com` and handles inbound and outbound mail for
`@example.com`, webmail, calendars, and shared mailboxes for reception and departmental addresses.
Staff reach it through the Zimbra web client and through desktop/mobile clients over IMAP, POP and
ActiveSync. Authentication is against Active Directory over LDAPS (`auth2.example.com` / `auth4.example.com`).
Mail logs are shipped to Graylog at `graylog01.example.com`, and `fail2ban` on the mail host bans hosts
that brute-force SMTP/IMAP logins.

This is the doc to open during a mail outage. Start at the triage runbook at the end.

## Quick Facts

| Field            | Value                                                                               |
|------------------|-------------------------------------------------------------------------------------|
| Owner            | IT lead (it@example.com), IT                                               |
| Environment      | prod                                                                                 |
| Location         | `mail.example.com` — [FILL IN: physical/VM host, Proxmox node or VM/CT ID, and internal IP] |
| Access           | SSH as `[FILL IN: admin login used to reach the host]`, then `su - zimbra` for all Zimbra commands. Admin console: `https://mail.example.com:7071`. Webmail: `https://mail.example.com` |
| Dependencies     | AD/LDAP (`auth2.example.com`, `auth4.example.com`, LDAPS/636); public DNS for `example.com` (MX, SPF, DKIM, DMARC); TLS certificate for `mail.example.com`; pfSense edge NAT/firewall; upstream relay if used ([FILL IN: smarthost/relay, or "none — direct delivery"]) |
| Dependents       | All staff mail and calendaring; application notification mail from [FILL IN: which systems send through mail.example.com — e.g. DMS, forums.example.com, sign.example.com, monitoring] |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                                              |

| Item                    | Value                                            |
|-------------------------|--------------------------------------------------|
| Zimbra edition/version  | [FILL IN: `zmcontrol -v` output]                  |
| Deployment shape        | [FILL IN: single-server all-in-one, or split LDAP/MTA/mailstore across hosts] |
| Mailstore path          | [FILL IN: typically /opt/zimbra/store — confirm]  |
| Mailbox count           | [FILL IN: `zmprov -l gaa \| wc -l`]               |
| Backup mechanism        | [FILL IN: Zimbra network-edition backup, or rsync/snapshot/Retrospect] |

## How It Works

Zimbra is a bundle of separate services managed together. Mail arrives at the MTA (Postfix,
wrapped by Zimbra), is filtered by the anti-spam/anti-virus chain, then handed to the mailstore
(the Java mailbox server plus its MariaDB metadata database) which writes the message to disk and
indexes it. The proxy tier fronts IMAP/POP/HTTP so clients only ever talk to one address.

    internet -> pfSense (NAT 25/465/587/993/995/443) -> mail.example.com
        postfix (MTA)  -> amavis -> clamav + spamassassin   # filtering chain
            -> mailboxd (mailstore)  -> MariaDB metadata + /opt/zimbra/store  # message + index
        zimbra-proxy (nginx) -> mailboxd                     # webmail and IMAP/POP client access
        zimbra-ldap                                          # Zimbra's own OpenLDAP: accounts, COS, config
        external auth -> ldaps://auth2.example.com:636            # password check against AD

Two directories are involved and confusing them wastes time. Zimbra has its **own** internal
OpenLDAP that holds accounts, distribution lists, Class of Service (COS) definitions and all server
config — that is what `zmprov` talks to. Separately, the domain is configured for **external
auth**, so when a user logs in, Zimbra binds to AD (`ldaps://auth2.example.com:636`, base
`CN=Users,DC=example,DC=com`, bind DN `CN=servicesadmin,CN=Users,DC=example,DC=com`, login attribute
`UserPrincipalName` — see `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11)) to verify the password. An account must
exist in Zimbra *and* in AD for a user to log in.

Config lives in the Zimbra LDAP (read/write with `zmprov`), not in scattered text files — editing
`/opt/zimbra/conf/*` by hand is usually wrong because Zimbra regenerates those files from LDAP.
Logs go to `/var/log/zimbra.log` on the host and are forwarded to `graylog01.example.com`.

---

## Setup / Installation

Not a build-from-scratch doc — the server exists. What matters operationally:

- All Zimbra commands run as the `zimbra` user, never as root:

      su - zimbra          # every command below assumes you have done this first

- Zimbra owns its own copies of Postfix, OpenLDAP, MariaDB, nginx and Jetty under `/opt/zimbra`.
  Do not install distro packages of those alongside it, and do not `systemctl` them individually —
  use `zmcontrol`.
- Rebuild reference: [FILL IN: whether a build/restore runbook exists elsewhere, and the OS version of the host].

## Configuration

| Setting | Value | Reason |
|---------|-------|--------|
| External LDAP auth against AD | `ldaps://auth2.example.com:636` | Single password for staff; AD remains the source of truth |
| Auth bind DN | `CN=servicesadmin,CN=Users,DC=example,DC=com` | Service account with read rights on `CN=Users` |
| Login attribute | `UserPrincipalName` | Matches the UPN users already know from Windows sign-in |
| Failover LDAP host | `auth4.example.com` | [FILL IN: confirm auth4 is listed as a secondary in `zimbraAuthLdapURL`] |
| Relay host | [FILL IN: relay host, or "none"] | [FILL IN: direct delivery, or relay through an upstream] |
| Max message size | [FILL IN: `zmprov gcf zimbraMtaMaxMessageSize`] | [FILL IN: why this value] |
| Spam thresholds | [FILL IN: `zmprov gcf zimbraSpamTagPercent zimbraSpamKillPercent`] | [FILL IN: why these thresholds were chosen] |

Read the live auth config rather than trusting the table:

    zmprov gd example.com | grep -i ldap          # shows zimbraAuthMech and the AD URLs/bind DN in use
    zmprov gacf | grep -i zimbraMta          # global MTA settings

---

## Operations (Day-2)

### Start / Stop / Restart — zmcontrol

    zmcontrol status                  # per-service status; the first thing to run on any mail complaint
    zmcontrol restart                 # full stop/start of all services, several minutes of downtime
    zmcontrol stop                    # stops everything
    zmcontrol start                   # starts everything
    zmcontrol -v                      # version and build
    zmmailboxdctl restart             # restarts only the mailstore (webmail/IMAP), leaves MTA queueing
    zmmtactl restart                  # restarts only Postfix/MTA
    zmproxyctl restart                # restarts the nginx proxy tier
    zmamavisdctl restart              # restarts the filtering chain

Prefer restarting the single failing service over a full `zmcontrol restart`. A full restart
during business hours disconnects every client.

### Account administration — zmprov

    zmprov -l gaa                                          # list all accounts (local LDAP, fast)
    zmprov -l gaa | wc -l                                  # mailbox count
    zmprov ga user@example.com                                  # dump every attribute of one account
    zmprov ga user@example.com zimbraAccountStatus              # just the status: active/locked/closed
    zmprov ca user@example.com '' displayName "First Last"      # create account, empty password = external auth
    zmprov ma user@example.com zimbraAccountStatus locked       # lock an account without deleting it
    zmprov ma user@example.com zimbraAccountStatus active       # unlock
    zmprov ma user@example.com zimbraMailQuota 5368709120       # set a 5 GB quota (bytes)
    zmprov ma user@example.com +zimbraMailAlias old.name@example.com # add an alias (the + appends)
    zmprov ra old@example.com new@example.com                        # rename an account
    zmprov da user@example.com                                  # DELETE the account and its mail — irreversible
    zmprov sp user@example.com 'newpassword'                    # set a local password; only meaningful if not external auth

With external auth to AD, passwords are set in AD, not here. `zmprov sp` on an externally
authenticated account creates confusion — the AD password still wins at login.

Class of Service controls quotas and features for a group of accounts:

    zmprov gac                                             # list all COS
    zmprov gc default                                      # show the default COS settings
    zmprov ma user@example.com zimbraCOSId <cos-id>             # move an account to a different COS

COS in use here: [FILL IN: COS names and what each is for].

### Distribution lists

    zmprov gadl                                            # list all distribution lists
    zmprov gdl list@example.com                                 # show a list and its members
    zmprov cdl list@example.com                                 # create a distribution list
    zmprov adlm list@example.com user@example.com                    # add a member
    zmprov rdlm list@example.com user@example.com                    # remove a member
    zmprov mdl list@example.com zimbraMailStatus disabled       # stop the list delivering without deleting it

To restrict a list to internal senders only:

    zmprov mdl list@example.com zimbraDistributionListSubscriptionPolicy ACCEPT   # [FILL IN: confirm the sender-restriction attributes actually used on Example Org lists]

### Shared mailbox delegation

The Zimbra model is a **grant on a mailbox folder** plus a **mountpoint** in the delegate's
account — not an Exchange-style permissions object. The GUI path is documented by the screenshots
in `Share Mailboxes Zimbra/` (two screenshots dated 2026-06-05 showing the sharing dialog);
this is the CLI equivalent.

    zmmailbox -z -m shared@example.com modifyFolderGrant /Inbox account user@example.com rwidx
        # grants user@example.com read/write/insert/delete/admin on the shared Inbox
    zmmailbox -z -m user@example.com createMountpoint /Shared-Reception shared@example.com /Inbox
        # mounts that shared Inbox into the delegate's folder tree
    zmmailbox -z -m shared@example.com gaf
        # get all folders, to confirm the grant landed on the right folder
    zmprov ma shared@example.com +zimbraPrefAllowAddressForDelegatedSender user@example.com
        # allows the delegate to send as the shared address

Permission letters: `r` read, `w` write, `i` insert, `d` delete, `x` workflow/admin. Granting on
`/Inbox` only shares the Inbox — subfolders and Sent need their own grants, which is the usual
cause of "I can see mail but not the Sent folder".

Shared mailboxes currently in use: [FILL IN: list of shared mailboxes and their delegates].

### Mail queue inspection and flushing

    zmqstat                              # queue counts per queue — quickest "is mail stuck?" check
    postqueue -p                         # full queue listing with recipients and reasons
    postqueue -p | tail -1               # just the total count
    postqueue -f                         # flush: retry every deferred message now
    postcat -q <queue-id>                # read one queued message, headers and body
    postsuper -d <queue-id>              # delete one message from the queue
    postsuper -d ALL deferred            # delete everything deferred — destructive, be certain
    mailq                                # same as postqueue -p

Queue names mean different things: `incoming` and `active` moving is normal; a growing `deferred`
queue means the remote end is refusing or unreachable; `hold` is manual quarantine; `corrupt` needs
investigation. A few hundred deferred messages after an internet blip is normal; thousands from one
sender usually means a compromised account relaying spam — check before flushing.

### Spam and RBL handling

    zmprov gacf zimbraSpamCheckEnabled zimbraVirusCheckEnabled    # confirm filtering is on
    zmprov gacf zimbraMtaRestriction                              # lists active RBLs and restrictions
    zmprov +zimbraMtaRestriction "reject_rbl_client zen.spamhaus.org"   # add an RBL
    zmprov -zimbraMtaRestriction "reject_rbl_client <list>"        # remove an RBL
    zmamavisdctl restart                                          # apply filtering changes

RBLs in use: [FILL IN: current `zimbraMtaRestriction` values].

Retraining the Bayes filter from the spam/ham accounts:

    zmtrainsa                            # runs the spam/ham training against the training accounts
    zmprov gacf zimbraSpamIsSpamAccount zimbraSpamIsNotSpamAccount   # the two training addresses

Whitelisting a sender that keeps getting tagged:

    zmprov ma user@example.com +amavisWhitelistSender sender@example.com   # per-user whitelist
    # For domain-wide, prefer fixing the sender's SPF/DKIM — see email-authentication-spf-dkim-dmarc.md

**If the organization's own IP gets listed on an RBL**: check the queue and Graylog for an outbound spam spike
first, lock the compromised account (`zimbraAccountStatus locked`), purge the spam from the queue,
then request delisting. Delisting before fixing the cause gets you relisted.
Outbound public IP: [FILL IN: the public IP mail.example.com sends from, per pfSense outbound NAT].

### SSL certificate renewal for the mail host

Zimbra keeps its own cert store and deploys the cert to all services at once; copying files into
nginx or Postfix by hand does not work.

    zmcertmgr viewdeployedcrt                       # shows what is deployed and its expiry — check this first
    zmcertmgr createcsr comm -new -subject "/C=US/ST=State/L=City/O=Example Org/CN=mail.example.com"
        # generates a CSR; SANs added with -subjectAltNames if needed
    # submit the CSR to the CA, receive the signed cert and chain
    cp <signed.crt> /opt/zimbra/ssl/zimbra/commercial/commercial.crt      # place the signed cert
    cp <chain.crt> /opt/zimbra/ssl/zimbra/commercial/commercial_ca.crt    # place the CA chain bundle
    zmcertmgr verifycrt comm /opt/zimbra/ssl/zimbra/commercial/commercial.key /opt/zimbra/ssl/zimbra/commercial/commercial.crt /opt/zimbra/ssl/zimbra/commercial/commercial_ca.crt
        # MUST pass before deploying — verifies key/cert/chain match
    zmcertmgr deploycrt comm /opt/zimbra/ssl/zimbra/commercial/commercial.crt /opt/zimbra/ssl/zimbra/commercial/commercial_ca.crt
        # installs into every Zimbra service's keystore
    zmcontrol restart                                # required for all services to pick up the new cert
    openssl s_client -connect mail.example.com:443 -servername mail.example.com | openssl x509 -noout -dates
        # verify from outside afterwards

Certificate details: [FILL IN: CA/issuer, expiry date, and whether the CSR SAN list includes anything beyond mail.example.com].
Set a calendar reminder 30 days before expiry. An expired mail cert breaks every desktop client at
once, usually first thing in the morning.

The LDAPS certificate on the AD side is a separate renewal with its own gotchas — see
`Security & Hardening/LDAPs certificate replacement procedure.txt`. Zimbra authentication breaks if
that one expires even though the mail cert is fine.

### Graylog queries for mail troubleshooting

Graylog at `graylog01.example.com` holds the forwarded `zimbra.log`. Stream/index:
[FILL IN: the Graylog stream name and index set used for mail.example.com logs].

Useful searches (adjust field names to match the actual extractors —
[FILL IN: confirm whether messages are parsed into fields or searched as full text]):

    message:"user@example.com"                                  # everything touching one address
    message:"status=bounced"                               # bounces in the window
    message:"status=deferred"                              # deferral reasons, grouped by remote server
    message:"NOQUEUE: reject"                              # mail rejected at the door and why
    message:"authentication failed"                        # failed logins — pairs with fail2ban bans
    message:"<message-id>"                                 # trace one message end to end
    message:"relay=" AND message:"status=sent"             # confirm delivery to a specific remote

Tracing a specific message: search the Message-ID, then follow the Postfix queue ID that appears
in the same line — every subsequent log line for that message carries the queue ID.

The existing PDF `[internal KB: unban a user in Graylog for the mail host]` documents the
Graylog-side view of bans with screenshots.

### fail2ban interaction and unban

fail2ban watches the auth failures in the mail log and bans source IPs at the firewall. The common
support call is a staff member at a remote site who fat-fingered a password several times and is now
locked out from everywhere on that IP.

    fail2ban-client status                                 # lists active jails
    fail2ban-client status <jail>                          # banned IPs in that jail
    fail2ban-client set <jail> unbanip 203.0.113.10        # unban a single IP
    fail2ban-client unban --all                            # clears every ban — use only after a false-positive storm
    grep 'Ban ' /var/log/fail2ban.log | tail -50           # recent bans with timestamps
    iptables -L -n | grep 203.0.113.10                     # confirm the ban rule is actually gone

Jails configured on this host: [FILL IN: jail names, findtime, maxretry, bantime from /etc/fail2ban/jail.local].

Before unbanning, confirm it is really a staff member — a ban firing repeatedly from an unfamiliar
IP is the system working. Two existing PDFs cover this workflow with screenshots:
`[internal KB: fail2ban unban procedure]` and
`[internal KB: unban a user in Graylog for the mail host]`. A separate ban layer exists at the
edge — Snort on pfSense — and an IP can be blocked there instead; see
`[internal KB: unblocking an IP from Snort]`. If unbanning
in fail2ban does not restore access, check Snort next.

### Backup & Restore of the mailstore

What is backed up: [FILL IN: what exactly is backed up — full mailstore, LDAP, config], to
[FILL IN: destination — Synology files.example.com, Retrospect, other], on [FILL IN: schedule], by
[FILL IN: mechanism].

A consistent backup needs the LDAP data and the config as well as the message store. Minimum set:

    zmcontrol stop                                         # for a cold copy; only for a full offline backup
    /opt/zimbra/libexec/zmslapcat /backup/ldap             # dumps Zimbra's internal LDAP
    zmprov -l gaa > /backup/account-list.txt               # account inventory, useful during a rebuild
    tar czf /backup/zimbra-conf.tgz /opt/zimbra/conf       # config snapshot
    zmcontrol start

Per-mailbox export and import, which covers most real restore requests:

    zmmailbox -z -m user@example.com getRestURL "//?fmt=tgz" > /backup/user.tgz          # export one mailbox
    zmmailbox -z -m user@example.com postRestURL "//?fmt=tgz&resolve=skip" /backup/user.tgz   # import it back, skipping existing items
    zmmailbox -z -m user@example.com getRestURL "/Inbox?fmt=tgz&query=after:1/1/2026" > /backup/inbox-2026.tgz   # export a subset

Restore procedure last tested: [FILL IN: date of last tested restore]. An untested restore is a
rumour — the template says so and it is right.

If this is Zimbra Network Edition, `zmbackup`/`zmrestore` are available and preferred:
[FILL IN: confirm edition — Network Edition has zmbackup, open-source Foss does not].

Related: `_ARCHIVE/superseded-stubs/Retrieve files from synology snapshot.txt` (archived 2026-09-11),
`SysAdmin Procedures/Backup_DR_Runbook.txt`.

### Updates / Upgrades

- Zimbra patches are applied with the installer against the same version tree; read the release
  notes for the exact source-to-target path, because Zimbra does not support arbitrary version jumps.
- Snapshot the VM before starting: [FILL IN: snapshot procedure/host for mail.example.com].
- Rollback is the snapshot. There is no clean in-place downgrade.
- Current version: [FILL IN: `zmcontrol -v`]. Patch level and last upgrade date: [FILL IN: patch level and date of last upgrade].

---

## Troubleshooting

### Symptom: nobody can log in to webmail, but mail is still arriving
- Likely cause: AD/LDAPS authentication is failing, not Zimbra. Mail delivery does not need auth,
  so delivery keeps working while logins fail — that split is the diagnostic.
- Check:  `openssl s_client -connect auth2.example.com:636` &nbsp;# expired or untrusted cert shows here
- Check:  `zmprov gd example.com | grep -i ldap` &nbsp;# confirms which AD hosts Zimbra is pointed at
- Fix:    if the LDAPS cert expired, follow `Security & Hardening/LDAPs certificate replacement procedure.txt`.
  If `auth2` is down, confirm `auth4.example.com` is listed as a fallback in `zimbraAuthLdapURL`.

### Symptom: one user cannot log in, everyone else is fine
- Check:  `zmprov ga user@example.com zimbraAccountStatus` &nbsp;# expect "active"
- Check:  the account is enabled and not locked out in AD
- Check:  `fail2ban-client status <jail>` &nbsp;# their IP may be banned
- Fix:    unlock in AD, or unban the IP as above.

### Symptom: outbound mail to one domain is deferring
- Likely cause: remote server refusing, greylisting, or the organization's IP on an RBL.
- Check:  `postqueue -p | grep example.com` &nbsp;# the deferral reason text is the answer
- Check:  the sending IP against a public blocklist lookup
- Fix:    if greylisting, wait — it retries. If RBL-listed, follow the RBL section above. If the
  remote rejects on authentication, check SPF/DKIM/DMARC alignment in
  `email-authentication-spf-dkim-dmarc.md`.

### Symptom: queue growing fast, thousands of messages to unknown recipients
- Likely cause: compromised account being used to relay spam. Treat as an incident.
- Check:  `postqueue -p | grep -oP 'from=<\K[^>]+' | sort | uniq -c | sort -rn | head` &nbsp;# which sender dominates the queue
- Fix:    `zmprov ma <account> zimbraAccountStatus locked` &nbsp;# stop the source first
- Then:   reset the AD password, revoke sessions, purge the spam from the queue, check RBL status,
  and follow `Security Procedures/IR_Security_Scenarios_Guide.md`.

### Symptom: users report mail is slow, webmail spinning
- Check:  `zmcontrol status` &nbsp;# a stopped or hung service shows here
- Check:  disk space — a full `/opt/zimbra` partition stops delivery silently:

      df -h /opt/zimbra                    # watch for >90% used
      du -sh /opt/zimbra/store /opt/zimbra/index /opt/zimbra/log | sort -h   # what is consuming it

- Fix:    clear old logs/backups, or extend the volume. Restart only the affected service.

### Symptom: a delegate can see the shared Inbox but not Sent Items
- Likely cause: the grant was applied to `/Inbox` only.
- Fix:    `zmmailbox -z -m shared@example.com modifyFolderGrant /Sent account user@example.com rwidx`

### Symptom: inbound mail rejected with "relay access denied"
- Likely cause: the recipient domain is not configured as a local domain, or the account does not exist.
- Check:  `zmprov gad` &nbsp;# lists domains Zimbra accepts mail for
- Check:  `zmprov ga recipient@example.com` &nbsp;# does the account exist at all

### Symptom: TLS warnings in every desktop client, all at once
- Likely cause: the `mail.example.com` certificate expired.
- Check:  `zmcertmgr viewdeployedcrt` &nbsp;# shows deployed cert and expiry
- Fix:    renewal procedure above. Remember `zmcontrol restart` after deploying, or half the
  services keep serving the old cert.

---

## Mail-Outage Triage Runbook

Work the list in order. Each step is cheap and rules out a whole class of cause. Do not skip ahead
to restarting things.

1. **Scope it.** One user, one department, or everyone? Sending, receiving, or both? Webmail,
   desktop client, or mobile? The answer to "can you get in at https://mail.example.com in a browser"
   separates a client problem from a server problem in ten seconds.
2. **Is the host up?** Ping and SSH to `mail.example.com`. If the host is down this becomes a VM/host
   recovery, not a mail problem.
3. **Are the services up?** `zmcontrol status`. Anything not "Running" is the lead.
4. **Is the disk full?** `df -h /opt/zimbra`. A full partition presents as a dozen unrelated
   symptoms and is the most common non-obvious cause.
5. **Is the queue moving?** `zmqstat` then `postqueue -p`. A large deferred queue means outbound
   trouble; an empty queue with complaints about missing mail means inbound trouble.
6. **Is it authentication?** If mail flows but logins fail, go to AD/LDAPS — cert expiry first
   (`openssl s_client -connect auth2.example.com:636`), then DC availability.
7. **Is it the certificate?** `zmcertmgr viewdeployedcrt`. Expired cert = clients fail, server looks
   healthy.
8. **Is it DNS?** Confirm MX, A and SPF records still resolve publicly. A registrar or DNS change is
   a silent, total outage for inbound mail.
9. **Is it the edge?** pfSense NAT rules and Snort blocks. A Snort rule that started blocking a
   sending partner looks exactly like that partner's mail server being down.
10. **Is it upstream?** If everything local is healthy, check whether the remote sender or the
    internet link is the problem before touching Zimbra.
11. **Only now consider restarting.** Restart the single failing service
    (`zmmailboxdctl` / `zmmtactl` / `zmproxyctl`), not the whole stack.
12. **Escalate.** If unresolved after the above: [FILL IN: vendor support arrangement for Zimbra — contract, portal, and case-opening details], and notify [FILL IN: who to tell at Example Org during a mail outage, and how, given that email is down].

Communication during an outage cannot go by email. Use [FILL IN: the agreed out-of-band channel — phone tree, Jabber, or other].

---

## Security

- **Exposure**: internet-facing. Ports open at the edge: 25 (SMTP in), 465/587 (submission),
  993 (IMAPS), 995 (POP3S), 443 (webmail), 7071 (admin console — should be internal/VPN only).
  [FILL IN: confirm which of these are actually exposed publicly in pfSense, especially 7071].
- **Auth**: staff authenticate against AD over LDAPS. Zimbra's internal admin account is separate:
  [FILL IN: the Zimbra admin account name and where its password is stored].
- **Certificates**: `mail.example.com` commercial cert, renewal procedure above. Expiry: [FILL IN: certificate expiry date].
- **Secrets**: the AD bind password for `servicesadmin` lives in the Zimbra LDAP config and in
  [FILL IN: password manager location]. Never in this doc.
- **Hardening**: fail2ban on auth failures; TLS required for submission;
  [FILL IN: whether 2FA is enabled for the Zimbra admin console]. Verify legacy protocols and
  weak ciphers are disabled: [FILL IN: result of a TLS scan against mail.example.com].
- Admin console access on 7071 should not be reachable from the internet. If it is, restrict it.

## Monitoring & Alerting

- Logs forwarded to `graylog01.example.com` — [FILL IN: stream name, and whether any alert conditions are defined on it].
- Queue depth alerting: [FILL IN: whether queue size is monitored, and the alert threshold].
- Disk space alerting on `/opt/zimbra`: [FILL IN: monitor and threshold].
- Certificate expiry alerting: [FILL IN: whether cert expiry is monitored or calendar-based].
- Uptime check on `https://mail.example.com`: [FILL IN: monitoring tool and check interval].
- Normal baseline: [FILL IN: typical daily message volume and steady-state queue depth, once measured].

## Disaster Recovery

- **RTO/RPO**: [FILL IN: agreed recovery time and acceptable data loss for mail].
- **Full rebuild**: provision a host of the same OS and Zimbra version, restore the LDAP dump and
  config, restore the mailstore, re-deploy the TLS cert, then re-point DNS/NAT. Sequence matters —
  restore LDAP before the store, or accounts will not exist to own the mail.
- Detailed rebuild steps: [FILL IN: link to a tested rebuild procedure, or write one].
- **If the mail server will be down for hours**, inbound mail from well-behaved senders queues on
  their side for [FILL IN: hours] before bouncing. A secondary MX would extend that:
  [FILL IN: whether a backup MX exists].
- **Escalation**: primary IT lead; secondary [FILL IN: secondary contact and after-hours escalation].

## Decisions & History (ADR-lite)

| Date       | Decision / Change                                   | Why / Ticket |
|------------|-----------------------------------------------------|--------------|
| 2026-09-11 | Guide created                                        | Documentation consolidation |
| [FILL IN: date]  | [FILL IN: when external AD auth was configured]       | [FILL IN: why / ticket]    |
| [FILL IN: date]  | [FILL IN: last Zimbra version upgrade]                | [FILL IN: why / ticket]    |
| [FILL IN: date]  | [FILL IN: last TLS certificate renewal]               | [FILL IN: why / ticket]    |

## References

- `Share Mailboxes Zimbra/` — GUI screenshots for the sharing dialog (2026-06-05)
- `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11) — LDAP bind parameters
- `Security & Hardening/LDAPs certificate replacement procedure.txt` — AD-side LDAPS cert renewal
- `[internal KB: fail2ban unban procedure]`
- `[internal KB: unban a user in Graylog for the mail host]`
- `[internal KB: unblocking an IP from Snort]`
- `Mail & Messaging/email-authentication-spf-dkim-dmarc.md` — SPF/DKIM/DMARC for example.com
- `Microsoft 365/m365-entra-admin-guide.md` — Exchange Online side, if any mail lives there
- `SysAdmin Procedures/Backup_DR_Runbook.txt`
- `Security Procedures/IR_Security_Scenarios_Guide.md` — compromised-account handling
- Upstream: Zimbra administrator guide and `zmprov`/`zmmailbox` command reference

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

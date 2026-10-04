# LDAP Connection Reference

> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

## Overview

This is the reference for connecting an application to the organization's Active Directory over
LDAP. It records the connection parameters, explains what each one is for, gives a
tested command line for verifying a bind, and covers the two failure modes that
actually occur here — certificate trust problems and LDAP slowdowns.

The organization's directory is Active Directory on domain `example.com`. Applications that need to
authenticate staff against AD bind to a domain controller over LDAPS, search for
the user, and then attempt a bind as that user to verify the password. Everything
in this document exists to make that sequence work.

**The bind account's DN is recorded here. Its password is not, and must never be
stored anywhere in this library.** See "Service account" below.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Environment      | prod                                                                   |
| Location         | `ldaps://auth2.example.com` port 636. Second DC: auth4.example.com — [FILL IN: confirm auth4 serves LDAPS and whether clients should be configured with both] |
| Access           | LDAPS (TCP 636) from application hosts. Bind as the `servicesadmin` service account; password in 1Password / IT vault |
| Dependencies     | Domain controllers auth2/auth4, DNS resolution of `example.com`, a valid LDAPS certificate in the DC's NTDS store, correct time (Kerberos/AD is time-sensitive) |
| Dependents       | [FILL IN: every application that authenticates users via LDAP — see the consumers table below] |
| Last reviewed    | 2026-09-11                                                             |

## Connection Parameters

These are the values to enter into an application's LDAP configuration screen.

| Parameter | Value | What it is for |
|---|---|---|
| URI / host | `ldaps://auth2.example.com` | The domain controller to connect to, over TLS. The `ldaps://` scheme means TLS is negotiated before anything else — nothing, including the bind password, crosses the wire in clear |
| Port | `636` | LDAPS. Plain LDAP is 389 and must not be used for authentication traffic |
| Base DN | `CN=Users,DC=example,DC=com` | Where the directory search starts. Only accounts under the `Users` container are found — an account in another OU will not be located from this base |
| Login attribute | `UserPrincipalName` | The attribute the user types as their username. UPN is `user@example.com` form, so users sign in with their email-style name rather than a short `DOMAIN\user` name |
| Bind DN | `CN=servicesadmin,CN=Users,DC=example,DC=com` | The service account the application binds as in order to *search* the directory. AD does not permit anonymous search, so a bind account is mandatory |
| Bind password | **Not recorded here.** 1Password / IT vault | — |
| Secondary host | [FILL IN: `ldaps://auth4.example.com` if it serves LDAPS] | Failover, so a single DC reboot does not take authentication down |
| Search filter | [FILL IN: the filter each application uses, e.g. `(&(objectClass=user)(userPrincipalName=%s))`] | Narrows the search to real user accounts |
| Group base DN | [FILL IN: if applications map AD groups to roles, the DN the group search starts from] | Role/permission mapping |
| Referral handling | [CONFIRM: proposed default — disable referral chasing.] | AD returns referrals that many LDAP libraries follow to a non-TLS port and then fail confusingly |

### Why a Base DN of `CN=Users`

`CN=Users,DC=example,DC=com` is AD's default container, not a custom OU. A search based
there finds only objects in that container and below it. If user accounts are also
kept in other OUs, those users cannot authenticate through an application
configured with this base and the symptom looks like "wrong password" rather than
"user not found". If that happens, the fix is usually to raise the base to
`DC=example,DC=com` and rely on the search filter, not to move the account.

[FILL IN: confirm whether all staff accounts genuinely live under `CN=Users`, or
whether some are in departmental OUs.]

## Applications That Consume LDAP Auth

[FILL IN: this table is the single most useful thing to complete in this document.
It answers "what breaks if auth2 is down or the service account password is
rotated". Candidates to check, based on the hosts in the library: sign.example.com,
files.example.com, forums.example.com, graylog01.example.com, the WireGuard/VPN stack, RADIUS for
802.1x Wi-Fi, and any Linux hosts joined with `adcli`/`sssd`.]

| Application | Host | Binds as | Base DN used | Notes |
|---|---|---|---|---|
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |

Note that RADIUS/802.1x Wi-Fi authentication is documented separately and uses its
own certificate — see `Security & Hardening/802.1x and Active Directory –
Systems Knowledge Base.pdf` and `Security & Hardening/Change Radius for WIfi.txt`.
Do not assume it uses the parameters in this document.

## How It Works

An application authenticating a user performs three steps:

1. **Service bind.** The application connects to `ldaps://auth2.example.com:636` and
   binds as `CN=servicesadmin,CN=Users,DC=example,DC=com`. This is a privileged-enough
   identity to read the directory, nothing more.
2. **Search.** Using the base DN and a filter on `userPrincipalName`, it finds the
   DN of the user who is trying to log in.
3. **User bind.** It attempts a second bind as that user's DN with the password
   the user supplied. Success means the password is correct; failure means it is
   not. The application then usually reads group memberships to decide what the
   user may do.

    app -> TLS handshake (port 636, DC cert validated against the CA)
        -> bind as servicesadmin          # search identity
        -> search base CN=Users,DC=example,DC=com for userPrincipalName=<typed name>
        -> bind as the found user DN      # password verification
        -> read group memberships         # authorisation

Everything after the handshake is inside TLS. That is the entire reason for using
port 636 rather than 389 — on 389 the user's password crosses the network in the
clear on step 3.

## Testing a Bind from the Command Line

Install the client tools on a Linux host:

    sudo apt install -y ldap-utils                    # provides ldapsearch, ldapwhoami

Test the service account bind and search, using the real values:

    ldapsearch -x -H ldaps://auth2.example.com:636 \
      -D "CN=servicesadmin,CN=Users,DC=example,DC=com" -W \
      -b "CN=Users,DC=example,DC=com" \
      "(userPrincipalName=user@example.com)" dn userPrincipalName

    # -x  simple authentication (not SASL/GSSAPI)
    # -H  the LDAPS URI — TLS from the first byte
    # -D  the bind DN: who you are binding as
    # -W  prompt for the password (never -w on the command line; it lands in shell history)
    # -b  search base
    # the filter and then the attributes to return

Expected: `result: 0 Success` and one entry returned. Common outcomes:

- `Invalid credentials (49)` — wrong bind password, or a locked/expired service account.
- `No such object (32)` — the base DN is wrong.
- `result: 0 Success` with zero entries — the bind worked, the filter matched nobody.
  This is the "user is in a different OU" case.
- `Can't contact LDAP server (-1)` — network, port, or TLS trust. See troubleshooting.

Confirm which identity the server thinks you are:

    ldapwhoami -x -H ldaps://auth2.example.com:636 \
      -D "CN=servicesadmin,CN=Users,DC=example,DC=com" -W        # expect: u:CORP\servicesadmin

Test a specific user's password (the step 3 bind), which is how you prove an
authentication problem is the password and not the configuration:

    ldapsearch -x -H ldaps://auth2.example.com:636 \
      -D "<user@example.com>" -W -b "CN=Users,DC=example,DC=com" -s base "(objectClass=*)" dn

AD accepts a UPN in the `-D` position, which is convenient — you do not need the
user's full DN to test their password.

Inspect the TLS layer on its own, separately from LDAP:

    openssl s_client -connect auth2.example.com:636 -showcerts </dev/null    # full chain, handshake result

Repeat every test against auth4.example.com before declaring a problem to be
DC-specific.

## Certificate Requirements for LDAPS

LDAPS does not work simply because port 636 is configured. The domain controller
must hold a certificate that meets all of the following, and the client must trust
its issuer.

On the domain controller, the certificate must:

- Be installed in the **NTDS service store** (`Certificates (Service – AD DS)` ->
  `NTDS` -> `Personal`), not only the Local Computer personal store. This is the
  step most guides omit and the most common reason LDAPS silently does not listen.
- Include the **Server Authentication** EKU (`1.3.6.1.5.5.7.3.1`).
- Have the DC's FQDN in the Subject or, preferably, the SAN. Modern clients
  validate SAN.
- Have its **private key present**, with `NETWORK SERVICE` granted read access to
  the key file.
- Be the **only** valid Server Authentication certificate in play. If several
  exist, AD picks one, and it may not be the one you installed.

On the client, the issuing root (and any intermediate) CA must be trusted:

    # Linux: install the CA into the system trust store
    sudo cp <ca>.crt /usr/local/share/ca-certificates/<ca>.crt
    sudo update-ca-certificates                       # then ldapsearch will validate the chain

Verify the DC is actually listening:

    # on the DC, from cmd:
    netstat -an | find ":636"                         # expect a LISTENING line

If 636 is not listening, no client-side change will help — the certificate was not
selected. Full replacement procedure, including the NTDS store import and the
private key permissions step:
**`Security & Hardening/LDAPs certificate replacement procedure.txt`**.

Certificate expiry here is not currently monitored. An expired LDAPS certificate
takes down every application in the consumers table at once, at a moment that was
predictable weeks earlier. See the certificate expiry section of
`SysAdmin Procedures/monitoring-alerting-guide.md`:

    echo | openssl s_client -connect auth2.example.com:636 2>/dev/null | openssl x509 -noout -enddate

## Service Account

`CN=servicesadmin,CN=Users,DC=example,DC=com` is the shared bind identity used to search
the directory.

**The password must never be written into this library** — not into this file, not
into a troubleshooting note, not into a pasted config snippet. It lives in
1Password / IT vault. This is not a formality: this documentation library already
contains several files with plaintext credentials that have been sitting in a
shared folder for years, and every one of those is now a rotation task. Do not add
to that list. The DN is recorded here because the DN is configuration; the password
is a secret.

Requirements for the account:

- It needs **read access to the directory only.** It does not need Domain Admin.
  [FILL IN: confirm the account's actual group memberships and reduce them if it
  holds more privilege than a directory read requires.]
- It should be set so the password **does not expire**, because an expiring
  password on a service account means every consuming application breaks
  simultaneously and without warning. [FILL IN: confirm the current setting.]
- Because it does not expire automatically, it must be **rotated deliberately**.

### Rotation requirement

[CONFIRM: proposed default — rotate annually, and immediately on any staff
departure where the credential may have been known, or any suspected exposure.]

The rotation is only safe if the consumers table above is complete, because the
password must be changed in every consuming application in the same maintenance
window. Rotating first and discovering consumers afterwards is an outage.

Procedure:

1. Complete and verify the consumers table. This is the whole job.
2. Schedule a window and notify affected users.
3. Generate a new password in 1Password.
4. Change the password in AD.
5. Update every consumer, then restart or re-test each one.
6. Verify with `ldapwhoami` using the new password.
7. Record the date below and in `SysAdmin Procedures/Change_Management_Log.txt`.

Last rotated: [FILL IN: date]. Next due: [FILL IN: date].

Related: `SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt`,
`Active Directory/AD-Admin-Security-Guide.md`.

## Troubleshooting

### Symptom: bind fails with "Invalid credentials (49)"

- Likely cause: wrong password, wrong bind DN format, or the account is locked,
  disabled, or expired.
- Check the DN is exactly right — `CN=servicesadmin,CN=Users,DC=example,DC=com`. A DN
  is not a username; `servicesadmin` alone will not work, though AD will also
  accept the UPN form `servicesadmin@example.com` in the `-D` position, which is a
  useful way to isolate a DN typo from a password problem.
- Check the account state in AD (`Get-ADUser servicesadmin -Properties LockedOut,Enabled,PasswordExpired`).
- Check for a `data 52e` / `data 775` code in the extended error text: `52e` is a
  bad password, `775` is a locked account, `533` is disabled, `532` is an expired
  password. AD returns the same error 49 for all of them, and the sub-code is the
  only thing that distinguishes them.
- Fix: correct the credential from 1Password, or unlock the account. If the
  password was rotated without updating consumers, this is the symptom you will
  see across every application at once.

### Symptom: "Can't contact LDAP server" or TLS handshake failure

- Likely cause: the client does not trust the DC's issuing CA, the DC is not
  listening on 636, or a firewall is blocking it. These look identical from the
  application's error message.
- Separate the layers, in this order:

    nc -vz auth2.example.com 636                           # is anything listening and reachable at all
    openssl s_client -connect auth2.example.com:636 -showcerts </dev/null   # does TLS complete, is the chain full
    ldapsearch -x -H ldaps://auth2.example.com:636 ...     # only then test LDAP itself

- A handshake that fails with "unable to get local issuer certificate" is a client
  trust problem: install the root and any intermediate CA into the client's trust
  store.
- A chain that shows the server certificate but not the intermediate means the DC
  is not sending the full chain; import the intermediate on the DC.
- Nothing listening on 636 means the certificate was never selected by AD. That is
  a server-side fix: `Security & Hardening/LDAPs certificate replacement procedure.txt`.
- Works from one host but not another: that is a trust store or firewall
  difference between the two hosts, not a DC problem.
- Note from the replacement procedure: LDAP error 54 usually means the connection
  was closed because no bind followed, not that the certificate is wrong.

### Symptom: authentication is slow, or connections drop intermittently

- Likely cause: DC resource pressure, LDAP policy limits, network instability, or
  an application opening far more connections than it closes.
- This has happened in this environment and has its own document. Go to
  **`Security & Hardening/LDAP Slow Down troubleshooting.txt`** — it covers
  enabling LDAP Interface Events diagnostic logging, the Event IDs to look for
  (2889 unsigned bind, 1215 client closed connection, 1535 LDAP error), the
  `NTDS` performance counters (LDAP Bind Time, LDAP Client Sessions, Request
  Latency), and adjusting LDAP policy with `ntdsutil`.
- Quick first checks:

    # on the DC:
    netstat -an | find ":636"                         # how many connections, in what state
    dcdiag /v > dcdiag_output.txt                     # DC health, replication, DNS
    w32tm /query /status                              # time skew breaks Kerberos and confuses everything

- Many connections in `CLOSE_WAIT` point at an application that is not closing
  connections. That is a client bug, and the fix is in the application, not the DC.
- Check `lsass.exe` CPU and memory on the DC — LDAP runs inside it.

### Symptom: bind succeeds but the user cannot log in to the application

- Likely cause: the search found nothing, so the application never got to the user
  bind. Almost always the base DN or the filter.
- Check: run the `ldapsearch` above with that user's UPN. Zero entries with
  `result: 0 Success` confirms it.
- Fix: widen the base DN to `DC=example,DC=com` and rely on the filter, or correct the
  filter. Confirm the attribute being matched is `userPrincipalName` and that the
  user is entering the full `user@example.com` form, not a short name.

### Symptom: Event ID 2889 appearing on the DC

- Meaning: a client is performing an unsigned, unencrypted simple bind — a
  password crossing the network in the clear on port 389.
- The event includes the client IP. Use it to find the misconfigured application
  and move it to LDAPS on 636.
- This matters beyond hygiene: Windows Server can be configured to require LDAP
  signing and channel binding, and when that is enforced these clients stop
  working entirely. Find them before the enforcement, not after.

## Security

- Exposure: TCP 636 from application hosts to the domain controllers. Port 389
  should not be used for authentication. [FILL IN: whether 389 is still open, and
  from where.]
- Auth: simple bind over TLS with the `servicesadmin` service account.
- Certificates: the DC's LDAPS certificate must be in the NTDS store with a
  Server Authentication EKU and matching FQDN. Expiry is unmonitored — see
  `SysAdmin Procedures/monitoring-alerting-guide.md`.
- Secrets: the bind password is in 1Password only. **Never in this library.**
- Least privilege: the bind account needs directory read, nothing more.

## References

- `_ARCHIVE/superseded-stubs/Ldap.txt` (archived 2026-09-11) — the original connection values this document expands on
- `Active Directory/AD-Admin-Security-Guide.md`
- `Security & Hardening/LDAPs certificate replacement procedure.txt` — LDAPS certificate installation and replacement
- `Security & Hardening/LDAP Slow Down troubleshooting.txt` — slow or dropping LDAP connections
- `Security & Hardening/802.1x and Active Directory – Systems Knowledge Base.pdf`
- `SysAdmin Procedures/Daily_SysAdmin_Procedures.txt` — AD tooling reference
- `SysAdmin Procedures/monitoring-alerting-guide.md` — certificate expiry monitoring
- Microsoft: view and set LDAP policy with ntdsutil — https://learn.microsoft.com/en-us/troubleshoot/windows-server/active-directory/view-set-ldap-policy-using-ntdsutil

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

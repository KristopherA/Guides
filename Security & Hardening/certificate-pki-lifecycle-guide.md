> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Certificate & PKI Lifecycle

## Overview

Example Org runs TLS on a dozen or so services, and each one has historically been
renewed by whoever noticed it was about to expire. The per-service how-to
documents already exist and are good — LDAPS on the domain controllers, MySQL
TLS rotation, Let's Encrypt on the Apache hosts, the Zimbra mail host, the
RADIUS server certificate. What has never existed is the layer above them: a
single list of every certificate we hold, a rule for which CA issues what, a
lead time for acting before expiry, and above all a mechanism that tells us a
certificate is expiring *before* a user does.

This document is that layer. It does not restate the per-service procedures —
it points at them. Its own content is the inventory, the policy, the calendar,
the monitoring, and the two procedures that no single-service document covers:
a generic renewal with a real verification step, and emergency replacement of a
key you believe is compromised. Audience is IT Ops; the owner of any individual
certificate is named in the inventory table.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Owner            | IT lead (it@example.com) — process owner `[CONFIRM: proposed default]` |
| Environment      | prod (all Example Org production TLS endpoints), plus lab/test certificates where issued |
| Location         | Certificates live on each service host; issuance records and private-key backups in `[FILL IN: password manager / vault in use]` |
| Access           | Per-service admin access (SSH, RDP to DCs, Barracuda WAF console, pfSense web UI); CA issuance requires `[FILL IN: who can request/approve issuance from the internal CA]` |
| Dependencies     | Internal CA `[FILL IN: internal CA name and host]`; public CA `[FILL IN: public CA in use for wildcard.example.com]`; Let's Encrypt (ACME, outbound 443); DNS (example.com) for ACME and CAA; NTP (a skewed clock breaks validation) |
| Dependents       | LDAPS auth for every LDAP-integrated app; 802.1x wireless and wired auth; MySQL client connections; all published web apps behind the Barracuda WAF; Zimbra mail clients; Graylog and Nessus web UIs |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                               |

## How It Works

Example Org has two issuers and they are not interchangeable. Anything a member, a
personal device, or an external party has to trust must come from a **public
CA**, because we cannot install our root on devices we do not manage.
Everything that is only ever trusted by domain-joined machines or by servers we
configure ourselves can come from the **internal CA**, which is free, fast to
issue, and lets us use long lifetimes and non-standard names.

The practical split looks like this:

```
Public CA   -> wildcard *.example.com, Zimbra mail host, anything published
               through the Barracuda WAF, Let's Encrypt on Apache hosts
Internal CA -> LDAPS on the domain controllers, MySQL server/client certs,
               RADIUS server cert for 802.1x, management UIs (pfSense,
               Graylog, Nessus) [CONFIRM: proposed default]
```

Three different renewal mechanisms are in play, and knowing which one a
certificate uses is the first thing to establish before touching it:

```
ACME / certbot   # fully automatic, Apache hosts; failure is silent
Manual reissue   # CSR -> CA -> install -> reload; wildcard, Zimbra, WAF
AD auto-enrol    # internal CA pushes to domain members via GPO [FILL IN: is auto-enrolment configured?]
```

Private keys are generated on the host that will use them wherever possible, so
the key never travels. The exception is the wildcard, which by definition is
installed in several places; its key and PFX live in
`[FILL IN: password manager / vault in use]` and every copy on a host must be
tracked in the inventory below, because a compromised wildcard key means
replacing it everywhere at once.

Logs worth knowing about: certbot writes to `/var/log/letsencrypt/`, Apache TLS
errors land in the vhost error log, MySQL TLS failures land in
`/var/log/mysql/error.log`, and LDAPS selection failures on a DC show up in the
Windows System and Directory Service event logs rather than anywhere obvious.

## Certificate Inventory

This is the master list. Every TLS endpoint Example Org operates gets a row. The rows
below are pre-seeded from known services so nothing gets forgotten; every value
still has to be filled in from the live systems.

| Service | Host | CN / SAN | Issuer | Key type | Expiry | Renewal method | Owner |
|---------|------|----------|--------|----------|--------|----------------|-------|
| LDAPS (domain controller) | auth2.example.com:636 | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| LDAPS (domain controller) | auth4.example.com:636 | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| MySQL server TLS | `[FILL IN: MySQL host]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| MySQL client certs (if used) | `[FILL IN: which clients]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| Wildcard | multiple — see note below | `*.example.com` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| Apache web hosts (Let's Encrypt) | `[FILL IN: list each Apache vhost/FQDN]` | `[FILL IN]` | Let's Encrypt | `[FILL IN]` | auto (90d) | certbot / ACME | `[FILL IN]` |
| Zimbra mail host | `[FILL IN: Zimbra host FQDN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| RADIUS server (802.1x) | `[FILL IN: RADIUS/NPS host]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| Barracuda WAF — published services | `[FILL IN: WAF management FQDN]` | `[FILL IN: each published hostname]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| pfSense web UI / VPN | `[FILL IN: pfSense host]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| Graylog web UI | graylog01.example.com | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| Nessus scanner UI | `[FILL IN: Nessus host]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| Internal CA — root certificate | `[FILL IN: internal CA host]` | `[FILL IN]` | self | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| Internal CA — issuing CA certificate | `[FILL IN: internal CA host]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` | `[FILL IN]` |
| `[FILL IN: any remaining TLS endpoint not listed above]` | | | | | | | |

Note on the wildcard: record every host it is installed on, not just one row.
`[FILL IN: complete list of hosts holding a copy of the *.example.com certificate and key]`

Note on the CA certificates: the root and issuing CA certificates expire too,
and their expiry invalidates every certificate beneath them. They are the two
rows in this table whose expiry must never be discovered late.

Where the inventory overlaps with Section 6 of
`SysAdmin Procedures/Inventory_Asset_Reference.txt`, this table is the
authoritative one; that section should be reduced to a pointer here.
`[CONFIRM: proposed default]`

## Internal vs Public CA — which to use

Use the **public CA** when any of the following is true:

- A device Example Org does not manage has to trust it (member laptops, phones,
  home machines, external partners).
- The name is reachable from the internet.
- It terminates on the Barracuda WAF for a published service.
- It is a mail host certificate — mail clients and peer MTAs will not have
  our root.

Use the **internal CA** when all of the following are true:

- Only domain-joined machines or servers we configure ourselves will validate
  it.
- The name is internal-only, or the service is internal-only.
- We want a lifetime longer than a public CA will issue.

Rules that apply to both:

- Key type: `[FILL IN: approved key type and size — e.g. RSA vs EC and the
  minimum accepted]`. Do not choose per-certificate; choose once, record it
  here, and follow it.
- Every certificate must carry the FQDN in the **SAN**, not only the CN.
  CN-only certificates are rejected by current clients, and this is the single
  most common cause of a "worked in testing" failure.
- Server certificates must carry the Server Authentication EKU. LDAPS in
  particular will silently not listen without it — see the LDAPS procedure.
- Publicly trusted certificates must not be issued for internal-only names that
  leak our internal topology into Certificate Transparency logs. If you need a
  name kept out of CT, it goes to the internal CA.
- CAA records on example.com constrain which public CAs may issue.
  `[FILL IN: current CAA record contents for example.com]`

## Renewal Calendar and Lead Times

Act on lead time, not on the expiry date. The lead time exists to absorb a
failed validation, a CA that takes three days to issue, a change window that is
a week out, and the discovery that the service needs a restart nobody scheduled.

| Certificate class | Lead time — start renewal | Escalate if not done by |
|-------------------|---------------------------|-------------------------|
| ACME / Let's Encrypt (auto) | n/a — renews at 30 days remaining | 14 days remaining `[CONFIRM: proposed default]` |
| Internal CA, single service | 30 days `[CONFIRM: proposed default]` | 14 days `[CONFIRM: proposed default]` |
| Public CA, single service | 45 days `[CONFIRM: proposed default]` | 21 days `[CONFIRM: proposed default]` |
| Wildcard *.example.com (multi-host) | 60 days `[CONFIRM: proposed default]` | 30 days `[CONFIRM: proposed default]` |
| LDAPS on domain controllers | 45 days `[CONFIRM: proposed default]` | 21 days `[CONFIRM: proposed default]` |
| RADIUS / 802.1x server cert | 60 days `[CONFIRM: proposed default]` | 30 days `[CONFIRM: proposed default]` |
| Internal CA issuing certificate | 180 days `[CONFIRM: proposed default]` | 90 days `[CONFIRM: proposed default]` |
| Internal CA root certificate | 365 days `[CONFIRM: proposed default]` | 180 days `[CONFIRM: proposed default]` |

The RADIUS and wildcard lead times are deliberately the longest because both
have a blast radius larger than one service: a bad RADIUS certificate takes
wireless down for everyone at once, and the wildcard has to be replaced on every
host that holds a copy in the same window.

Standing calendar entries to create:

- Monthly, first business day: run the expiry sweep (Operations below) and
  reconcile it against the inventory table.
- Quarterly: review the inventory for rows that no longer exist and endpoints
  that were added without a row.
- Annually: review this document, the CA split, and the CAA record.

`[CONFIRM: proposed default]` for all three cadences.

## Operations (Day-2)

### Generic renewal procedure

This is the shape every manual renewal takes. Where a per-service procedure
exists, follow that for the install step and use the verification here.

**Step 1 — Establish what you are replacing.** Do not skip this; half of all
certificate incidents are caused by replacing the wrong file.

```
openssl s_client -connect <host>:443 </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName   # what is actually being served right now
openssl x509 -in /path/to/current.crt -noout -subject -issuer -dates -ext subjectAltName                                        # what is in the file you think is being served
```

If those two disagree, find out why before continuing.

**Step 2 — Generate a new key and CSR on the host that will use it.**

```
umask 077                                                                          # new key must not be world-readable, set this first
openssl req -new -newkey rsa:3072 -nodes -keyout new.key -out new.csr -subj "/CN=<fqdn>" -addext "subjectAltName=DNS:<fqdn>"   # adjust key type to the standard recorded above
openssl req -in new.csr -noout -text | grep -A1 "Subject Alternative Name"         # confirm the SAN is actually present before you send it
```

`[FILL IN: approved key type and size]` — if the standard is EC rather than RSA,
substitute the matching `openssl ecparam`/`req` invocation and record it here.

**Step 3 — Submit the CSR.**
`[FILL IN: internal CA submission method — web enrolment URL, certreq, or CLI]`
`[FILL IN: public CA submission method and who approves the order]`

**Step 4 — Verify the issued certificate before installing it.** Four checks,
all cheap, and each one catches a failure mode that is expensive later.

```
openssl x509 -in new.crt -noout -subject -issuer -dates -ext subjectAltName,extendedKeyUsage   # right name, right SAN, right EKU, sane dates
openssl x509 -noout -modulus -in new.crt | openssl md5                                          # key/cert pairing, part 1
openssl rsa  -noout -modulus -in new.key | openssl md5                                          # part 2 — these two hashes MUST match
openssl verify -CAfile chain.pem new.crt                                                        # chain builds to a root we trust
```

**Step 5 — Install.** Follow the per-service pointer below. Keep the old
certificate and key in place, renamed, until step 6 passes.

**Step 6 — Verify from a client, not from the server.** This is the step that
most often gets skipped and it is the only one that proves anything.

```
openssl s_client -connect <host>:<port> -servername <fqdn> </dev/null 2>/dev/null | openssl x509 -noout -dates -subject   # new dates served?
openssl s_client -connect <host>:<port> -servername <fqdn> -showcerts </dev/null | grep -c "BEGIN CERTIFICATE"            # full chain present, not just the leaf?
```

Then exercise a real client: log in to the application, bind over LDAPS,
connect a MySQL client, join wireless with a test device. A handshake succeeding
is not the same as the service working.

**Step 7 — Record it.** Update the inventory row (new expiry, new issuer if
changed), and only then remove the old key material.

### Per-service renewal pointers

Do not renew any of these from first principles — each has a documented
procedure with the site-specific traps already written down.

| Service | Procedure | The trap it documents |
|---------|-----------|-----------------------|
| LDAPS on domain controllers (auth2/auth4.example.com:636) | `Security & Hardening/LDAPs certificate replacement procedure.txt` | Certificate must go in the **NTDS** service store, not Local Computer\Personal, and only one valid Server Authentication cert may exist or AD picks the wrong one |
| MySQL server TLS | `Databases/mysql certificate rotation.txt` | Zero-downtime rotation via `ALTER INSTANCE RELOAD TLS`; file permissions must be right or the reload fails silently; rotating the CA requires distributing the new CA to clients first |
| MySQL TLS setup / key generation reference | `_ARCHIVE/superseded-2018-mysql/SSL and MYSQL.txt` (archived 2026-09-11) | Key and CSR generation reference |
| Apache hosts (Let's Encrypt) | `SysAdmin Procedures/Advanced_SysAdmin_Procedures.txt` (certbot section, ~line 300) and `SysAdmin Procedures/Server_Troubleshooting_Guide.txt` (renewal failures, ~line 670) | `certbot renew --dry-run` before trusting the timer; LE rate limits during repeated attempts |
| Zimbra mail host | `[FILL IN: path to the Zimbra mail host certificate procedure — not located in this library as of 2026-09-11]` | `[FILL IN]` |
| RADIUS / 802.1x | `Security & Hardening/Update Radius Certificate.jpg` and `Security & Hardening/Change Radius for WIfi.txt`; background in `[internal KB: 802.1x and Active Directory]` | Supplicants that pin the old server certificate will refuse to connect until the new one is trusted — plan the client side before the server side |
| Barracuda WAF published services | `Security & Hardening/barracuda_web_application_firewall_best_practices_guide.pdf` | `[FILL IN: how certificates are uploaded and bound to services on the WAF at the organization]` |
| Wildcard *.example.com | `[FILL IN: path to the wildcard renewal/distribution procedure]` | Must be replaced on every host holding a copy within one window |

### Expiry monitoring

This is the gap the per-service documents leave, and the reason this guide
exists. Automatic renewal is not monitoring — certbot fails silently, and a
failed renewal looks exactly like a successful one until the certificate
expires. Every certificate in the inventory needs an active check, including the
ones that renew themselves.

The approach: a scheduled script probes every endpoint and every local
certificate file, emits JSON, and ships it into Graylog through the existing
Sidecar/Filebeat pipeline already documented in
`Security & Hardening/fresh-box-hardening-cheatsheet.txt` (the audit shipping
section). Graylog then alerts on days-remaining thresholds. This reuses
infrastructure that already exists rather than adding another tool.

**Step 1 — the sweep.** Run from a host that can reach every endpoint.

```
for h in auth2.example.com:636 auth4.example.com:636 graylog01.example.com:443; do echo | openssl s_client -connect "$h" 2>/dev/null | openssl x509 -noout -enddate -subject; done   # remote endpoints — extend this list from the inventory table
find /etc/ssl /etc/letsencrypt/live -name '*.pem' -o -name '*.crt' 2>/dev/null | while read c; do echo "$c $(openssl x509 -enddate -noout -in "$c" 2>/dev/null)"; done   # local certificate files on this host
```

`[FILL IN: full endpoint list for the sweep, derived from the inventory table]`

**Step 2 — emit days-remaining as JSON** into the watched audit directory so the
existing Filebeat collector picks it up.

```
mkdir -p /var/log/org-audit                                                   # same directory the audit pipeline already tails
# per endpoint: compute (notAfter - now) in days and write one JSON object per line, e.g.
# {"tool":"certcheck","host":"auth2.example.com:636","subject":"...","days_remaining":27}
```

`[FILL IN: path where the cert-check script will live — suggest /org-scripts/ alongside audit-collect.sh]`

**Step 3 — alert in Graylog.** Create an event definition on the audit stream:

- `days_remaining < 30` → warning to `[FILL IN: alert destination — email address or channel]`
- `days_remaining < 14` → escalation, and open a ticket
- `days_remaining < 7` → treat as an incident in progress
- **no certcheck message received for a host in 48h** → alert. A monitor that
  stops reporting is indistinguishable from a monitor reporting "fine", and
  this is how expiry monitoring usually fails.

`[CONFIRM: proposed default]` for all four thresholds.

**Step 4 — a second, independent check.** The sweep runs from inside our
network and will not notice a chain that is broken only from outside. Add an
external check on the publicly reachable names.
`[FILL IN: external monitoring service in use, if any]`

Interim measure until the above is built: the certificate expiry loop in
Section 6 of `SysAdmin Procedures/Inventory_Asset_Reference.txt` can be run by
hand monthly. It is a manual check with no alerting — acceptable as a stopgap,
not as the answer.

### Emergency replacement of a compromised key

Triggers: a private key found in a repository, a backup, an email, a ticket or a
chat; a host compromise on any machine holding a key; a key handled by someone
who has left; a CA compromise notification. Treat suspicion as confirmation —
the cost of an unnecessary replacement is an afternoon, and the cost of leaving a
compromised key live is unbounded.

**Step 1 — Scope it before you touch anything.** Which certificate, which key,
and every host holding a copy. For the wildcard this is the whole list in the
inventory note. Record the serial number now:

```
openssl x509 -in <cert> -noout -serial -subject -issuer   # serial is what you give the CA for revocation
```

**Step 2 — Issue a replacement with a brand new key.** Never reuse the key, and
never reuse the CSR — a CSR carries the public key of the compromised pair.
Follow the generic renewal, steps 2–4.

**Step 3 — Install the replacement everywhere at once.** Prioritise
internet-facing and authentication endpoints. For the wildcard, all hosts in the
same maintenance window; a partial rollout leaves the compromised key live
somewhere.

**Step 4 — Revoke the old certificate.** Only after the replacement is confirmed
working, unless the exposure is active and severe, in which case revoke first and
accept the outage.

```
# internal CA:  [FILL IN: revocation command/console path for the internal CA]
# public CA:    [FILL IN: public CA revocation process and who is authorised to request it]
```

Revocation reason should be `keyCompromise`, not `superseded` — the reason code
is what tells relying parties this was not routine.

**Step 5 — Destroy every copy of the old key.** Hosts, backups, vault entries,
the workstation you generated it on, `~/.ssh`, Downloads folders.
`[FILL IN: secure destruction standard for key material]`

**Step 6 — Work out how it got out, and rotate anything else that travelled with
it.** A key exposed in a repository or a backup rarely travelled alone. Hand off
to `secrets-management-guide.md` (secret-exposure incident response) and, if a
host compromise is suspected, to `Security Procedures/IR_Security_Scenarios_Guide.md` scenario 4
(Exposed Credentials / Secrets in Code) and the endpoint isolation procedure in
`endpoint-security-operations-guide.md`.

**Step 7 — Record it** in Decisions & History below, and in the change log at
`SysAdmin Procedures/Change_Management_Log.txt`.

### Adding a new TLS endpoint

Any new service that terminates TLS gets an inventory row **before** it goes
live, not after. The row is what puts it into the monitoring sweep; a service
that is not in the sweep will expire unnoticed. This is a one-line addition and
it is the highest-value habit in this document.

## Troubleshooting

### Symptom: browser or client reports "unable to get local issuer certificate"

- Likely cause: the server is serving the leaf certificate only, without the
  intermediate. Works in a browser that has the intermediate cached, fails
  everywhere else — which is why it reaches production so often.
- Check: `openssl s_client -connect <host>:443 -servername <fqdn> -showcerts </dev/null | grep -c "BEGIN CERTIFICATE"` — expect 2 or more, 1 means leaf-only.
- Fix: install the full chain file (leaf + intermediates, leaf first, root
  omitted) in the service's certificate directive and reload.

### Symptom: certificate name mismatch

- Likely cause: the name being requested is in the CN but not in the SAN, or the
  client is connecting by IP or by an alias that was never in the SAN.
- Check: `openssl x509 -in <cert> -noout -ext subjectAltName` — the exact name the client uses must appear here.
- Fix: reissue with the correct SAN list. Adding the name to the CN does nothing.

### Symptom: LDAPS not listening on 636 after installing a new certificate

- Likely cause: certificate landed in Local Computer\Personal instead of the
  NTDS service store, or a second valid Server Authentication certificate is
  present and AD selected it.
- Check: `netstat -an | find ":636"` on the DC — expect `LISTENING`.
- Fix: follow `Security & Hardening/LDAPs certificate replacement procedure.txt`
  steps 3, 5 and 6 — NTDS store, remove conflicting certificates, grant
  NETWORK SERVICE read on the private key.

### Symptom: MySQL still presents the old certificate after replacing the files

- Likely cause: TLS was not reloaded, or file permissions are wrong and the
  reload failed without an obvious error.
- Check: `SHOW STATUS LIKE 'Ssl_server_not_after';` on a **new** connection — existing connections keep the old context.
- Fix: correct ownership to `mysql:mysql` and mode 600 on the key, then
  `ALTER INSTANCE RELOAD TLS;` — see `Databases/mysql certificate rotation.txt`.

### Symptom: certbot renewal has silently stopped

- Likely cause: the timer is not running, the HTTP-01 challenge path is blocked
  by a redirect or the WAF, or DNS changed.
- Check: `certbot renew --dry-run` — exercises the full renewal without consuming rate limit.
- Fix: depends on what the dry run says; repeated failures hit Let's Encrypt
  rate limits, so use `--staging` while debugging. See
  `SysAdmin Procedures/Server_Troubleshooting_Guide.txt`.

### Symptom: wireless clients stopped connecting after a RADIUS certificate change

- Likely cause: supplicants configured to validate a specific server
  certificate, or a client OS that does not trust the new issuer.
- Check: RADIUS authentication log for TLS handshake rejections;
  `[FILL IN: RADIUS log location]`
- Fix: push updated trust/profile to clients before rotating, or roll back to
  the previous certificate and re-plan. This is the reason the RADIUS lead time
  above is 60 days.

### Symptom: everything fails validation at once, on one host

- Likely cause: clock skew, or the CA certificate itself expired.
- Check: `timedatectl` (or `w32tm /query /status` on Windows), then check the expiry of the issuing CA certificate in the inventory.
- Fix: correct time sync; if the CA certificate has expired, that is a separate,
  larger piece of work — see the CA rows in the inventory table.

### Symptom: a certificate was renewed and the service still serves the old one

- Likely cause: the service was never reloaded, or there is more than one copy
  of the certificate file and you updated the one that is not referenced.
- Check: compare the served certificate against the file on disk, as in step 1 of the generic renewal.
- Fix: find the path the service config actually references, then reload.

## Security

- Exposure: certificates themselves are public; **private keys are the secret**
  and are governed by `secrets-management-guide.md`.
- Key handling: generate on the host that will use the key; never email, never
  paste into a ticket or chat, never commit. A key that has travelled through any
  of those is compromised and goes through emergency replacement above.
- Storage of exportable keys (wildcard, PFX bundles):
  `[FILL IN: password manager / vault in use]`, access limited to
  `[FILL IN: who holds access to key material]`.
- File permissions on hosts: key `0600` owned by the service account, certificate
  `0644`. Wrong permissions are a common silent-failure cause, not just a
  hardening nicety.
- Revocation authority: `[FILL IN: who is authorised to request revocation]`
- Internal CA protection: the issuing CA's own key is the highest-value key the organization
  holds. `[FILL IN: where the internal CA key lives and how it is protected]`

## Monitoring & Alerting

- Monitored: days-to-expiry for every inventory row, via the sweep above into
  Graylog (graylog01.example.com).
- Also monitored: absence of a check result per host (staleness), because a
  silent monitor is the failure mode that actually bites.
- Thresholds: 30 / 14 / 7 days, per Expiry monitoring above
  `[CONFIRM: proposed default]`.
- Alert destination: `[FILL IN: alert destination — email address or channel]`
- Baseline: every inventory row reports once per run; ACME certificates never
  drop below 30 days remaining in normal operation, so an ACME certificate at 25
  days means automation has stopped.

## Disaster Recovery

- RTO for a single expired service certificate: 4 hours from detection
  `[CONFIRM: proposed default]`. RTO for an expired RADIUS or wildcard
  certificate: 1 hour `[CONFIRM: proposed default]` — both are outages affecting
  everyone.
- If the internal CA host is lost: `[FILL IN: internal CA backup and restore
  procedure — is the CA database backed up, and where]`. Until this is answered,
  the loss of the CA host is an unbounded incident, which makes it the highest
  priority item in this document.
- Key escrow: the wildcard key and any PFX bundles must be recoverable from
  `[FILL IN: password manager / vault in use]` independently of any single host.
- Break-glass: if the vault is unavailable, see the break-glass section of
  `secrets-management-guide.md`.
- Escalation if the owner is unavailable: `[FILL IN: escalation contact]`

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why / Ticket |
|------------|-------------------|--------------|
| 2026-09-11 | Created this guide as the unifying layer over the existing per-service certificate procedures | Renewals were reactive; no inventory and no expiry alerting existed |
| `[FILL IN]` | `[FILL IN: record each renewal, reissue and revocation here]` | `[FILL IN]` |

## References

- `Security & Hardening/LDAPs certificate replacement procedure.txt` — LDAPS on auth2/auth4.example.com:636
- `Databases/mysql certificate rotation.txt` — zero-downtime MySQL TLS rotation
- `_ARCHIVE/superseded-2018-mysql/SSL and MYSQL.txt` (archived 2026-09-11) — MySQL TLS key/CSR generation
- `SysAdmin Procedures/Advanced_SysAdmin_Procedures.txt` — certbot / Let's Encrypt operations
- `SysAdmin Procedures/Server_Troubleshooting_Guide.txt` — certbot renewal failures, LE rate limits
- `SysAdmin Procedures/Inventory_Asset_Reference.txt` — Section 6, the inventory this table supersedes
- `[internal KB: 802.1x and Active Directory]` — 802.1x/RADIUS background
- `Security & Hardening/Change Radius for WIfi.txt`, `Security & Hardening/Update Radius Certificate.jpg`
- `Security & Hardening/barracuda_web_application_firewall_best_practices_guide.pdf`
- `Security & Hardening/fresh-box-hardening-cheatsheet.txt` — Graylog Sidecar shipping pipeline reused for expiry monitoring; `testssl.sh` for TLS endpoint grading
- `Security & Hardening/secrets-management-guide.md` — private key storage, exposure response
- `Security Procedures/IR_Security_Scenarios_Guide.md` — scenario 13 (TLS / Certificate Issue), scenario 4 (Exposed Credentials)
- `SysAdmin Procedures/Change_Management_Log.txt`

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

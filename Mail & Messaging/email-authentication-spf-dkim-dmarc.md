> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Email Authentication for example.com — SPF, DKIM and DMARC

---

## Overview

SPF, DKIM and DMARC are the three DNS-published records that let receiving mail servers decide
whether a message claiming to be from `@example.com` really came from the organization. Without them, anyone can
forge the organization's domain, and the organization's own legitimate mail is more likely to land in junk folders. With them
configured and enforced, forged mail claiming to be from Example Org gets rejected at the receiver.

Example Org sends mail from Zimbra on `mail.example.com`, and from an unknown number of applications and
third-party services that also send as `@example.com`. Getting the third-party inventory complete is the
hard part of this work — the technical record changes are the easy part. This doc covers how the
three mechanisms work, what Example Org currently publishes, how to rotate DKIM keys in Zimbra, and how to
move DMARC from monitoring to enforcement without breaking legitimate mail.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Owner            | IT lead (it@example.com), IT                                 |
| Environment      | prod — public DNS, affects all outbound mail                           |
| Location         | Public DNS zone for `example.com`, hosted at [FILL IN: DNS provider/registrar and the admin URL] |
| Access           | DNS zone editing at [FILL IN: where the example.com zone is edited and who has access]; DKIM key management on `mail.example.com` as the `zimbra` user |
| Dependencies     | DNS zone availability and DNSSEC (see `_ARCHIVE/superseded-stubs/enable dnssec.txt` (archived 2026-09-11)); Zimbra MTA for DKIM signing; a mailbox or service to receive DMARC aggregate reports |
| Dependents       | Deliverability of every outbound `@example.com` message, from Zimbra and from every third-party sender listed below |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                                |

## How It Works

The three mechanisms answer different questions and only make sense together.

**SPF (Sender Policy Framework)** publishes, in a DNS TXT record, the list of IP addresses and
hosts allowed to send mail for the domain. The receiver compares the connecting server's IP against
that list. The check is against the **envelope sender** (the `MAIL FROM`/Return-Path), not the
`From:` header the user sees.

- What it does not do: it does not protect the visible `From:` address, and it breaks on forwarding
  — when a message is forwarded, the forwarding server's IP is not in the organization's SPF record, so SPF
  fails legitimately.
- Hard limit: an SPF record may trigger at most **10 DNS lookups** when evaluated. Every `include:`,
  `a`, `mx`, `redirect` counts, and nested includes count too. Exceeding it makes the record
  `permerror`, which most receivers treat as a failure. This is the single most common way SPF
  quietly breaks as services are added.
- Ending: `-all` (hard fail, reject anything not listed) or `~all` (soft fail, accept but mark).
  Start at `~all`, move to `-all` once the sender inventory is known to be complete.

**DKIM (DomainKeys Identified Mail)** adds a cryptographic signature header to outbound messages.
The sending server signs selected headers and the body with a private key; the public key is
published in DNS at `<selector>._domainkey.example.com`. The receiver fetches the public key and verifies
the signature.

- What it does not do: it does not say anything about *who* was allowed to send, only that the
  message was signed by a key published for that domain and was not modified in transit. A signature
  survives forwarding, which is why DKIM is the more robust of the two.
- Mailing lists that rewrite subjects or append footers break DKIM signatures.

**DMARC (Domain-based Message Authentication, Reporting and Conformance)** ties the other two to the
visible `From:` header and tells receivers what to do when both fail. It adds two things SPF and
DKIM lack: **alignment** and **reporting**.

- *Alignment* means the domain that passed SPF or DKIM must match the domain in the `From:` header.
  A message can pass SPF for some marketing vendor's own domain while showing `From: someone@example.com`
  — SPF passes, but alignment fails, so DMARC fails. This is exactly what stops the common forgery.
- *Policy* (`p=`) is `none` (monitor only), `quarantine` (junk it), or `reject` (refuse it).
- *Reporting* (`rua=`) gets receivers to send daily aggregate XML reports of what they saw. This is
  the whole value of `p=none` — it is a discovery tool.
- DMARC passes if **either** SPF or DKIM passes *and* is aligned. Both do not need to pass.
- What it does not do: DMARC does not stop lookalike domains (`example-com.com`), display-name spoofing,
  or a compromised real account sending real mail. It only protects the exact domain it is published
  on.

    sender -> receiving MTA
        SPF:   does the connecting IP appear in example.com's TXT SPF record?          (envelope sender)
        DKIM:  does the signature verify against <selector>._domainkey.example.com?    (message integrity)
        DMARC: did SPF or DKIM pass AND align with the From: header domain?       (visible identity)
               -> if not, apply p= policy, and report to rua= address

---

## Current Record Inventory — example.com

Read these from live DNS before making any change. Do not trust the table until the Last verified
column is filled in.

    dig +short TXT example.com                                   # SPF and DMARC-adjacent TXT records
    dig +short TXT _dmarc.example.com                            # DMARC policy record
    dig +short TXT <selector>._domainkey.example.com             # DKIM public key for one selector
    dig +short MX example.com                                    # MX, to confirm inbound routing
    dig +short TXT _mta-sts.example.com                          # MTA-STS, if published

| Record | Host | Current value | Last verified |
|--------|------|---------------|---------------|
| SPF | `example.com` | [FILL IN: full `v=spf1 ...` string from live DNS] | [FILL IN: date verified] |
| DKIM | `[FILL IN: selector]._domainkey.example.com` | [FILL IN: `v=DKIM1; k=rsa; p=...` public key] | [FILL IN: date verified] |
| DMARC | `_dmarc.example.com` | [FILL IN: full `v=DMARC1; p=...` string] | [FILL IN: date verified] |
| MX | `example.com` | [FILL IN: MX hosts and priorities] | [FILL IN: date verified] |
| A | `mail.example.com` | [FILL IN: public IP] | [FILL IN: date verified] |
| PTR (reverse DNS) | [FILL IN: sending public IP] | [FILL IN: should resolve to mail.example.com] | [FILL IN: date verified] |
| MTA-STS | `_mta-sts.example.com` | [FILL IN: policy record, or "not published"] | [FILL IN: date verified] |
| TLS-RPT | `_smtp._tls.example.com` | [FILL IN: value, or "not published"] | [FILL IN: date verified] |
| BIMI | `default._bimi.example.com` | [FILL IN: value, or "not published"] | [FILL IN: date verified] |

Notes on the above:
- Only **one** SPF TXT record may exist for a domain. Two SPF records is a permanent error, and it
  happens when someone adds a vendor's record instead of editing the existing one.
- Reverse DNS for the sending IP is not part of SPF/DKIM/DMARC but is checked by many receivers.
  It must be set by the ISP — [FILL IN: whether PTR for the sending IP is correct, and who to ask at the ISP to change it].
- The `example.com` zone has DNSSEC considerations — see `_ARCHIVE/superseded-stubs/enable dnssec.txt` (archived 2026-09-11). A
  record change that breaks DNSSEC signing takes down more than mail.

### Subdomains

Each sending subdomain needs its own SPF record; SPF does not inherit. DMARC does inherit unless
overridden with `sp=`. Subdomains that send mail: [FILL IN: any subdomains sending mail — e.g. forums.example.com, sign.example.com, data.example.com — and their records].

Non-sending subdomains and parked domains should publish an explicit null SPF and a reject DMARC so
they cannot be used for forgery:

    v=spf1 -all                                             # this host sends no mail at all
    v=DMARC1; p=reject; rua=mailto:[FILL IN: report address]  # reject anything claiming to be from it

Domains Example Org owns that send no mail: [FILL IN: list of parked/secondary domains].

---

## Third-Party Senders Inventory

Every service that sends mail with `@example.com` in the `From:` header needs to be listed here, and
needs either an SPF `include:` or its own DKIM signing, ideally both. A service missing from this
table is a service whose mail will start failing the moment DMARC reaches `p=reject`.

Build this list from three sources: the DMARC aggregate reports (they will show senders you forgot),
the Zimbra/Graylog logs, and asking each department what tools they use to mail members.

| Service / system | What it sends | From address used | SPF mechanism | DKIM signing | Owner / dept | Verified |
|------------------|---------------|-------------------|---------------|--------------|--------------|----------|
| Zimbra (`mail.example.com`) | All staff mail | `*@example.com` | [FILL IN: `ip4:` or `a:mail.example.com`] | [FILL IN: selector] | IT | [FILL IN: date verified] |
| [FILL IN: DMS / Dynamics notifications] | [FILL IN: what it sends] | [FILL IN: From address used] | [FILL IN: SPF mechanism] | [FILL IN: DKIM signing] | [FILL IN: owner / dept] | [FILL IN: date verified] |
| [FILL IN: forums.example.com phpBB] | Forum notifications | [FILL IN: From address used] | [FILL IN: SPF mechanism] | [FILL IN: DKIM signing] | [FILL IN: owner / dept] | [FILL IN: date verified] |
| [FILL IN: sign.example.com DocuSeal] | Signature requests | [FILL IN: From address used] | [FILL IN: SPF mechanism] | [FILL IN: DKIM signing] | [FILL IN: owner / dept] | [FILL IN: date verified] |
| [FILL IN: bulk/marketing mail platform, if any] | Member mailouts | [FILL IN: From address used] | [FILL IN: SPF mechanism] | [FILL IN: DKIM signing] | [FILL IN: owner / dept] | [FILL IN: date verified] |
| [FILL IN: Microsoft 365 / Exchange Online, if it sends] | [FILL IN: what it sends] | [FILL IN: From address used] | `include:spf.protection.outlook.com` | [FILL IN: DKIM signing] | IT | [FILL IN: date verified] |
| [FILL IN: monitoring/alerting senders] | Alerts | [FILL IN: From address used] | [FILL IN: SPF mechanism] | [FILL IN: DKIM signing] | IT | [FILL IN: date verified] |
| [FILL IN: any HR/payroll SaaS] | [FILL IN: what it sends] | [FILL IN: From address used] | [FILL IN: SPF mechanism] | [FILL IN: DKIM signing] | [FILL IN: owner / dept] | [FILL IN: date verified] |

Count the DNS lookups in the resulting SPF record before publishing it. If the list of includes
pushes past 10, the fix is to replace includes with explicit `ip4:` entries for senders with stable
IPs, or to move low-value senders onto a subdomain with its own SPF record — not to hope receivers
are lenient.

    dig +short TXT example.com | tr ' ' '\n' | grep -c 'include:'   # rough count of direct includes; nested ones still count

---

## Operations (Day-2)

### DKIM key rotation in Zimbra

Rotate when: a key is suspected compromised, the key is shorter than 2048 bits, a server is
rebuilt, or on a routine schedule ([FILL IN: agreed rotation interval — annual is a reasonable default]).

Zimbra manages DKIM through `zmdkimkeyutil`, run as the `zimbra` user on `mail.example.com`.

    su - zimbra                                              # all commands run as this user
    /opt/zimbra/libexec/zmdkimkeyutil -q -d example.com           # query: shows the current selector and public key
    /opt/zimbra/libexec/zmdkimkeyutil -a -d example.com           # add: generates a NEW key and prints the DNS TXT record to publish
    /opt/zimbra/libexec/zmdkimkeyutil -u -d example.com           # update: regenerates the key for an existing domain, new selector
    /opt/zimbra/libexec/zmdkimkeyutil -r -d example.com           # remove DKIM signing for the domain — do not do this casually

Rotation order matters. Publish first, switch second, retire last:

1. Query the current key and record the selector in the inventory table above.
2. Generate the new key with `-u`. Zimbra prints a DNS TXT record with a **new selector**.
3. Publish the new selector's TXT record in the `example.com` zone. Leave the old selector's record in
   place.
4. Wait for DNS propagation — at least the TTL of the old record, practically a few hours:

       dig +short TXT <new-selector>._domainkey.example.com @8.8.8.8    # confirm the new key resolves from outside

5. Restart the filtering chain so Zimbra signs with the new key:

       zmamavisdctl restart                                  # picks up the new signing key

6. Send a test message to an external address and inspect the received headers for
   `dkim=pass` and the new selector name.
7. Leave the old selector's DNS record published for at least [FILL IN: retention window — long enough that in-flight and queued mail still verifies; a week is typical], then delete it.

Deleting the old DNS record too early makes already-sent mail fail verification. That is the usual
rotation mistake.

Key length should be 2048-bit. Some DNS providers require splitting a 2048-bit key across multiple
quoted strings in one TXT record — [FILL IN: whether the example.com DNS provider needs the key split, and how it handles long TXT values].

### Publishing an SPF change

1. Read the current record and copy it somewhere before editing.
2. Make the change in a text editor, not directly in the DNS panel.
3. Count DNS lookups. Stay under 10.
4. Publish, then verify from an external resolver:

       dig +short TXT example.com @8.8.8.8                        # confirm exactly one v=spf1 record, and it is the new one

5. Send test mail to an external address and read the `Authentication-Results` header.

Never add a second SPF TXT record. Merge into the existing one.

### DMARC policy progression

The point of the progression is that each step is gated on evidence from the aggregate reports, not
on elapsed time. Do not advance because a month has passed; advance because the reports show that
all legitimate mail is passing.

**Step 0 — prepare.** Make sure there is somewhere for reports to go. A dedicated mailbox is
better than a person's inbox, because the volume is high and the content is XML.
Report address: [FILL IN: the mailbox or third-party DMARC processing service used for rua=].

**Step 1 — `p=none`, monitoring only.**

    v=DMARC1; p=none; rua=mailto:[FILL IN: aggregate report address]; ruf=mailto:[FILL IN: forensic address, or omit]; fo=1; adkim=r; aspf=r; pct=100

This changes nothing about how mail is handled. It only turns on reporting.

*Gate to advance:* aggregate reports have been reviewed for at least [FILL IN: agreed monitoring window — 4 to 6 weeks is typical], every sending source in the reports has been identified and appears in the third-party inventory above, and legitimate sources are passing DMARC (aligned SPF or aligned DKIM) at a rate you are comfortable with — realistically well above 95%, with every remaining failure explained. Unexplained failures mean you are not ready.

**Step 2 — `p=quarantine`, ramped with `pct=`.**

    v=DMARC1; p=quarantine; pct=10; rua=mailto:[FILL IN: aggregate report address]; fo=1     # 10% of failing mail junked
    v=DMARC1; p=quarantine; pct=50; rua=mailto:[FILL IN: aggregate report address]; fo=1     # then 50%
    v=DMARC1; p=quarantine; pct=100; rua=mailto:[FILL IN: aggregate report address]; fo=1    # then all failing mail

Advance one step at a time, waiting at least [FILL IN: agreed soak time per step — a week or two] between increments.

*Gate to advance:* at `pct=100` quarantine, no helpdesk reports of legitimate Example Org mail landing in
recipients' junk folders, and the pass rate in the reports has not dropped.

**Step 3 — `p=reject`.**

    v=DMARC1; p=reject; rua=mailto:[FILL IN: aggregate report address]; fo=1; adkim=s; aspf=s   # strict alignment once everything is known to sign correctly

*Gate to enter:* everything above holds, and there is an agreed rollback — if legitimate mail starts
bouncing, revert the record to `p=quarantine` immediately; DNS TTL is the recovery time, so keep the
DMARC record's TTL short ([FILL IN: TTL used on the _dmarc record]) during the progression.

Only tighten alignment (`adkim=s`, `aspf=s`) after reject is stable at relaxed alignment. Strict
alignment breaks subdomain senders that were previously fine.

Record the date of each step in the Decisions & History table below. Current policy stage:
[FILL IN: which step example.com is at today].

### Reading aggregate reports

Aggregate (RUA) reports arrive daily as gzipped XML, one per reporting receiver. Each report covers
a 24-hour window and lists, per source IP, how many messages were seen and what SPF/DKIM/DMARC
results they got. They contain no message content.

The fields that matter in each `<record>`:

- `<source_ip>` — who sent it. Unfamiliar IPs are either a forgotten legitimate sender or a forger.
- `<count>` — volume from that IP.
- `<policy_evaluated><disposition>` — what the receiver did (`none`/`quarantine`/`reject`).
- `<policy_evaluated><dkim>` and `<spf>` — the **aligned** results, which is what DMARC actually
  judges on.
- `<auth_results>` — the raw SPF and DKIM results, which can pass while the aligned result fails.
  When `auth_results` shows SPF pass but `policy_evaluated` shows SPF fail, the problem is
  alignment, not SPF.

Reading them by hand works for a small domain but gets old fast:

    gunzip -c report.xml.gz | xmllint --format -            # pretty-print one report for reading
    gunzip -c report.xml.gz | grep -A6 '<row>'              # just the per-source rows

For ongoing work, a DMARC report processing service or self-hosted parser is worth it.
[FILL IN: whether Example Org uses a DMARC reporting service, and if so which].

Triage pattern for each unfamiliar source: is it the organization's own infrastructure, a third-party Example Org hired,
a forwarder (mail that was legitimately relayed and lost its SPF), or a forger? Only the last one
should be allowed to fail once enforcement is on. Forwarders are the reason DKIM matters — a
DKIM-signed message survives forwarding and still passes DMARC.

Forensic (RUF) reports contain message headers and sometimes content, so they carry privacy
implications and most large receivers do not send them. Consider whether to request them at all:
[FILL IN: decision on whether to publish a ruf= address, given privacy considerations].

---

## Troubleshooting

### Symptom: legitimate Example Org mail is being rejected or junked by a recipient
- Check the `Authentication-Results` header of a copy of the message the recipient received — it
  states `spf=`, `dkim=` and `dmarc=` results directly, and is the fastest route to the answer.
- Then determine which of the three failed, and whether it failed outright or failed alignment.

### Symptom: SPF returns permerror
- Likely cause: more than 10 DNS lookups, or two SPF records published.
- Check:  `dig +short TXT example.com` &nbsp;# look for more than one `v=spf1` string
- Fix:    merge duplicate records into one; replace `include:` entries with `ip4:` for senders with
  fixed IPs to get back under the lookup limit.

### Symptom: a new SaaS vendor's mail as @example.com is failing
- Likely cause: vendor not in SPF, or the vendor signs DKIM with its own domain so alignment fails.
- Fix:    add the vendor's SPF include, and set up domain-level DKIM signing with the vendor so the
  signature is on `example.com` rather than the vendor's domain. Most vendors support this and call it
  "custom sending domain" or similar. SPF alone will not survive `p=reject` if the vendor uses its
  own envelope domain.

### Symptom: DKIM was passing, now failing after a change
- Likely cause: DNS record for the selector was removed too early during rotation, or the key was
  regenerated without publishing the new record.
- Check:  `dig +short TXT <selector>._domainkey.example.com` &nbsp;# empty means the record is gone
- Check:  `/opt/zimbra/libexec/zmdkimkeyutil -q -d example.com` &nbsp;# what Zimbra thinks it is signing with
- Fix:    republish the selector whose key Zimbra is actually using.

### Symptom: mail to one large provider fails, everyone else is fine
- Likely cause: that provider has stricter requirements (bulk sender rules, required DMARC, required
  one-click unsubscribe on bulk mail) or has rate-limited the sending IP.
- Check:  the bounce text — large providers name the specific policy in the rejection.
- Fix:    per the named policy. Do not weaken the organization's DMARC to work around one receiver.

### Symptom: mail forwarded by a staff member to a personal address fails DMARC
- Expected behaviour. Forwarding breaks SPF. It is why DKIM must be working before enforcement.
- Fix:    confirm DKIM signs all outbound mail and is aligned. Nothing to fix at the receiver end.

### Symptom: mailing-list mail from @example.com fails at subscribers
- Likely cause: the list modifies the subject or body, breaking the DKIM signature, and the list
  relays from its own IP, breaking SPF.
- Fix:    well-behaved lists implement ARC or rewrite the `From:` address. For organization-run lists, see
  the distribution list section of `zimbra-mail-administration-guide.md`.

### Symptom: someone reports a phishing email that appears to come from an Example Org executive
- Check the `From:` header domain. If it is genuinely `@example.com` and DMARC is at `p=reject`, then
  either enforcement is not actually live or the account is compromised — check the Zimbra logs and
  treat as an incident (`Security Procedures/IR_Security_Scenarios_Guide.md`).
- If it is a lookalike domain or a display-name spoof with a different sending domain, DMARC on
  `example.com` cannot stop it. That needs receiver-side controls and user awareness —
  `Security & Hardening/helpdesk-social-engineering-awareness_1.txt`.

---

## Security

- **Exposure**: everything here is public DNS by design. The private DKIM key is the one secret,
  and it lives on `mail.example.com` under `/opt/zimbra` — never copy it off the host, never paste it
  into a ticket.
- **Access**: DNS zone editing is the sensitive control. Anyone who can edit the `example.com` zone can
  redirect mail entirely. [FILL IN: who holds DNS registrar/zone credentials, and whether MFA is enforced on that account].
- **Registrar lock**: [FILL IN: whether domain transfer lock is enabled on example.com].
- **DNSSEC**: see `_ARCHIVE/superseded-stubs/enable dnssec.txt` (archived 2026-09-11). Signed zones make it harder to spoof the
  SPF/DKIM/DMARC lookups themselves.
- **Secrets**: DKIM private keys on the mail host; DNS credentials in [FILL IN: password manager location]. Never in this doc.

## Monitoring & Alerting

- DMARC aggregate reports to [FILL IN: report mailbox] — reviewed [FILL IN: review cadence, and by whom].
- Record-change monitoring: an external check that alerts if the SPF, DKIM or DMARC record for
  `example.com` changes or disappears. [FILL IN: whether such a check exists, and where it alerts].
- DKIM key expiry/rotation reminder: [FILL IN: calendar reminder or ticket that triggers rotation].
- Baseline: [FILL IN: typical daily message volume and DMARC pass rate once reports have been reviewed].

## Disaster Recovery

- **If the SPF/DMARC record is accidentally deleted**: outbound mail deliverability degrades within
  minutes to hours (TTL-dependent) and forgery protection disappears immediately. Republish from the
  inventory table above — which is why the Current value column must be kept filled in. Keep an
  exported copy of the full `example.com` zone file outside the DNS provider:
  [FILL IN: where the zone file export is kept, and how often it is refreshed].
- **If the DKIM private key is lost** (mail server rebuilt from a backup without it): generate a new
  key with `zmdkimkeyutil -u`, publish the new selector, and accept that mail signed with the old key
  can no longer be verified. Not an outage, but do it promptly.
- **If the DKIM private key is exposed**: rotate immediately and remove the old selector's DNS record
  as soon as in-flight mail allows — a leaked key lets anyone sign mail as `example.com`.
- **RTO/RPO**: [FILL IN: agreed expectations for DNS record restoration].

## Decisions & History (ADR-lite)

| Date       | Decision / Change                                        | Why / Ticket |
|------------|----------------------------------------------------------|--------------|
| 2026-09-11 | Guide created; record inventory not yet verified          | Documentation consolidation |
| [FILL IN: date]  | [FILL IN: date SPF record last changed and why]           | [FILL IN: why / ticket]    |
| [FILL IN: date]  | [FILL IN: date DKIM last rotated, and the selector used]  | [FILL IN: why / ticket]    |
| [FILL IN: date]  | [FILL IN: date DMARC moved to p=none]                     | [FILL IN: why / ticket]    |
| [FILL IN: date]  | [FILL IN: date DMARC moved to quarantine / reject]        | [FILL IN: why / ticket]    |

## References

- `Mail & Messaging/zimbra-mail-administration-guide.md` — the sending platform, DKIM signing, queues
- `Microsoft 365/m365-entra-admin-guide.md` — if Exchange Online sends as @example.com it belongs in the inventory
- `_ARCHIVE/superseded-stubs/enable dnssec.txt` (archived 2026-09-11) — DNSSEC on the example.com zone
- `Security & Hardening/helpdesk-social-engineering-awareness_1.txt` — the attacks DMARC does not stop
- `Security Procedures/IR_Security_Scenarios_Guide.md` — compromised-account and phishing response
- `Security Procedures/WORKFLOW.txt` and the phishing triage toolkit — analysing a reported message
- Upstream: RFC 7208 (SPF), RFC 6376 (DKIM), RFC 7489 (DMARC)

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

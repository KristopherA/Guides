# Monitoring & Alerting

> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

## Overview

This guide describes what the organization actually monitors today, what it does not, and how
Graylog — the one piece of monitoring infrastructure that is genuinely in use and
genuinely relied on — is administered. It is written for the sysadmin on duty.

Be clear about the starting position: the organization has centralised **logging**, not
centralised **monitoring**. Graylog at graylog01.example.com collects logs and is used
daily for mail troubleshooting and fail2ban unbans. Everything else — service
uptime, certificate expiry, disk capacity, replication lag, backup success — is
either checked by a human when someone complains, or not checked at all. The most
valuable part of this document is therefore the gap analysis, not the Graylog
reference. Read that section first.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Owner            | Sysadmin (it@example.com)                                             |
| Environment      | prod                                                                   |
| Location         | Graylog: graylog01.example.com. Web UI reached as `graylog.example.com:7555` in existing runbooks — [CONFIRM: whether graylog.example.com is a CNAME/alias for graylog01.example.com, and whether 7555 is the current UI port] |
| Access           | Graylog web UI over HTTPS, admin credentials in 1Password / IT vault. SSH to the Graylog host as `orgadmin` — [FILL IN: is SSH to graylog01 restricted to a management VLAN or VPN?] |
| Dependencies     | Network reachability from log sources to the Graylog input port; Elasticsearch/OpenSearch backing store on [FILL IN: same host or separate?]; MongoDB for Graylog config; NTP (skewed clocks make searches lie); DNS |
| Dependents       | Mail troubleshooting on mail.example.com, fail2ban unban workflow, Snort/pfSense event review, any future alerting |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                                |

## How It Works

Log sources ship messages to a Graylog **input**. Graylog optionally runs them
through **pipeline rules** that parse fields, then routes them into **streams**.
Streams are the unit of everything downstream: permissions, dashboards, alert
conditions, and index sets are all attached to streams, not to raw searches.
Messages land in an **index set**, which is an Elasticsearch/OpenSearch index
family with its own rotation and retention policy.

    log source (syslog / GELF / Beats)
      -> Graylog input            # listener on a port, per protocol
      -> pipeline rules           # optional parsing, e.g. parse_json for Snort alerts
      -> stream routing           # "is this a mail message? a fail2ban message?"
      -> index set                # rotation + retention live here
      -> search / dashboard / alert

Known sources today:

- mail.example.com — Zimbra mail logs and fail2ban actions. This is the source the
  fail2ban unban runbook depends on.
- pfSense / Snort at the edge — [CONFIRM: the Snort 3 -> Graylog `parse_json`
  pipeline with a `snort_` field prefix is described in
  `Security Procedures/modules/detection-validation.md`, but that document
  presents it as a worked example. Confirm whether it is actually deployed.]
- [FILL IN: any other hosts shipping logs — Proxmox nodes, MySQL hosts, auth2/auth4,
  files, forums, sign, db02]

Config and data locations on the Graylog host:

- `/etc/graylog/server/server.conf`   # Graylog node config
- `/var/lib/graylog-server/`          # journal (on-disk buffer before indexing)
- [FILL IN: Elasticsearch/OpenSearch data path and the filesystem it lives on]

---

## What Is Monitored Today vs. What Is Not

This is an honest audit, not an aspiration.

### Monitored today

| Thing | How | Who sees it |
|---|---|---|
| Mail auth failures and fail2ban bans on mail.example.com | Graylog Overview dashboard, Fail2Ban tab | Sysadmin, only when a user reports being locked out |
| Snort/edge alerts | Graylog search, if the pipeline is in place | Sysadmin, on demand |
| Log arrival itself | Nothing. If a host stops shipping, nobody is told | — |

Note the pattern: every item above is **pull**. A human goes and looks. Nothing
pushes a notification to a person. That is the core gap.

### Not monitored at all (gap analysis)

| Gap | Risk if it fails silently | Priority |
|---|---|---|
| Certificate expiry (LDAPS on auth2/auth4, wildcard cert on MySQL hosts, web certs on sign/files/forums) | Hard outage at a predictable moment nobody predicted. The organization has already had cert-driven work on LDAPS and MySQL | **Highest — do this first** |
| Disk capacity (Proxmox nodes, MySQL datadirs, Graylog index store, mail spool) | Full disk stops MySQL writes, stops Graylog indexing, can corrupt tables | **Highest — do this second** |
| Service uptime / HTTP reachability of public services (sign.example.com, files.example.com, forums.example.com, mail.example.com) | Users report the outage before IT knows | High |
| MySQL replication health (db01.example.com -> db02.example.com) | Replica silently drifts; discovered only when a failover is needed | High |
| Backup success/failure (Retrospect jobs, mysqldump cron, binlog copies) | An untested, unwatched backup is a rumour | High |
| Graylog itself — log flow stopping, journal backing up, index errors | Every other Graylog-based check becomes a lie | High |
| Proxmox host and guest health (node up, VM/CT running, memory pressure) | [FILL IN: does Proxmox's own notification email work and go to a monitored address?] | Medium |
| Docker container health and restart loops | Medium |
| pfSense / Snort service health (as opposed to its alerts) | Medium |
| UPS / power and environmental | [FILL IN: is there UPS monitoring in the server room?] | Medium |
| Domain and DNS expiry, DNSSEC validity | Low frequency, high blast radius | Medium |
| AD replication and DC health (dcdiag) | Medium |

A realistic reading of this table: the organization has roughly one twelfth of a monitoring
system. The two rows marked highest priority are cheap to implement and would have
caught the majority of past self-inflicted outages. Start there.

---

## Monitoring Coverage Matrix

One row per service. Fill this in as checks are actually implemented — a row with
values still marked [FILL IN] means that service is unmonitored, which is itself
the useful signal. Do not pre-fill a row to make the table look finished.

| Service | Host | What is checked | Check interval | Alerts to | Escalation |
|---|---|---|---|---|---|
| Log aggregation (Graylog) | graylog01.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Mail (Zimbra) | mail.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| MySQL primary | db01.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| MySQL replica | db02.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| MySQL (DMS / Dynamics GP backend) | erp-db.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| MySQL | app-db01.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| MySQL | app-db02.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| MySQL | prod01.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Document signing | sign.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| File services | files.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Forums | forums.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Directory / LDAPS | auth2.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Directory / LDAPS | auth4.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Edge firewall / IDS | pfSense + Snort | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Virtualisation | Proxmox cluster — [FILL IN: node names] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| Backup | Retrospect server — [FILL IN: hostname] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |

---

## Graylog Administration

### Inputs

An input is a listener: a protocol and a port on a Graylog node. Sources are
configured to ship to it. Inputs are global (all nodes) or node-local.

    System -> Inputs                    # in the web UI: list, create, start, stop

Check an input is actually receiving before you debug anything downstream. The
input list shows a live throughput counter per input; zero there means the
problem is the network or the sender, not Graylog.

Current inputs: [FILL IN: list each input — type (Syslog UDP/TCP, GELF, Beats),
port, and which hosts ship to it]

When you add a source host, you add it to an existing input — you do not normally
create a new input per host. Create a new input only for a new protocol or a
different port.

### Streams

A stream is a rule-based view of incoming messages. Streams matter more than
they look:

- Alerts (event definitions) are scoped to a stream.
- Dashboard widgets are usually scoped to a stream.
- Retention differs per stream only if the stream writes to its own index set.
- User permissions are granted per stream.

Practical rule: if you want to alert on a class of message, first make sure that
class has its own stream. Alerting off an ad-hoc search across "All messages" is
slower and harder to reason about.

    Streams -> [stream] -> Manage Rules      # define the match conditions
    Streams -> [stream] -> More -> Index Set # which index set it writes to

Current streams: [FILL IN: list each stream, its matching rule, and its index set]

### Saved searches and dashboards

Two real conventions already exist and are easy to get wrong, so they are recorded
here explicitly:

- The fail2ban investigation view is the **Fail2Ban tab inside the Overview
  dashboard**, not the separate dashboard also called "Fail2ban". The two show
  different things. The existing runbook is emphatic about this; respect it.
- The default search time range is short. Ban investigations routinely need
  7 to 30 days. Widen the range before concluding an IP is not there.

Save any search you run more than twice. An unsaved search that you rebuild from
memory each time is where mistakes come from.

### Queries that are already useful

Mail and fail2ban work:

    <user@example.com>                     # find a user's mail activity by address; widen range to 7-30d
    "203.0.113.45"                         # find an IP across all messages (quoted, to match exactly)
    source:mail.example.com                     # everything from the mail host
    source:mail.example.com AND fail2ban        # ban/unban actions on the mail host

[FILL IN: the exact field names Graylog parses out of the Zimbra and fail2ban
messages at the organization — e.g. whether there is a parsed `client_ip` or jail name field,
or whether these are full-text searches only. Run one known-good search and record
the field names from the message detail pane.]

Snort / edge work, if the JSON pipeline is deployed:

    snort_sid:<SID>                        # did a specific rule fire
    # then confirm snort_class / snort_priority / snort_src_addr are populated, not missing

Generic operational queries worth saving:

    level:<=3                              # syslog severity error and worse
    source:<host> AND level:<=3            # errors from one host
    NOT source:mail.example.com                 # what else is even shipping logs

### Retention and index rotation

Retention in Graylog is a property of the **index set**, not of a stream and not
of the server as a whole. Every index set has:

- A rotation strategy — by index size, by message count, or by time.
- A retention strategy — delete, close, or archive the oldest index once the
  configured number of indices is exceeded.

    System -> Indices -> [index set] -> Edit      # rotation and retention live here

The number that matters is **max indices x max index size**, which is the real
disk ceiling. If nobody has ever computed that against the size of the filesystem,
it is the single most likely cause of a future Graylog outage.

Current settings: [FILL IN: for each index set — rotation strategy and threshold,
retention strategy, number of indices retained, and the resulting worst-case disk
usage] against [FILL IN: size of the filesystem holding the index data]

Retention policy target: [CONFIRM: proposed default — 90 days of searchable mail
and security logs, which is long enough to investigate a ban or a compromise
reported weeks late, and short enough to bound disk. Confirm against any organizational or
legal retention requirement before adopting.]

Do not change rotation settings on a full disk. Fix the disk first (see
Troubleshooting), then adjust retention so it does not recur.

### Health checks

    curl -sf http://localhost:9000/api/system/lbstatus     # expect: ALIVE
    systemctl status graylog-server --no-pager             # service up
    systemctl status mongod --no-pager                     # Graylog config store
    df -h                                                  # index filesystem headroom

[FILL IN: confirm the API port and whether it is 9000 locally behind a proxy that
publishes 7555.]

---

## Alert Routing and On-Call

Nothing below is in force yet. These are proposed defaults to argue with, not
current practice.

### Routing

| Severity | Definition | Route | Response time |
|---|---|---|---|
| P1 | Public service down, data loss risk, security incident in progress | [CONFIRM: proposed default — page the sysadmin by SMS and email to it@example.com] | [CONFIRM: proposed default — 30 minutes, 24/7] |
| P2 | Degraded service, redundancy lost (e.g. replica broken), backup failed | [CONFIRM: proposed default — email to an IT distribution address] | [CONFIRM: proposed default — next business day] |
| P3 | Predictable future failure (cert expiring in 21 days, disk at 80%) | [CONFIRM: proposed default — email digest, once daily] | [CONFIRM: proposed default — within the week] |

[FILL IN: the actual destination addresses — an IT distribution list, an SMS
gateway or paging service, and whether a ticketing queue should receive P2/P3.]

### On-call expectations

[CONFIRM: proposed default — IT is effectively a one-person on-call rotation.
Written honestly, that means: P1 alerts reach one person at any hour; P2 and P3
are business hours only; and if that person is unavailable, the escalation is to
[FILL IN: named backup contact or vendor], not to an unread inbox.]

Two things must be true before any paging alert is enabled:

1. The alert has a documented response — a runbook section that says what to do.
   An alert with no action is noise with extra steps.
2. Someone has agreed to receive it. An alert routed to a mailbox nobody reads is
   worse than no alert, because it creates the belief that the thing is watched.

### Escalation order

1. Sysadmin on duty — [FILL IN]
2. Backup contact — [FILL IN]
3. Vendor / external support — [FILL IN: per service, e.g. Retrospect support,
   Zimbra support, ISP]

---

## What a Good Alert Is

An alert is worth creating only if all four are true:

1. **It is actionable.** There is something a human can do at 2am. "Disk 90% full"
   is actionable. "CPU spiked for 30 seconds" is not.
2. **It is specific.** It names the host and the condition, so the responder knows
   where to start without opening three dashboards.
3. **It fires rarely.** If it fires daily and is usually ignored, it has already
   failed. The correct response to a noisy alert is to fix the threshold or delete
   the alert, never to train yourself to ignore it.
4. **It has a runbook.** A link to the section of a guide that says what to do.

Symptom-based alerts beat cause-based alerts. Alert on "sign.example.com is not
responding" rather than on every individual reason it might not be responding.
Cause-based alerts multiply; symptom-based alerts stay at one per service.

Thresholds: no default thresholds are proposed here, because a threshold invented
without baseline data is a guess that will either flood you or never fire. Set
each one after watching the metric for [CONFIRM: proposed default — two weeks] and
record the observed normal range next to the threshold in the coverage matrix.
[FILL IN: baseline values per service once observed]

### The rule for muting

Muting is allowed. Silent, permanent, undocumented muting is not.

- Every mute has an **expiry**. Maximum [CONFIRM: proposed default — 7 days].
  A mute that needs to outlive a week is a broken alert definition, not a mute.
- Every mute has a **reason** recorded — in the ticket, in the change log, or in
  `SysAdmin Procedures/Change_Management_Log.txt`.
- Mute the **specific alert on the specific host**, never a whole stream or the
  whole notification channel.
- Planned maintenance: mute before you start, and set the expiry to the end of the
  window, not to "forever".
- When a mute expires and the alert immediately fires again, that is information.
  Fix the condition or fix the alert. Do not re-mute twice in a row without
  raising it as work.

---

## Synthetic and External Checks

Internal monitoring cannot tell you that a service is reachable from outside. If
the check runs on the same network — or worse, the same host — as the thing it
checks, it will report healthy through a firewall misconfiguration, a DNS failure,
or an ISP outage.

Public services that need an external check:

| Service | URL | Expected response |
|---|---|---|
| Document signing | https://sign.example.com/ | [FILL IN: health endpoint and expected status/body] |
| File services | https://files.example.com/ | [FILL IN] |
| Forums | https://forums.example.com/ | [FILL IN] |
| Mail | mail.example.com — SMTP 25/587, IMAPS 993, web UI | [FILL IN: which ports and endpoints should be probed externally] |

Local probe form, for testing a check by hand before automating it:

    curl -sS -o /dev/null -w '%{http_code} %{time_total}s\n' https://sign.example.com/   # status and latency
    openssl s_client -connect sign.example.com:443 -servername sign.example.com </dev/null    # TLS handshake and chain

[CONFIRM: proposed default — external checks run from a third-party service or a
cheap VPS outside the organization network, every 60 seconds, alerting only after two
consecutive failures. Two consecutive failures avoids paging on a single dropped
packet.] [FILL IN: which external checking service, if any, is adopted.]

Do not point an external synthetic check at an authenticated endpoint with stored
credentials. Probe an unauthenticated health path instead.

---

## The Two Highest-Value Additions

If only two things ever get built, build these.

### 1. Certificate expiry monitoring

The organization runs enough TLS to make this the most likely source of a surprise outage:
LDAPS on the domain controllers, the wildcard cert used by MySQL, public web
certs, RADIUS. Each of these has already generated documented emergency work in
this library. Every one of those events was predictable weeks in advance.

Check form:

    echo | openssl s_client -connect auth2.example.com:636 2>/dev/null | openssl x509 -noout -enddate    # LDAPS cert expiry
    echo | openssl s_client -connect sign.example.com:443 -servername sign.example.com 2>/dev/null | openssl x509 -noout -enddate
    openssl x509 -in /etc/mysql/wildcardcert.pem -noout -enddate                                    # local cert file on a MySQL host

Certificates to watch: [FILL IN: full inventory — for each, the host, port or file
path, issuing CA, and who renews it]

[CONFIRM: proposed default — warn at 30 days, escalate at 14 days, page at 7 days.
30 days is chosen because it is long enough to complete a CA issuance and an
LDAPS replacement, which is the slowest renewal the organization performs.]

Related procedures: `Security & Hardening/LDAPs certificate replacement procedure.txt`,
`Databases/mysql certificate rotation.txt`.

### 2. Disk capacity monitoring

A full disk is the failure mode that damages data rather than just interrupting
service. MySQL stops accepting writes, Graylog stops indexing, mail queues back up.

Check form:

    df -h                                        # human check
    df -P | awk 'NR>1 && int($5)>85 {print}'     # any filesystem over 85% — use in a cron check

Filesystems that matter most: MySQL datadirs (`/var/lib/mysql`) on db01.example.com,
db02.example.com, erp-db.example.com, app-db01.example.com, app-db02.example.com, prod01.example.com; the
Graylog index filesystem on graylog01.example.com; the mail spool on mail.example.com; the
Proxmox storage volumes; the Retrospect backup target.

[CONFIRM: proposed default — warn at 80%, page at 90%. Warn early enough that
growth can be handled in business hours, page before MySQL is at risk.] Adjust for
volumes that legitimately run near full, and record the exception in the matrix.
[FILL IN: per-filesystem sizes and observed growth rate]

Also worth watching alongside raw capacity: inode exhaustion (`df -i`) on hosts
that write many small files, and the Graylog journal directory, which grows when
indexing stalls even if the index filesystem is fine.

---

## Operations (Day-2)

### Add a new log source to Graylog

1. Confirm which existing input the source should ship to (protocol and port).
2. Configure the sender — on Ubuntu, an rsyslog forwarding rule:

    # /etc/rsyslog.d/90-graylog.conf
    *.* @graylog01.example.com:[FILL IN: input port]     # @ = UDP, @@ = TCP

    systemctl restart rsyslog                       # apply on the sending host

3. Confirm arrival in Graylog: search `source:<newhost>` over the last 5 minutes.
4. Add or extend a stream rule so the messages are routed somewhere meaningful.
5. Add the host to the coverage matrix in this document.

### Review alert noise

[CONFIRM: proposed default — monthly.] For each alert that fired: was it acted on?
If an alert fired more than [CONFIRM: proposed default — 4 times] in the month and
was never acted on, either the threshold is wrong or the alert should be deleted.
Record the decision in `SysAdmin Procedures/Change_Management_Log.txt`.

### Verify that monitoring itself works

An unverified monitor is indistinguishable from no monitor. Once a quarter:

- Stop a non-critical log source and confirm a "no messages received" condition is
  noticed. [FILL IN: whether such a condition exists yet — as of this draft it does not.]
- Send a deliberate test message and confirm it appears in the expected stream:

    logger -n graylog01.example.com -P [FILL IN: input port] "graylog test message from $(hostname) $(date)"

- Trigger one alert deliberately and confirm it reaches a human, not just a log line.

---

## Troubleshooting

### Symptom: no logs arriving in Graylog from a host

- Likely cause: sender misconfigured, network path blocked, or the input is stopped.
- Check: `System -> Inputs` in the UI — is the input running, and is its throughput
  counter non-zero?
- Check, from the sending host:

    logger -n graylog01.example.com -P [FILL IN: input port] "test $(date)"   # send a message by hand
    ss -tunp | grep -E '514|12201|[FILL IN: input port]'                 # is the sender holding a connection

- Check, on graylog01: `ss -lnup` and `ss -lntp` — is anything listening on the
  input port at all?
- Check the firewall path: pfSense rules between the source VLAN and graylog01,
  and `ufw status` on graylog01 itself.
- Fix: restart the input from the UI; restart rsyslog on the sender; open the
  firewall path. If throughput is non-zero but you cannot find the messages,
  the messages are arriving and the problem is stream routing or your search time
  range — widen the range and search `source:<host>` with no other terms.

### Symptom: messages arriving but not appearing in the expected stream

- Likely cause: stream rule does not match, or a pipeline rule is dropping or
  rewriting the field the rule depends on.
- Check: open a message in the search detail pane and read its actual fields.
  Stream rules match on fields, and the field you assume exists often does not.
- Fix: correct the stream rule. Test with the stream's built-in rule tester
  against a real message before saving.

### Symptom: Graylog index is full / indexing has stopped

- Likely cause: the index set hit its configured maximum, retention is not deleting,
  or Elasticsearch/OpenSearch has gone read-only because the disk watermark was hit.
- Check:

    df -h                                                    # is the index filesystem actually full
    curl -s localhost:9200/_cat/indices?v                    # index list, sizes, health — [CONFIRM: ES/OS port]
    curl -s localhost:9200/_cluster/health?pretty            # status: green / yellow / red

- Check the Graylog UI: `System -> Overview` will show indexing failures, and
  `System -> Indices` shows each index set against its limits.
- Fix, in order: free disk space first (delete the oldest closed indices via
  `System -> Indices`, not by deleting files on disk), then clear the read-only
  block if the search backend set one:

    curl -XPUT localhost:9200/_all/_settings -H 'Content-Type: application/json' \
      -d '{"index.blocks.read_only_allow_delete": null}'     # lifts the disk-watermark read-only block

  Then reduce retention so it does not recur. Do not simply raise the index count
  on a disk that is already full.

### Symptom: disk pressure on the Graylog host

- Likely cause: index growth, or the Graylog journal backing up because indexing
  has stalled. These look similar and have different fixes.
- Check:

    df -h /var/lib/graylog-server                            # journal filesystem
    du -sh /var/lib/graylog-server/journal                   # journal size — should be small and stable
    df -i                                                    # inodes, occasionally the real culprit

- If the journal is large and growing, the indexing backend is the problem, not
  the journal. Fix the search backend and the journal drains on its own.
- If the index data is simply large, reduce retention on the largest index set.
- Never delete index files from the filesystem directly. Close and delete indices
  through Graylog so its metadata stays consistent.

### Symptom: searches return nothing for a time range you know had activity

- Likely cause: clock skew on the sending host, or the search is scoped to a
  stream that does not contain those messages.
- Check: `timedatectl` on the sending host; compare with graylog01. Then repeat the
  search against "All messages" instead of a stream.
- Fix: correct NTP on the sender. Messages with a wrong timestamp are indexed at
  that wrong time and will not reappear when the clock is fixed — note the offset
  and search accordingly.

### Symptom: Graylog web UI unreachable but the host is up

- Check: `systemctl status graylog-server mongod --no-pager`. Graylog will not
  start without MongoDB.
- Check: `journalctl -u graylog-server --since "30 min ago"`.
- Fix: start MongoDB first, then graylog-server. If Graylog starts but reports
  "no active nodes", the search backend is down — start that before investigating
  Graylog further.

---

## Security

- Exposure: [FILL IN: is the Graylog UI internet-facing or internal/VPN only?]
  It should not be internet-facing.
- Auth: Graylog admin login, credentials in 1Password / IT vault.
  [FILL IN: whether Graylog is integrated with LDAP/AD for user accounts — if it
  is, see `Active Directory/ldap-connection-reference.md`.]
- Logs are evidence. Anyone with Graylog read access can see mail metadata and
  user IP addresses. Grant stream-level permissions rather than global read.
- Certificates: [FILL IN: what cert the Graylog UI presents and who renews it]
- Secrets: never in this document. Graylog admin password and any input shared
  secrets live in 1Password.

---

## Roadmap

A defensible order of work, given one sysadmin:

1. Certificate expiry checks. Highest value, lowest effort, catches the failure
   mode that has already bitten the organization.
2. Disk capacity checks on MySQL hosts, Graylog, and mail.
3. Compute Graylog's worst-case index disk usage and set retention deliberately.
4. External synthetic checks on the four public services.
5. A "no logs received from host X" alert — this is the check that makes every
   other Graylog-based check trustworthy.
6. MySQL replication lag and backup success alerts.
7. Everything else in the gap table.

## Decisions & History (ADR-lite)

| Date       | Decision / Change                                   | Why / Ticket |
|------------|-----------------------------------------------------|--------------|
| 2026-09-11 | Initial draft. Documents Graylog as the only real monitoring, and records the gap analysis | Library consolidation |

## References

- The internal fail2ban unban runbook — fail2ban jails, unban commands, and the Graylog Overview/Fail2Ban tab convention
- The internal mail-host unban walkthrough in Graylog
- The internal Snort IP-unblock procedure
- `Security Procedures/modules/detection-validation.md` — the Snort -> Graylog -> Sigma chain and how to verify each link
- `Security & Hardening/LDAPs certificate replacement procedure.txt`
- `Databases/mysql certificate rotation.txt`
- `SysAdmin Procedures/Daily_SysAdmin_Procedures.txt`
- `SysAdmin Procedures/Change_Management_Log.txt`
- Graylog documentation: https://go2docs.graylog.org/

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

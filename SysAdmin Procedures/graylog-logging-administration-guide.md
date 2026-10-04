# Graylog Central Logging — Administration Guide

> Status: initial draft, 2026-10-04. Contains [FILL IN] markers — see INDEX.md.

## Overview

Graylog is the organization's central log platform. It receives logs from mail.example.com (Zimbra and
fail2ban), from the edge (pfSense / Snort, where configured) and from servers enrolled
through Graylog Sidecar. It is used daily for two things: mail troubleshooting, and
finding which fail2ban jail a locked-out user is sitting in before unbanning them.

This guide is the deep administration companion to
[`SysAdmin Procedures/monitoring-alerting-guide.md`](monitoring-alerting-guide.md). That
guide holds the honest gap analysis (the organization has centralised *logging*, not centralised
*monitoring*), the alert-routing proposal, and a short Graylog reference. **Read its gap
analysis first; it is not repeated here.** This guide covers what sits underneath: the
three-component architecture, inputs, streams, pipelines, index sets and disk sizing,
health checks, backup, upgrades, troubleshooting, security and rebuild.

Audience: the sysadmin who has to keep Graylog running, not the helpdesk person who
searches it. The helpdesk-level unban workflow lives in the fail2ban PDFs listed in
References.

## Quick Facts

| Field | Value |
|---|---|
| Owner | IT lead (it@example.com) |
| Environment | prod |
| Host | `graylog01.example.com` (zimbra guide, monitoring guide, Sidecar config, IPAM reference, DNS guide). Older KB runbooks and the fail2ban PDFs use `https://graylog.example.com:7555`; the triage toolkit config uses `https://graylog.example.com` with no port. [CONFIRM: proposal — `graylog01.example.com` is the host, `graylog.example.com` is a DNS alias for it, and 7555 is a published UI port in front of Graylog's own `:9000`. Check with `dig +short graylog.example.com` and `ss -lntp` on the host, then fix the other docs. This is an open item in INDEX.md.] |
| Web UI | `https://graylog.example.com:7555` per the unban runbooks — [CONFIRM: still current] |
| API | `http://graylog01.example.com:9000/api/` — this is the `server_url` the Sidecar procedure in `Security & Hardening/fresh-box-hardening-cheatsheet.txt` uses. [CONFIRM: whether 9000 is reachable from other hosts or only locally / via proxy] |
| Deployment type | [CONFIRM: package install (`graylog-server`, `opensearch`, `mongod` systemd units) or Docker Compose? The Docker diagnostics runbook attaches to a container named `graylog` on network `graylog_default`, and the hardening cheatsheet scans image `graylog/graylog:6.0`, which both suggest Compose. Commands below are given for the package layout with Docker equivalents where they differ.] |
| Version | [FILL IN: Graylog version from System -> Overview, or `curl -s localhost:9000/api/ \| jq .version`. The `graylog/graylog:6.0` image tag above is a hint, not proof] |
| Search backend | [FILL IN: OpenSearch or Elasticsearch, version, and whether it runs on graylog01 or a separate host. `curl -s localhost:9200 \| jq .version`] |
| Config store | MongoDB — [FILL IN: version, and whether local to graylog01] |
| Access | Web UI over HTTPS. Admin credentials in 1Password / IT vault ("Admin login's in 1pass" — fail2ban KB). SSH as `orgadmin`. [FILL IN: management VLAN / WireGuard requirement for SSH and UI] |
| Dependencies | MongoDB, OpenSearch/Elasticsearch, DNS, NTP, the pfSense path from every source VLAN to the input ports, mail.example.com for email notifications, Proxmox (if graylog01 is a guest — [CONFIRM], see `Linux & Servers/proxmox-cluster-administration-guide.md`) |
| Dependents | fail2ban unban workflow; mail troubleshooting; Snort review; triage toolkit scripts (`graylog_hunt.sh`, `log_coverage.sh`); certificate-expiry and audit pipelines planned in the PKI and hardening docs; incident timelines in the ransomware playbook |
| Backup tier | Tier 3 — supporting (`Hardware & Backup/backup-restore-testing-procedure.md`) |
| Last reviewed | NEVER (created 2026-10-04) |

## How It Works

Graylog is three services. Knowing which one is broken is most of troubleshooting.

    log source (rsyslog, pfSense, Sidecar+Filebeat/Winlogbeat, GELF app)
      -> Graylog input (UDP/TCP listener)       # graylog-server, Java
      -> on-disk journal                         # /var/lib/graylog-server/journal — buffer
      -> process buffer: extractors, stream rules, pipeline rules
      -> output buffer -> OpenSearch index set   # graylog_* indices, via deflector alias
      -> search / dashboards / event definitions -> notifications

    MongoDB holds: users, roles, streams, pipelines, dashboards, inputs, index-set
    definitions, event definitions, Sidecar configs. No log messages.
    OpenSearch holds: the log messages. Nothing else Graylog cannot rebuild.

| Component | Role | Holds | If it is down |
|---|---|---|---|
| graylog-server | Inputs, processing, web UI, REST API | Journal on local disk | No ingestion, no UI. Senders using UDP lose messages; TCP/Beats senders back off and retry |
| OpenSearch / Elasticsearch | Message storage and search | All indexed logs | Ingestion continues into the journal until the journal fills; searches fail |
| MongoDB | Configuration database | All configuration | Graylog will not start; running node degrades |

Key consequence: **MongoDB is the thing to back up.** Indexed logs can be lost and
accepted (they are Tier 3 evidence), but rebuilding streams, pipelines, dashboards, the
Overview/Fail2Ban tab and the Sidecar configs from memory is days of work.

Default file locations (package install):

    /etc/graylog/server/server.conf                # Graylog node config
    /etc/graylog/server/node-id                    # node identity — keep with backups
    /etc/default/graylog-server                    # JVM heap (GRAYLOG_SERVER_JAVA_OPTS)
    /var/lib/graylog-server/journal/               # message journal
    /var/log/graylog-server/server.log             # Graylog's own log
    /etc/opensearch/opensearch.yml                 # search backend config
    /etc/opensearch/jvm.options.d/                 # search backend heap
    /var/lib/opensearch/                           # index data — [FILL IN: confirm path and the filesystem it lives on]
    /etc/mongod.conf  /var/lib/mongodb/            # config store

[FILL IN: if Compose, the path of the compose file and the named volumes that map to the
three data directories above — `docker compose -f <path> config` lists them.]

---

## Inputs and Log Sources

### Input inventory

An input is one protocol on one port. Add sources to existing inputs; create a new input
only for a new protocol. Inputs run as a non-root Java process, so ports below 1024 are
normally not used directly — 1514 is the conventional syslog port.

| Input | Protocol / port | Senders | Status |
|---|---|---|---|
| Syslog | UDP [FILL IN: port] | mail.example.com (rsyslog: `/var/log/zimbra.log`, fail2ban) | In use — the fail2ban search `application_name:fail2ban.*` only works because a Syslog input parses `application_name` |
| Syslog | TCP [FILL IN: port] | [FILL IN] | [CONFIRM: exists] |
| Beats | TCP 5044 | Sidecar-managed Filebeat, tag `audit` (`/var/log/org-audit/*.json`, `/var/log/audit/audit.log`) | Documented in `Security & Hardening/fresh-box-hardening-cheatsheet.txt` STEP B — [CONFIRM: deployed] |
| Snort JSON | [FILL IN: input type and port] | pfSense / Snort 3 `alert_json` | [CONFIRM: deployed — `Security Procedures/modules/detection-validation.md` presents it as a worked example] |
| GELF | UDP/TCP 12201 (default) | [FILL IN: any application sending GELF, e.g. Docker `--log-driver gelf`] | [CONFIRM: exists] |

Enumerate the real list from the API rather than the UI when filling this in:

    curl -s -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://127.0.0.1:9000/api/system/inputs | jq -r '.inputs[] | [.id,.title,.type,.attributes.port] | @tsv'   # id, title, type, port
    sudo ss -lnup | grep java                      # UDP ports Graylog is actually bound to
    sudo ss -lntp | grep java                      # TCP ports Graylog is actually bound to

`~/.graylog_token` is a personal, read-only API token (your user -> Edit Tokens), mode 600.
Never paste a token on the command line; it lands in shell history. The token is the
username and the literal word `token` is the password.

### Log sources in this environment

| Source | Mechanism | What arrives | Status |
|---|---|---|---|
| mail.example.com | rsyslog forward of `/var/log/zimbra.log`; fail2ban logs via syslog | Postfix, saslauthd, Zimbra, fail2ban Ban/Unban actions for jails `zimbra-domain-alias-auth-failures`, `zimbra-submission`, `zimbra-web`, `zimbra-bad-emails`, `zimbra-bad-emails-smtp` | In use. [FILL IN: are `/opt/zimbra/log/mailbox.log` and `audit.log` also shipped? Three of the five jails read those files, so without them Graylog shows the ban but not the triggering login] |
| pfSense | Status -> System Logs -> Settings -> Remote Logging | Firewall, system, DHCP | [FILL IN: configured? `Networking Guide/firewall-change-procedure.md` records this as unknown] |
| Snort on pfSense | `alert_json` -> Graylog, `parse_json` pipeline with `snort_` prefix | `snort_sid`, `snort_class`, `snort_priority`, `snort_src_addr`, `snort_dst_addr` | [CONFIRM: deployed] |
| Ubuntu servers | Graylog Sidecar + Filebeat, per `Linux & Servers/server-build-standard.md` section 10 | auth.log, syslog, auditd, `/var/log/org-audit/` JSON | [FILL IN: which of db01, db02, sign, appdb2, appdb1, erpdb, prod01, files, forums actually have Sidecar Active] |
| Proxmox nodes | rsyslog forward | PVE task log, corosync | [FILL IN — proxmox guide records this as unknown] |
| AD / Windows (auth2, auth4, other DCs) | Sidecar + Winlogbeat (preferred) or NXLog | Security log | [FILL IN: not deployed as far as the library shows — endpoint-security and ransomware docs both mark it unconfirmed] |
| Synology (files.example.com) | DSM Log Center -> Log Sending | DSM system/connection logs | [FILL IN] |
| UPS, switches, wireless controller | syslog | device events | [FILL IN] |

### Adding a Linux source (rsyslog, no agent)

Use this for appliances and for hosts where the Sidecar is not appropriate. For general
servers, use the Sidecar procedure in `fresh-box-hardening-cheatsheet.txt` instead.

    echo '*.* @@graylog01.example.com:[FILL IN: TCP syslog port];RSYSLOG_SyslogProtocol23Format' | sudo tee /etc/rsyslog.d/90-graylog.conf   # @@ = TCP, RFC 5424 format
    sudo rsyslogd -N1                              # validate rsyslog config before restarting
    sudo systemctl restart rsyslog                 # apply
    logger -t graylog-test "hello from $(hostname) $(date -Is)"   # emits a test line via the local syslog
    # then search in Graylog: source:<hostname> AND graylog-test   (last 5 minutes)

[CONFIRM: proposal — TCP for anything security-relevant (fail2ban, auth). UDP silently
drops under load and when Graylog restarts, which is exactly when you need the logs.]

### Adding a Windows / AD source (Sidecar + Winlogbeat)

    # PowerShell, elevated, on the DC. Token comes from System -> Sidecars -> Create API token (store in IT vault)
    .\graylog_sidecar_installer_<version>.exe /S -SERVERURL=http://graylog01.example.com:9000/api -TAGS="[\"windows\",\"ad\"]"   # silent install; add -APITOKEN at the prompt from the vault, never in a saved script
    & "C:\Program Files\Graylog\sidecar\graylog-sidecar.exe" -service install   # register service
    & "C:\Program Files\Graylog\sidecar\graylog-sidecar.exe" -service start     # start
    Get-Service graylog-sidecar                    # expect Running

Then in Graylog: System -> Sidecars -> Configuration, create a Winlogbeat config for tag
`ad` that ships the Security log to the Beats input. Event IDs worth having from DCs on
day one: 4624/4625 (logon success/failure), 4740 (lockout), 4720 (user created),
4728/4732/4756 (added to privileged group), 1102 (audit log cleared).
[CONFIRM: proposal — filter to these IDs initially to bound index growth; widen later.]

NXLog Community Edition is the alternative where Sidecar is not wanted; it ships GELF to
a GELF TCP input. [CONFIRM: proposal — standardise on Sidecar so collector config is
central and versioned in Graylog, matching the Linux standard.]

---

## Streams

Streams route messages. Event definitions, dashboard widgets, permissions and index sets
all attach to streams, so a class of message you want to alert on needs its own stream.

| Stream | Rule | Index set | Status |
|---|---|---|---|
| All messages (default) | everything not removed by another stream | Default index set | Built in |
| [FILL IN: mail stream name — zimbra guide also asks] | `source` matches `mail` | [FILL IN] | [FILL IN] |
| Fail2Ban | [CONFIRM: proposal — `application_name` matches regex `^fail2ban`] | [FILL IN] | [FILL IN: does it exist, or is the Overview Fail2Ban tab a saved query?] |
| Org Audit | pipeline `route_to_stream(name: "Org Audit", remove_from_default: true)` | [CONFIRM: proposal — own index set] | Documented in hardening cheatsheet STEP E — [CONFIRM: deployed] |
| Snort | pipeline after `parse_json` | [FILL IN] | [CONFIRM: deployed] |
| [CONFIRM: proposal] pfSense | `source` = pfSense hostname | Security index set | Not built |
| [CONFIRM: proposal] Windows-AD | `beats_type` / tag `ad` | Security index set | Not built |
| [CONFIRM: proposal] Graylog-Self | `source:graylog01*` | Default | Not built — needed for the watchdog |

Dump what actually exists:

    curl -s -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://127.0.0.1:9000/api/streams | jq -r '.streams[] | [.id,.title,.index_set_id,.disabled] | @tsv'   # id, title, index set, disabled?

Rules of thumb:

- Tick **"Remove matches from 'All messages'"** when a stream has its own index set,
  otherwise every message is stored twice.
- Test rules with the stream rule tester against a real message before saving.
- A paused stream drops routing silently. `"disabled": true` above is a finding.

---

## Pipelines and Extractors

### Processing order — check this first

System -> Configurations -> Message Processors. The **Message Filter Chain** (stream
routing, extractors) must run **before** the **Pipeline Processor** if pipelines are
connected to anything other than "All messages". Otherwise pipeline rules connected to a
stream never see messages. [FILL IN: record the current order here.]

### Pipelines in use or documented

Snort (from `Security Procedures/modules/detection-validation.md`): `alert_json` lines are
parsed with `parse_json` and flattened with a `snort_` prefix. The most common failure is
field-name drift (`snort_class` vs `snort_classification`) — see that doc's section 4.

Audit (from `fresh-box-hardening-cheatsheet.txt` STEP E — reproduced only as the
reference shape; the cheatsheet is authoritative):

    rule "parse org audit json"
    when
      has_field("source_type") && to_string($message.source_type) == "audit"
    then
      let j = parse_json(to_string($message.message));
      set_fields(fields: to_map(j), prefix: "audit_");
      route_to_stream(name: "Org Audit", remove_from_default: true);
    end

### Proposed: parse fail2ban actions into fields

Today the unban workflow is a full-text search for an IP. Parsing jail and IP into fields
makes the Fail2Ban tab, dashboards and alerts precise. A fail2ban action line looks like
`fail2ban.actions [1234]: NOTICE [zimbra-web] Ban 203.0.113.45`.

    rule "fail2ban: extract jail, action, ip"
    when
      starts_with(to_string($message.application_name), "fail2ban")
    then
      let m = grok(pattern: "%{DATA}\\[%{DATA:f2b_jail}\\] %{WORD:f2b_action} %{IP:f2b_ip}", value: to_string($message.message), only_named_captures: true);
      set_fields(m);
    end

[CONFIRM: proposal — field names `f2b_jail`, `f2b_action`, `f2b_ip`. Check the actual
message format in the detail pane first; the rule matches nothing if the prefix differs.]
Use the pipeline simulator (System -> Pipelines -> Simulator) with a real message before
connecting the pipeline. Once deployed, update the unban PDFs' search to
`f2b_ip:203.0.113.45`.

### Extractors

Extractors are the legacy per-input parser. They still work but are slower and harder to
audit than pipelines. Inventory them before any upgrade:

    curl -s -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://127.0.0.1:9000/api/system/inputs/<input-id>/extractors | jq -r '.extractors[] | [.title,.type,.target_field] | @tsv'   # per input

[FILL IN: list of extractors per input.] [CONFIRM: proposal — no new extractors; new
parsing goes in pipelines; migrate existing ones opportunistically.]

Pipeline performance: a rule with an unanchored regex or a heavy grok on every message is
the usual cause of a growing journal with a healthy backend. Guard rules with a cheap
`when` clause (as above) so the expensive function only runs on relevant messages.

---

## Index Sets, Retention and Disk Sizing

Retention belongs to the index set. Graylog writes through an alias (`<prefix>_deflector`)
to the newest index, rotates to a new index on a size/count/time trigger, and deletes or
closes the oldest once the configured index count is exceeded.

### Current settings

| Index set | Prefix | Rotation | Max indices | Retention action | Shards / replicas | Worst-case size |
|---|---|---|---|---|---|---|
| Default | `graylog` | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN: others] | | | | | | |

    curl -s -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://127.0.0.1:9000/api/system/indices/index_sets | jq -r '.index_sets[] | [.title,.index_prefix,.rotation_strategy_class,.retention_strategy.max_number_of_indices,.shards,.replicas] | @tsv'   # fill the table from this
    curl -s 'localhost:9200/_cat/indices?v&s=index&h=index,health,docs.count,store.size'   # actual size per index
    df -h /var/lib/opensearch                      # filesystem holding the indices — [CONFIRM path]

### Proposed index sets and retention

| Index set | Streams | Rotation | Retention | Reason |
|---|---|---|---|---|
| Default | All messages, Graylog-Self, Linux syslog | [CONFIRM: proposal — P1D (daily)] | [CONFIRM: proposal — 30 indices = 30 days, delete] | Operational noise; rarely needed beyond a month |
| Mail & fail2ban | Mail, Fail2Ban | [CONFIRM: proposal — P1D] | [CONFIRM: proposal — 90 indices = 90 days, delete] | Unban and compromise investigations routinely go back 7–30 days; 90 covers a late report (matches the monitoring guide's 90-day proposal) |
| Security | Snort, pfSense, Windows-AD | [CONFIRM: proposal — P1D] | [CONFIRM: proposal — 180 days, delete] | Breach discovery is often months late; the ransomware playbook depends on a timeline |
| Org Audit | Org Audit | [CONFIRM: proposal — P1W] | [CONFIRM: proposal — 52 indices = 1 year] | Low volume; trend data (Lynis hardening index, cert expiry) is only useful over long windows |

[FILL IN: any organizational, regulatory or legal retention requirement for authentication or mail
logs — this overrides the proposals above.]

Use time-based rotation (P1D) so "N indices" means "N days". Size-based rotation makes
retention depend on traffic, which means a log flood quietly shortens retention.

Replicas: on a single-node backend set **replicas = 0**. A replica that can never be
allocated leaves the cluster permanently yellow and hides real problems.

### Disk sizing

    required_disk = daily_ingest_GB x retention_days x (1 + replicas) x 1.25   # 1.25 = index overhead + headroom for merges

Keep `required_disk` under **75%** of the filesystem. OpenSearch's default disk
watermarks are 85% (stop allocating new shards), 90% (move shards away) and 95%
(flood stage: every index goes read-only and ingestion stops). On a single node, 85% is
effectively the ceiling.

    curl -s 'localhost:9200/_cat/indices/graylog_*?h=index,creation.date.string,store.size&s=index' | tail -n 8   # last week's daily index sizes = daily ingest
    curl -s 'localhost:9200/_cat/allocation?v'     # disk used/available as OpenSearch sees it
    curl -s 'localhost:9200/_cluster/settings?include_defaults=true' | jq '.defaults.cluster.routing.allocation.disk'   # current watermarks

| Measurement | Value |
|---|---|
| Daily ingest, all index sets | [FILL IN: GB/day from the commands above, averaged over 7 days] |
| Index filesystem size | [FILL IN] |
| Required disk at proposed retention | [FILL IN: computed] |
| Journal filesystem size and `message_journal_max_size` | [FILL IN] |

Change retention in the UI (System -> Indices -> edit index set). Never delete index
directories from disk; Graylog's index ranges in MongoDB will then point at data that
does not exist. After changing an index set, rotate so the new settings apply:
System -> Indices -> index set -> Maintenance -> Rotate active write index.

---

## Dashboards and Saved Searches

**The Overview dashboard has a Fail2Ban tab. That tab — not the separate dashboard also
named "Fail2ban" — is the unban investigation view.** Both fail2ban PDFs are emphatic
about this ("DO NOT GO TO FAIL2BAN OPTION... totally different view"). Do not rename,
merge or delete either without updating
`[internal KB: unban a user via Graylog]`,
`[internal KB: fail2ban instructions]` and the fail2ban KB PDF.

[FILL IN: what the separate "Fail2ban" dashboard shows and whether anyone uses it. If
nobody does, rename it "Fail2ban (legacy — use Overview tab)" rather than deleting it.]

Searches already in use (from the KB and PDFs):

    application_name:fail2ban.* AND "203.0.113.45"     # was this IP banned, and by which jail — widen range to 7–30 days
    "user@example.com"                                       # a user's mail activity (step 2 of the internal fail2ban method)
    source:mail.example.com AND "authentication failed"      # failed logins that feed the zimbra-* jails

Mail-specific searches are in `Mail & Messaging/zimbra-mail-administration-guide.md`
("Graylog queries for mail troubleshooting"); Snort searches in
`Security Procedures/modules/detection-validation.md`.

Proposed: a **Graylog Health** dashboard — message count per source over 24h (a flat
line is a silent sender), messages per stream, and `level:<=3` errors from graylog01
itself. [CONFIRM: proposal.]

Export dashboards and streams as a content pack (System -> Content Packs -> Create) after
any significant change. Store the JSON with the config backup. It is the fastest way to
rebuild the Overview/Fail2Ban tab on a new node.

---

## Alerts: Event Definitions and Notifications

Alerts -> Event Definitions. An event definition is a search (filter) plus optional
aggregation, evaluated on a schedule, which fires notifications. Graylog has nothing
defined that reaches a human today ([FILL IN: confirm — Alerts -> Event Definitions]).

Notification channel. Email requires the transport settings in `server.conf`:

| Setting | Value | Reason |
|---|---|---|
| `transport_email_enabled` | `true` | Required for email notifications |
| `transport_email_hostname` | [CONFIRM: proposal — `mail.example.com`] | Internal relay |
| `transport_email_port` | [CONFIRM: proposal — 587] | Submission with STARTTLS |
| `transport_email_use_tls` | `true` | Do not send alert content in clear |
| `transport_email_auth_username` / `_password` | [FILL IN: relay account — password in IT vault, not here] | Only if the relay requires auth |
| `transport_email_from_email` | [CONFIRM: proposal — `graylog@example.com`] | Recognisable sender for mail rules |
| `transport_email_web_interface_url` | [CONFIRM: the canonical UI URL] | Makes links in alerts clickable |

Circular dependency: if mail.example.com is down, mail-delivered alerts about mail.example.com are
not delivered. Add a second channel (HTTP notification to a chat webhook or SMS gateway)
for P1 events. [FILL IN: second channel, per the routing table in monitoring-alerting-guide.md.]

Proposed starter event definitions, in build order (all [CONFIRM: proposal]):

| # | Event | Stream / filter | Condition | Priority | Reason |
|---|---|---|---|---|---|
| 1 | Source silent | `source:mail.example.com` (one per critical source) | count() < 1 over 60 min, run every 15 min | High | Every other alert is meaningless if logs stop |
| 2 | Break-glass account used | Windows-AD, account names [FILL IN] | count() >= 1 | High | Highest-value alert per `secrets-management-guide.md` |
| 3 | fail2ban ban spike | Fail2Ban, `f2b_action:Ban` | count() > [FILL IN after baseline] per hour | Normal | Credential-stuffing campaign or the organization NAT address being banned |
| 4 | Snort priority 1 | Snort, `snort_priority:1` | count() >= 1 | High | Validated per detection-validation.md before enabling |
| 5 | Firewall config change | pfSense, config-change message [FILL IN pattern] | count() >= 1 | Normal | Proposed in firewall-change-procedure.md |
| 6 | Cert expiring | Org Audit, days-remaining field | < 30 | Normal | Per certificate-pki-lifecycle-guide.md step 3 |

Set thresholds only after two weeks of baseline, and follow the muting rule in the
monitoring guide (every mute has an expiry and a reason).

Test every notification: Alerts -> Notifications -> Edit -> **Execute Test Notification**.
An untested notification is a rumour.

---

## Users, Roles and LDAP Authentication

| Item | Current | Proposed |
|---|---|---|
| Local admin | One admin account, password in 1Password | Keep as break-glass only; do not use daily [CONFIRM] |
| Directory auth | [FILL IN: System -> Authentication — is an AD service configured?] | Active Directory service over LDAPS to `auth2.example.com:636` / `auth4.example.com:636` [CONFIRM] |
| Bind account | [FILL IN] | Dedicated read-only service account, password in IT vault. Connection details and the cert gotchas are in `Active Directory/ldap-connection-reference.md` |
| Default role for directory users | — | `Reader` [CONFIRM] |
| Stream access | [FILL IN] | Share streams/dashboards per user or team; do not grant global Reader to non-IT staff [CONFIRM] |
| API tokens | Sidecar token; triage toolkit tokens | One token per purpose, on a user with the minimum role; recorded in the IT vault inventory per `secrets-management-guide.md` |

Notes:

- Group-to-role synchronisation from AD is a Graylog Enterprise feature. On Graylog Open,
  directory users get the default role and are then granted access individually.
  [CONFIRM: edition — Open or Enterprise.]
- The LDAPS certificate on the DCs is replaced on its own cycle
  (`Security & Hardening/LDAPs certificate replacement procedure.txt`). When it changes,
  Graylog directory logins can fail; the local admin still works. Add graylog01 to the
  "consumers of LDAPS" list in `ldap-connection-reference.md`.
- Offboarding: disabling the AD account blocks directory logins, but **API tokens survive**.
  Revoke a departing person's tokens under System -> Users -> (user) -> Edit Tokens.

---

## Operations (Day-2)

### Start / stop / restart

Order matters: MongoDB, then OpenSearch, then Graylog to start; reverse to stop.

    sudo systemctl start mongod                    # 1. config store first
    sudo systemctl start opensearch                # 2. search backend — wait for status yellow/green
    curl -s 'localhost:9200/_cluster/health?wait_for_status=yellow&timeout=120s' | jq .status   # blocks until the backend is usable
    sudo systemctl start graylog-server            # 3. Graylog last
    sudo journalctl -u graylog-server -f           # watch for "Graylog server up and running"

Docker equivalent: `docker compose up -d` brings all three up; `docker compose ps` and
`docker compose logs -f graylog` replace the above. [CONFIRM: deployment type.]

### Health check (run daily, or when anything looks odd)

    curl -s http://127.0.0.1:9000/api/system/lbstatus   # expect: ALIVE (no auth needed)
    curl -s -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://127.0.0.1:9000/api/system | jq '{version,lifecycle,is_processing,lb_status,timezone}'   # expect lifecycle "running", is_processing true
    curl -s -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://127.0.0.1:9000/api/system/journal | jq '{uncommitted_journal_entries,append_events_per_second,read_events_per_second,journal_size,journal_size_limit}'   # uncommitted should be near 0; read >= append
    curl -s -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://127.0.0.1:9000/api/system/throughput | jq .throughput   # messages/sec now; 0 is a problem
    curl -s -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://127.0.0.1:9000/api/system/indexer/cluster/health | jq .   # Graylog's view of the backend
    curl -s -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://127.0.0.1:9000/api/system/notifications | jq -r '.notifications[] | [.type,.severity,.timestamp] | @tsv'   # system warnings shown in the UI bell
    curl -s 'localhost:9200/_cluster/health?pretty' # expect green (yellow acceptable only if replicas > 0 on one node — fix that instead)
    curl -s 'localhost:9200/_cat/shards?v' | grep -v STARTED   # anything listed is unassigned/initialising
    df -h /var/lib/opensearch /var/lib/graylog-server   # index and journal headroom
    mongosh --quiet --eval 'db.adminCommand({ping:1}).ok'   # expect 1
    chronyc tracking | grep -E 'System time|Leap'   # clock offset and sync status

What "normal" looks like: [FILL IN: baseline messages/sec, daily ingest GB, typical
journal uncommitted count, disk % — record after a week of observation.]

### Logs (Graylog's own)

    sudo tail -f /var/log/graylog-server/server.log            # Graylog
    sudo journalctl -u opensearch --since "1 hour ago"          # search backend
    sudo tail -n 100 /var/log/opensearch/graylog.log            # backend log, named after cluster.name — [CONFIRM cluster.name]
    sudo journalctl -u mongod --since "1 hour ago"              # config store

### Key configuration

| File / setting | Value | Reason |
|---|---|---|
| `server.conf` `is_leader` | `true` | Single node; leader runs periodic jobs (retention, index ranges) |
| `server.conf` `http_bind_address` | [FILL IN — default `127.0.0.1:9000`] | Must be reachable by Sidecars (`server_url` uses `graylog01.example.com:9000`) |
| `server.conf` `http_publish_uri` / `http_external_uri` | [FILL IN] | Must match the URL users and Sidecars use, or UI links and Sidecar registration break; ties to the hostname [CONFIRM] |
| `server.conf` `elasticsearch_hosts` | [FILL IN — expect `http://127.0.0.1:9200`] | Backend location |
| `server.conf` `mongodb_uri` | [FILL IN — expect `mongodb://localhost/graylog`] | Config store location; credentials, if any, in IT vault |
| `server.conf` `password_secret` | in IT vault — never in this doc | Encrypts stored secrets in MongoDB; **must be identical on a restored node** |
| `server.conf` `root_password_sha2` | in IT vault — never in this doc | Built-in admin |
| `server.conf` `root_timezone` | [CONFIRM: proposal — `<Region/City>`] | Display timezone for the admin; storage is always UTC |
| `server.conf` `message_journal_max_size` | [FILL IN — default 5gb] | Buffer while the backend is down; size it to [CONFIRM: proposal — 12h of peak ingest] |
| `/etc/default/graylog-server` heap | [FILL IN] | Graylog heap; 1–2 GB is typical in this environment's scale |
| `opensearch.yml` `cluster.name` | `graylog` [CONFIRM] | Name used in log filenames |
| `opensearch.yml` `discovery.type` | `single-node` [CONFIRM] | Single node; no cluster formation |
| `opensearch.yml` `network.host` | `127.0.0.1` [CONFIRM] | Backend has no auth when the security plugin is disabled; never expose 9200 |
| `opensearch.yml` `action.auto_create_index` | `false` | Graylog manages index templates; auto-create produces wrong mappings |
| `opensearch.yml` `path.repo` | [CONFIRM: proposal — snapshot directory] | Required for snapshots (see Backup) |
| OpenSearch heap (`jvm.options.d/heap.options`) | [FILL IN] | ~50% of RAM, max 31g, `-Xms` = `-Xmx` |
| `mongod.conf` `net.bindIp` | `127.0.0.1` [CONFIRM] | MongoDB must not be network-reachable |

### "Is Graylog itself alive?" — the watchdog

Graylog cannot alert on its own death. The check must run **somewhere else** and alert
through a path that does not depend on Graylog. Nothing does this today.

[CONFIRM: proposal — run from cron every 10 minutes on a different host (not graylog01,
and ideally not mail.example.com, since that is Graylog's busiest source), alert by mail and
the second channel.] The check, in increasing order of value:

    curl -sf --max-time 10 http://graylog01.example.com:9000/api/system/lbstatus | grep -q ALIVE || echo "graylog API down"   # 1. process up
    curl -sf --max-time 10 -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' 'http://graylog01.example.com:9000/api/search/universal/relative?query=source%3Amail.example.com&range=900&limit=1&fields=timestamp' | jq -e '.total_results > 0' >/dev/null || echo "no mail.example.com logs in 15 min"   # 2. logs actually arriving
    curl -sf --max-time 10 -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://graylog01.example.com:9000/api/system/journal | jq -e '.uncommitted_journal_entries < 100000' >/dev/null || echo "journal backing up"   # 3. not silently buffering

Check 2 is the one that matters: Graylog can be ALIVE and indexing nothing. The legacy
`/api/search/universal/relative` endpoint is used because it is a simple GET; if it
returns 404 on the installed version, switch to `GRAYLOG_API=export` as
`graylog_hunt.sh` does. For per-source coverage across every expected host, use
`Security Procedures/security-toolkit-review/curated/log_coverage.sh --quiet` from cron with
a `~/.triage/sources.txt` list. [FILL IN: the watchdog host and cron entry once deployed.]

### Backup

| What | How | Where | Schedule | Reason |
|---|---|---|---|---|
| MongoDB (all configuration) | `mongodump` | [FILL IN: backup target] | [CONFIRM: proposal — daily] | Irreplaceable; small (MBs) |
| `server.conf`, `node-id`, `/etc/default/graylog-server`, `opensearch.yml`, `jvm.options.d/`, `mongod.conf`, compose file if any | tar | same | daily, and before every change | Needed to rebuild identically |
| `password_secret` | IT vault entry | IT vault | on change | Without it, restored inputs/notifications with stored secrets are unreadable |
| Content pack export | UI export | with config backup | after significant change | Fast partial rebuild |
| Indexed messages | Rebuildable: no; acceptable loss: yes. Optionally an OpenSearch snapshot | [CONFIRM: proposal — snapshots only for the Security index set, weekly] | | Tier 3. Logs needed for an incident are exported at the time, per the ransomware playbook |
| Whole VM | Proxmox `vzdump` | [FILL IN] | [FILL IN] | Fastest full restore, if graylog01 is a guest [CONFIRM] |

    sudo install -d -m 700 /var/backups/graylog                                         # local staging dir
    sudo mongodump --db graylog --gzip --archive=/var/backups/graylog/mongo-$(date +%F).gz   # config database dump
    sudo tar czf /var/backups/graylog/etc-$(date +%F).tgz /etc/graylog /etc/default/graylog-server /etc/opensearch /etc/mongod.conf   # config files
    ls -lh /var/backups/graylog/                                                        # confirm non-zero sizes

The tarball contains `server.conf`, which contains `password_secret` and the admin hash.
Treat it as a secret: restrict permissions and do not copy it to a share staff can read.

Optional OpenSearch snapshot (requires `path.repo` set and a restart):

    curl -s -XPUT localhost:9200/_snapshot/org_fs -H 'Content-Type: application/json' -d '{"type":"fs","settings":{"location":"[CONFIRM: path.repo directory]"}}'   # register repository once
    curl -s -XPUT "localhost:9200/_snapshot/org_fs/snap-$(date +%F)?wait_for_completion=true" -H 'Content-Type: application/json' -d '{"indices":"[FILL IN: security index prefix]_*","include_global_state":false}'   # snapshot one index set
    curl -s 'localhost:9200/_snapshot/org_fs/_all' | jq -r '.snapshots[] | [.snapshot,.state] | @tsv'   # list snapshots

Restore test: [FILL IN: date of last tested restore]. Test by restoring the Mongo dump into
a scratch VM with the same `password_secret`, starting Graylog, and confirming the Overview
dashboard's Fail2Ban tab is present. Record per `backup-restore-testing-procedure.md`.

### Upgrades

Order: **MongoDB -> OpenSearch -> Graylog**, one component at a time, with Graylog
stopped while its dependencies change. Before anything:

1. Read the Graylog upgrade notes and compatibility matrix for the *target* version
   (go2docs.graylog.org -> Upgrading). Confirm the target Graylog supports the MongoDB and
   OpenSearch versions you will end up on. [FILL IN: current versions and target.]
2. MongoDB major versions are upgraded **one major at a time** (e.g. 5.0 -> 6.0 -> 7.0),
   setting featureCompatibilityVersion after each step.
3. Snapshot the VM and take the Mongo + config backup above.
4. Log the change in `SysAdmin Procedures/Change_Management_Log.txt`.

Steps (package install):

    apt-mark showhold                              # see what is pinned
    sudo systemctl stop graylog-server             # stop ingestion processing; senders buffer or drop (UDP)
    sudo mongodump --db graylog --gzip --archive=/var/backups/graylog/mongo-preupgrade-$(date +%F).gz   # last-chance backup
    mongosh --quiet --eval 'db.adminCommand({getParameter:1, featureCompatibilityVersion:1})'   # record current FCV
    sudo apt-mark unhold mongodb-org && sudo apt-get install -y mongodb-org   # after switching the apt repo to the next major only
    sudo systemctl restart mongod && mongosh --quiet --eval 'db.version()'   # confirm new version
    mongosh --quiet --eval 'db.adminCommand({setFeatureCompatibilityVersion: "[FILL IN: new major, e.g. 7.0]", confirm: true})'   # finalise this step; omit confirm on MongoDB < 7
    sudo apt-mark unhold opensearch && sudo apt-get install -y opensearch=[FILL IN: version]   # search backend
    sudo systemctl restart opensearch && curl -s 'localhost:9200/_cluster/health?wait_for_status=yellow&timeout=180s' | jq .status   # wait for health
    sudo apt-mark unhold graylog-server && sudo apt-get install -y graylog-server=[FILL IN: version]   # Graylog last; keep the existing server.conf when dpkg asks
    sudo systemctl start graylog-server && sudo journalctl -u graylog-server -f   # watch migrations complete
    sudo apt-mark hold mongodb-org opensearch graylog-server   # prevent accidental upgrades by unattended-upgrades or apt upgrade

Docker: change one image tag at a time in the compose file, `docker compose pull <svc>`,
`docker compose up -d <svc>`, in the same order. [CONFIRM: deployment type.]

After upgrading: run the health check, confirm inputs show throughput, confirm the
Fail2Ban tab and a known unban search still work, confirm Sidecars are Active
(Sidecar and collector versions may need upgrading separately).

Rollback: Graylog database migrations are one-way. Rollback is "restore the VM snapshot",
not "downgrade the package".

---

## Troubleshooting

### Symptom: no messages arriving (from one source or all)

- Check whether the input sees traffic: System -> Inputs, throughput counter. Zero means
  network or sender; non-zero means routing or search range.
- Check on graylog01 that the port is bound, and capture to prove packets arrive:

      sudo ss -lnup | grep java                                # UDP inputs bound
      sudo tcpdump -ni any -c 20 port [FILL IN: input port]   # packets reaching the host at all

- Check on the sender: `logger` test (see "Adding a Linux source"), `systemctl status
  rsyslog` or `systemctl status graylog-sidecar`, and for Sidecars
  `journalctl -u graylog-sidecar --since "1 hour ago"`.
- Check the path: pfSense rules from the source VLAN to graylog01, and `sudo ufw status`
  on graylog01.
- Fix: restart the input (UI), fix the sender, open the firewall path. If all inputs show
  zero after a reboot, check `http_bind_address` and that graylog-server actually started.
- If everything arrives but nothing is searchable, see the next three symptoms.

### Symptom: journal growing / "uncommitted messages" / UI shows journal warning

- Likely cause: the backend is down, read-only or slow; or a pipeline rule is too slow.
- Check:

      curl -s -u "$(cat ~/.graylog_token):token" -H 'Accept: application/json' http://127.0.0.1:9000/api/system/journal | jq '{uncommitted_journal_entries,append_events_per_second,read_events_per_second}'   # read < append = falling behind
      curl -s 'localhost:9200/_cluster/health?pretty'          # backend status
      sudo grep -iE 'index.*(error|fail)|blocked' /var/log/graylog-server/server.log | tail -n 20   # indexing failures

- Fix: fix the backend first; the journal drains by itself. If the backend is healthy and
  the process buffer is full (System -> Nodes -> Details), disable recently changed
  pipeline rules one at a time. Do **not** delete the journal directory unless you accept
  losing everything in it; if you must, stop graylog-server first.

### Symptom: journal full / disk full on graylog01

- Check: `df -h /var/lib/graylog-server /var/lib/opensearch` and
  `sudo du -sh /var/lib/graylog-server/journal`.
- Fix: when the journal reaches `message_journal_max_size`, Graylog discards the oldest
  journal segments — that is data loss, not an outage. Restore indexing first (previous
  symptom), then free index disk via retention (next symptom). Do not raise
  `message_journal_max_size` on a full disk.

### Symptom: indices read-only (flood-stage disk watermark)

- Graylog log shows `cluster_block_exception` / `FORBIDDEN/12/index read-only / allow delete`.
- Check:

      curl -s 'localhost:9200/_cat/allocation?v'              # disk.percent at or above 95
      curl -s 'localhost:9200/_all/_settings/index.blocks*?pretty' | grep -B3 read_only   # which indices are blocked

- Fix, in order:
  1. Free space: lower max indices on the largest index set (System -> Indices) and let
     retention delete, or delete the oldest index from that page. Never `rm` index files.
  2. Clear the block if it did not lift itself (OpenSearch releases it automatically once
     usage drops below the high watermark):

         curl -s -XPUT localhost:9200/_all/_settings -H 'Content-Type: application/json' -d '{"index.blocks.read_only_allow_delete": null}'   # lift the block

  3. Rotate the active write index (System -> Indices -> Maintenance) and confirm the
     journal drains.
  4. Redo the disk sizing table so it does not recur.

### Symptom: cluster yellow or red

- Check:

      curl -s 'localhost:9200/_cat/shards?v' | grep -v STARTED          # which shards are not started
      curl -s 'localhost:9200/_cluster/allocation/explain?pretty'        # why the first unassigned shard is unassigned

- Fix: yellow on a single node is almost always replicas > 0 — set replicas to 0 on the
  index set and on existing indices:

      curl -s -XPUT 'localhost:9200/graylog_*/_settings' -H 'Content-Type: application/json' -d '{"index.number_of_replicas":0}'   # repeat per index prefix

  Red means a primary shard is missing — usually a disk or corruption problem. Do not
  start deleting indices until `allocation/explain` says why.

### Symptom: messages appear at the wrong time, or "not there" for a time you know had activity

- Likely cause: clock skew on the sender or on graylog01; sender in local time without
  zone info; syslog input "Allow overriding date" mis-set.
- Check: `timedatectl` and `chronyc tracking` on both ends; open a message and compare
  `timestamp` with the time in the raw `message` text.
- Fix: correct NTP on the sender. Messages already indexed with a wrong timestamp stay
  wrong — search a wider range. For appliances sending local time with no zone, set the
  input's timezone option or fix the device to send UTC / RFC 5424.

### Symptom: messages arrive but are unprocessed / missing fields / not in the expected stream

- Check: open the message, read its actual fields. Stream rules and pipelines match on
  fields that often do not exist as assumed (see `detection-validation.md`, field drift).
- Check: Message Processors order (Pipelines section above), and that the pipeline is
  connected to the stream the message is in.
- Fix: test with the stream rule tester and the pipeline simulator using the real message.

### Symptom: unban search finds nothing for a user who is definitely banned

- Widen the time range to 7–30 days (the PDFs' instruction; bans last up to `1mo` for
  some jails).
- Search the VPN or NAT address as well as the home IP.
- If Graylog has nothing, check the host directly — fail2ban is the authority, Graylog is
  only the index of it:

      sudo fail2ban-client status zimbra-web         # banned IPs in one jail
      ip r | grep 203.0.113.45                       # route-action bans show as unreachable routes

  Full unban procedure: the fail2ban KB PDF and `Mail & Messaging/zimbra-mail-administration-guide.md`.
  If logs from mail.example.com are missing entirely, treat it as "no messages arriving".

### Symptom: web UI unreachable, host up

- Check: `systemctl status mongod opensearch graylog-server --no-pager`; Graylog will not
  start without MongoDB, and reports "no active nodes"/indexer errors without OpenSearch.
- Check: `curl -s http://127.0.0.1:9000/api/system/lbstatus` locally — if ALIVE, the
  problem is the proxy/port 7555 path or the certificate, not Graylog.
- Fix: start in order (Operations -> Start). Check TLS expiry per
  `certificate-pki-lifecycle-guide.md` (Graylog web UI row).

---

## Security

- **Exposure.** Graylog UI and inputs must be internal only. [FILL IN: confirm no NAT or
  pfSense rule publishes 7555, 9000 or any input port to the internet.] OpenSearch (9200)
  and MongoDB (27017) must be bound to localhost — they have no authentication in a
  typical single-node Graylog install.

      sudo ss -lntp | grep -E ':(9200|9300|27017)\b'   # expect 127.0.0.1 only

- **Inputs are unauthenticated.** Anyone who can reach a syslog/GELF port can inject
  messages, including fake fail2ban or Snort lines. Restrict input ports with pfSense and
  ufw to known source VLANs. [CONFIRM: proposal — TLS on the Beats input for hosts outside
  the server VLAN.]
- **Logs are sensitive.** Graylog holds email addresses, source IPs, usernames and
  authentication patterns for all staff. Access is least-privilege, per stream.
- **Secrets.** Admin password, `password_secret`, `root_password_sha2`, bind account,
  SMTP relay credentials and API tokens are in the IT vault (1Password). None belong in
  this document, in a script, or in a shared folder. The triage toolkit's `~/.triage.conf`
  holds a token and must be mode 600.
- **TLS.** [FILL IN: certificate the UI presents on 7555 / 443, issuer and renewal] — add to
  the inventory in `certificate-pki-lifecycle-guide.md`.
- **Graylog is a target.** An attacker who reaches graylog01 can delete the timeline. The
  ransomware playbook requires exporting relevant logs outside the environment during an
  incident. [CONFIRM: proposal — SSH to graylog01 only from the management path, key-only
  per `sshd hardening conf.txt`.]
- **Patching.** Hold packages (see Upgrades) and patch deliberately; Graylog, OpenSearch
  and MongoDB all publish security advisories. Track via
  `Security & Hardening/vulnerability-management-process.md`.

---

## Disaster Recovery

| Item | Value |
|---|---|
| Criticality | Tier 3 — supporting. Restored at layer 8 of the business-continuity order (`SysAdmin Procedures/business-continuity-plan.md`) |
| RTO | [CONFIRM: proposal — 2 business days. Unban work can be done directly on mail.example.com with `fail2ban-client` and `ip r` while Graylog is down] |
| RPO — configuration | [CONFIRM: proposal — 24 hours (daily Mongo dump)] |
| RPO — log data | [CONFIRM: proposal — logs since last snapshot may be lost; accepted for Tier 3, except evidence for an open incident, which is exported at the time] |
| Escalation if owner unavailable | [FILL IN: backup contact / vendor] |

Rebuild outline (new VM, same name):

1. Build Ubuntu per `Linux & Servers/server-build-standard.md`; same hostname and IP so
   senders and Sidecars need no change. [FILL IN: IP from `ipam-vlan-topology-reference.md`.]
2. Install the **same versions** of MongoDB, OpenSearch and Graylog as the backup came
   from (or Docker images at the same tags).
3. Restore `/etc/graylog`, `/etc/opensearch`, `/etc/mongod.conf`, `/etc/default/graylog-server`
   from the tarball. Confirm `password_secret` matches the vault entry.
4. Start MongoDB and restore configuration:

       sudo mongorestore --gzip --archive=/var/backups/graylog/mongo-YYYY-MM-DD.gz --drop   # replaces the graylog database

5. Start OpenSearch; if snapshots exist, restore the index sets you need.
6. Start Graylog. In System -> Indices, recalculate index ranges if searches return
   nothing for restored indices.
7. Verify: inputs running with throughput, Sidecars Active, Overview -> Fail2Ban tab
   present, a known fail2ban search returns results, a test notification delivers.

---

## Decisions & History (ADR-lite)

| Date | Decision / Change | Why / Ticket |
|---|---|---|
| [FILL IN] | Graylog deployed on graylog01.example.com | [FILL IN] |
| [FILL IN] | Overview dashboard Fail2Ban tab created as the unban view | internal fail2ban method |
| [FILL IN] | Sidecar + Filebeat chosen over per-host syslog for servers | Central collector config (hardening cheatsheet) |
| 2026-10-04 | This guide created; hostname conflict recorded as [CONFIRM] | Library consolidation |

## References

Internal:

- [`SysAdmin Procedures/monitoring-alerting-guide.md`](monitoring-alerting-guide.md) — gap analysis, alert routing, muting rules, short Graylog reference
- `[internal KB: fail2ban unban]` — mail.example.com jails, `application_name:fail2ban.*` search, `graylog.example.com:7555`
- `[internal KB: unban a user via Graylog]` and `[internal KB: fail2ban instructions]` — the Overview -> Fail2Ban tab convention
- `Security & Hardening/fresh-box-hardening-cheatsheet.txt` — "Shipping audit output to Graylog": Beats input 5044, Sidecar install, Filebeat config, `Org Audit` pipeline
- `Mail & Messaging/zimbra-mail-administration-guide.md` — Graylog queries for mail troubleshooting
- `Security Procedures/modules/detection-validation.md` — Snort `parse_json` / `snort_` pipeline and field-drift failures
- `Security Procedures/security-toolkit-review/curated/graylog_hunt.sh`, `log_coverage.sh`, `triage.conf` — API-based hunting and source-coverage checks
- `Linux & Servers/server-build-standard.md` — section 10, log shipping is part of every build
- `Linux & Servers/docker-disposable-diagnostic-containers-runbook.txt` — attaching diagnostics to the `graylog` container network
- `Linux & Servers/proxmox-cluster-administration-guide.md`
- `Networking Guide/firewall-change-procedure.md` and `Networking Guide/ipam-vlan-topology-reference.md`
- `Active Directory/ldap-connection-reference.md` and `Security & Hardening/LDAPs certificate replacement procedure.txt`
- `Security & Hardening/certificate-pki-lifecycle-guide.md` — Graylog UI cert, expiry pipeline into Graylog
- `Security & Hardening/secrets-management-guide.md` — API tokens, break-glass alerting
- `Security Procedures/ransomware-response-playbook.md` — Graylog as incident timeline and export requirement
- `Hardware & Backup/backup-restore-testing-procedure.md` and `SysAdmin Procedures/business-continuity-plan.md`
- `SysAdmin Procedures/Change_Management_Log.txt`

External:

- Graylog documentation: https://go2docs.graylog.org/ (Upgrading, Sidecar, Pipelines, Index model)
- OpenSearch cluster and cat APIs: https://opensearch.org/docs/latest/api-reference/
- MongoDB upgrade procedures: https://www.mongodb.com/docs/manual/release-notes/

## Change log

| Date | Author | Change |
|---|---|---|
| 2026-10-04 | IT lead (it@example.com) | NEW — initial draft, UNVERIFIED. Built from existing library docs; all site-specific values not found in the library are marked [FILL IN] / [CONFIRM] |

---
Template v1 (Tier 3) | Maintained by IT Ops | Review annually or after major changes

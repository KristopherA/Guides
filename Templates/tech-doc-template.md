# [System / Service / Tool Name]

<!--
HOW TO USE THIS TEMPLATE
- TIER 1 (minimal): Fill in only the CORE sections. Delete everything below the
  "EXTENDED SECTIONS" divider. Result: a 2-3 paragraph explainer.
- TIER 2 (standard): CORE + Operations + Troubleshooting. Good for most internal
  services (a Docker stack, a reverse proxy, a backup job).
- TIER 3 (full): Use everything. For large systems, multi-host deployments,
  anything with security/compliance implications.
- Delete unused sections rather than leaving them empty. Empty headings rot.
- One doc per system. Link out rather than duplicating.
-->

================================================================
CORE SECTIONS (always fill these in)
================================================================

## Overview

One paragraph: what this is, what problem it solves, who uses it.
Example: "DocuSeal is our self-hosted document signing service at
sign.example.com, used by HR and admin staff. It replaces per-signature
DocuSign costs."

## Quick Facts

| Field            | Value                                  |
|------------------|----------------------------------------|
| Owner            | [person/team responsible]              |
| Environment      | [prod / staging / lab]                 |
| Location         | [host, VM/CT ID, URL, IP]              |
| Access           | [how to reach it: SSH, web UI, VPN]    |
| Dependencies     | [what it needs: DB, proxy, DNS, certs] |
| Dependents       | [what breaks if this goes down]        |
| Last reviewed    | [YYYY-MM-DD]                           |

## How It Works

Two or three paragraphs (or a short diagram description) explaining the
architecture at the level a competent colleague needs to reason about
it. Include: components, data flow, where config lives, where data
lives, where logs go.

Example one-liner style:
  Caddy (host, port 443) -> reverse proxy -> DocuSeal container (port 3000)
  # TLS terminated at Caddy using wildcard.example.com cert

================================================================
EXTENDED SECTIONS (delete what you don't need)
================================================================

## Setup / Installation

Reproducible steps to build this from nothing. Prefer copy-paste
commands over prose. One command per line with a trailing comment:

  docker compose up -d          # starts the full stack from /opt/docuseal
  ufw allow 443/tcp             # required for external access

Link to the compose file / config repo rather than pasting large files.

## Configuration

Where config lives, what the non-default settings are, and WHY each
was chosen. The "why" is the part future-you actually needs.

| Setting          | Value          | Reason                          |
|------------------|----------------|---------------------------------|
| [key]            | [value]        | [why it isn't the default]      |

## Operations (Day-2)

Routine tasks and how to do them:

### Start / Stop / Restart
  systemctl restart caddy       # reloads proxy config, ~1s downtime

### Health Check
  curl -sf https://sign.example.com/health && echo OK   # expect: OK

### Logs
  docker logs -f docuseal --since 1h   # app logs
  journalctl -u caddy --since today    # proxy logs

### Backup & Restore
- What is backed up, where to, on what schedule, by what mechanism.
- The restore procedure, tested on [date]. An untested restore is a rumor.

### Updates / Upgrades
- How to update, in what order, and how to roll back.

## Troubleshooting

Known failure modes, symptom-first so they're greppable:

### Symptom: [what the user/monitor sees]
- Likely cause: [...]
- Check:  [one diagnostic command]     # what output means what
- Fix:    [one remediation command]

(Repeat per known issue. Add entries every time something bites you.)

## Security

- Exposure: [internal only / internet-facing, ports open]
- Auth: [how users authenticate, where accounts live]
- Certificates: [what cert, where it lives, renewal process, expiry]
- Secrets: [where stored — never in this doc]
- Hardening notes / audit findings

## Monitoring & Alerting

- What is monitored, by what (e.g., Graylog stream, uptime check)
- Alert thresholds and where alerts go
- What "normal" looks like (baseline metrics)

## Disaster Recovery

- RTO/RPO expectations
- Full rebuild procedure or link to it
- Contact/escalation order if the owner is unavailable

## Decisions & History (ADR-lite)

| Date       | Decision / Change                  | Why / Ticket        |
|------------|------------------------------------|---------------------|
| YYYY-MM-DD | [e.g., moved TLS to manual wildcard] | [CAA blocked LE]  |

## References

- [Link: upstream docs]
- [Link: config repo / compose file]
- [Link: related internal docs]

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

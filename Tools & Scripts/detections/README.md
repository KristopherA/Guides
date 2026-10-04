# detections-as-code

Detection-as-code for a Snort 3 IDS feed piped into Graylog. Sigma rules are
version-controlled, ATT&CK-tagged, linted and compiled on every change, and
deployed as Graylog Event Definitions.

## Layout

```
rules/                 Sigma detection rules (one idea per rule)
pipelines/             Sigma -> Graylog field mapping (Snort schema)
tests/                 Per-rule validation evidence
coverage/              ATT&CK Navigator layer generator
Makefile               check = lint + convert (the CI gate)
```

## Prerequisites

```bash
pipx install sigma-cli
sigma plugin install elasticsearch
pip install pyyaml            # for the coverage script
```

## Workflow

```bash
make check                    # lint + compile every rule (run before commit)
make convert                  # print the Graylog Lucene query for each rule
make coverage                 # regenerate coverage/attack-layer.json
```

## Deploying a rule to Graylog

1. `make convert` and copy the Lucene query for the rule.
2. Graylog -> Alerts -> Event Definitions -> Create.
3. Filter & Aggregation: paste the query as the filter.
4. For threshold rules (port-scan, repeated-sid): add the aggregation in the
   UI — `count() > N grouped by snort_src_addr [, snort_sid]`, 5-min window.
   Sigma's `count()` line documents intent but does not reliably survive
   conversion to the ES/Graylog backend; enforce the count here.
5. Set notification + severity from the rule's `level`.

## Before trusting these against a real environment

These rules target the standard Snort 3 `alert_json` schema with a `snort_`
field prefix (matching a "parse Snort alert JSON" Graylog pipeline rule).
Placeholder values are used throughout — confirm these against your own
environment before relying on any rule:

- **Field names** in `pipelines/snort-graylog.yml` — match against one real
  parsed event.
- **Internal CIDR ranges** in `snort-internal-compromise.yml`.
- **Watchlist host IPs** in `snort-crown-jewel-watchlist.yml`.

## Rules

| Rule | Detects | Level |
|------|---------|-------|
| snort-priority-1 | Snort's own priority-1 alerts | high |
| snort-dangerous-class | trojan/admin/shellcode/web-attack classes | high |
| snort-port-scan | port fan-out from one source | medium |
| snort-repeated-sid | same signature hammering from one source | medium |
| snort-internal-compromise | internal host sourcing C2 alerts | critical |
| snort-crown-jewel-watchlist | any alert touching key hosts | medium |

## License

MIT — see LICENSE.
 
# Detection Validation Runbook

Turns "I wrote detections" into "I run a detection program."

Scope: Snort 3 IDS -> Graylog -> Sigma-derived searches (detections-as-code repo)


## 1. The detection chain (validate every link, not just the last one)

A Sigma rule can only fire if every upstream link works. Test each one.

```
  adversary traffic
     -> Snort 3 sensor        (link A: SID fires, sensor sees the packets)
     -> alert_json output     (link B: alert is emitted with the fields you need)
     -> Graylog input
     -> parse_json pipeline   (link C: snort_ fields populate correctly)
     -> stream routing
     -> Sigma search / alert  (link D: rule logic matches the parsed event)
```

Five questions per detection:
1. Does the stimulus produce the network behavior?      (traffic gen)
2. Does Snort see it and alert?                          (link A)
3. Is the alert emitted with the right fields?           (link B)
4. Do the `snort_` fields parse correctly in Graylog?    (link C)
5. Does the Sigma logic match the parsed event?          (link D)

Most "my rule didn't fire" mysteries are link A (sensor placement / SID
disabled) or link C (field name drift), NOT rule logic.


## 2. Two-stage validation: offline first, then online

Validate the Snort layer deterministically OFFLINE before touching the wire.
This isolates rule-logic failures from sensor-placement failures.

```bash
snort -c /etc/snort/snort.lua -r sample.pcap -A alert_json   # offline: does YOUR config fire on this pcap
tail -n50 alert_json.txt | jq .                              # inspect what fired (sid, class, priority, addrs)
```

If it fires offline but not live, the problem is link A (interface, span
port, DAQ) — not your rules. If it fires live too, links A+B are good; move
to Graylog.

```bash
tcpreplay -i <mon-iface> sample.pcap                          # online: does the LIVE sensor see it
tail -f /var/log/snort/alert_json.txt | jq '.sid, .msg'       # watch alerts arrive in real time
```

SAFETY: replay malicious pcaps only on the sensor's monitoring segment or a
lab/test VLAN, never onto a production data path. Offline (`-r`) touches no
wire.


## 3. Stimulus catalog — one trigger per rule

Map each rule to a concrete, repeatable stimulus.

| Rule | Stimulus |
|---|---|
| priority-1 alerts | `tcpreplay -i <mon> high-prio.pcap` — a pcap with a known high-priority SID |
| dangerous classifications | `snort -r trojan-activity.pcap -A alert_json` — pcap with the target classtype; confirm class first |
| port-scan fan-out | `nmap -sS -p1-1000 <lab-target>` — SYN scan across many ports |
| repeated-SID hammering | `for i in $(seq 1 50); do nmap -sS -p22 <t>; done` — loop one trigger past the threshold |
| internal C2 sourcing | `while true; do curl -s http://<c2-lab>/beacon; sleep 30; done` — beacon from an internal host to a flagged dst |
| crown-jewel watchlist | `nmap -sS -p1-100 <watchlisted-ip>` — any alert-worthy traffic to/from a watchlisted asset |

Curated pcaps: malware-traffic-analysis.net, Snort registered-rule test
pcaps. Keep a `tests/pcaps/` dir in the repo, one pcap per rule, named by
rule id.


## 4. Per-detection validation loop (symptom -> check -> fix)

Run this for each rule. Record PASS/FAIL in the validation log (section 7).

**Step 1** — identify the SID(s) the rule depends on:
```bash
grep -i 'snort_sid' rules/snort-port-scan.yml   # what SIDs / conditions the rule keys on
```

**Step 2** — fire the stimulus (section 3).

**Step 3** — confirm Snort alerted (link A/B):
```bash
tail -f /var/log/snort/alert_json.txt | jq 'select(.sid==<SID>)'   # did the SID fire at all
```

**Step 4** — confirm it parsed in Graylog (link C). Search the raw stream:
```
snort_sid:<SID>
```
Verify `snort_classification` / `snort_priority` / `snort_src_addr` are
populated, not missing.

**Step 5** — confirm the Sigma search matches (link D). Run the rule's
translated query:
```bash
make convert   # emit the Graylog query for each rule, paste into search
```
It should return the event(s) from your stimulus.

**Step 6** — record the result and the detection latency (stimulus -> alert).

Common failures:

| Symptom | Likely cause |
|---|---|
| Fires offline, not live | Sensor not on the traffic path / wrong mon-iface (link A) |
| In Graylog raw, not in rule | Field-name drift; rule uses `snort_class`, pipeline emits `snort_classification` (link C) |
| Nothing in Graylog at all | `parse_json` pipeline rule not attached to the stream / input down |


## 5. False-positive baselining

A rule that fires on the stimulus but also fires dozens of times a day on
normal traffic is not production-ready.

- Run each rule's query against a **clean traffic window** (a period with no
  known incidents or intentional stimulus) — ideally 24-72 hours.
- Record the hit count. Zero or near-zero on clean traffic + a clean hit on
  the stimulus = a healthy rule.
- If a rule is noisy on clean traffic, tighten the logic (narrower
  classification match, higher threshold, an allowlist for known-good
  scanners/tooling) before shipping it live.
- Re-baseline after any rule-logic change.


## 6. Coverage: has-a-rule vs. has-a-passing-test

The ATT&CK Navigator layer (`coverage/attack-layer.py`) shows which
techniques have a *rule*. That is not the same as which techniques have a
*validated* detection. Track both:

- **Has-rule**: a Sigma rule exists and tags the technique.
- **Has-passing-test**: the rule has been run through this runbook's
  validation loop (sections 2-4) with a recorded PASS in the validation log.

A rule with no passing test is a hypothesis, not a detection. Prioritize
closing that gap over writing new rules.


## 7. Validation log

Keep a running log at `tests/validation-log.md`:

| Date | Rule | Stimulus used | Link A | Link B | Link C | Link D | FP baseline | Result |
|---|---|---|---|---|---|---|---|---|
| 2026-08-01 | snort-port-scan | nmap -sS -p1-1000 | PASS | PASS | PASS | PASS | 0/24h | PASS |

Update on every validation run, not just the first one — field mappings and
sensor placement drift over time.


## 8. CI vs. scheduled purple-team exercises

- **CI (`make check`)**: catches syntax and schema errors on every commit.
  It cannot tell you the rule actually fires against real traffic.
- **Scheduled validation (this runbook)**: run the full stimulus catalog on
  a recurring cadence (e.g. quarterly, or after any pipeline/sensor change)
  to catch drift that CI can't see — field renames, sensor placement
  changes, SID deprecations.

Treat CI as a fast, cheap gate and scheduled validation as the actual proof
the detection program works.


## Next step

The highest-leverage next addition is a `make offline-test` Makefile target
that automates section 2-4 for every rule against its pcap in
`tests/pcaps/`, plus a `make fields` target that checks the live `snort_`
schema against what `pipelines/snort-graylog.yml` expects.

# MODULE: DETECTION VALIDATION

Stand-alone. Reached from: `detection-engineering.txt` step 7, after a rule is
written; and from `../triage/stage-3-verdict/procedure.txt` step 11, when a
confirmed indicator becomes a test stimulus.

Purpose: prove each detection rule actually fires on real adversary behavior,
and measure its false-positive rate. Turns "we wrote detections" into "we run
a detection program."

Last reviewed: 2026-09-04
Owner: whoever owns detection engineering — put a name here.
Detections repo: `<your-detections-repo>` — substitute your own path
throughout.

## Adapting this to your stack

The worked example below is a Snort 3 -> Graylog -> Sigma chain, because a
concrete chain is the only way to show what validating each *link* means. The
method is not specific to those tools. Substitute your own and the five
questions in section 1 do not change:

  | This runbook says | Substitute your |
  |---|---|
  | Snort 3 sensor        | IDS/NIDS, EDR, or whatever generates the event |
  | `alert_json` output   | that sensor's output format |
  | Graylog input + pipeline | your SIEM's ingest and parsing layer |
  | `snort_` field prefix | your parser's field namespace |
  | Sigma-derived search  | your detection rule in its native query language |
  | `make` targets        | your build or CI runner |

Anything in `<angle brackets>` is a placeholder to replace.


## 1. The detection chain (validate every link, not just the last one)

A Sigma rule can only fire if every upstream link works. Test each one.

  adversary traffic
     -> Snort 3 sensor        (link A: SID fires, sensor sees the packets)
     -> alert_json output     (link B: alert is emitted with the fields you need)
     -> Graylog input
     -> parse_json pipeline   (link C: snort_ fields populate correctly)
     -> stream routing
     -> Sigma search / alert  (link D: rule logic matches the parsed event)

Five questions per detection:
  1. Does the stimulus produce the network behavior?      (traffic gen)
  2. Does Snort see it and alert?                          (link A)
  3. Is the alert emitted with the right fields?           (link B)
  4. Do the snort_ fields parse correctly in Graylog?      (link C)
  5. Does the Sigma logic match the parsed event?          (link D)

Most "my rule didn't fire" mysteries are link A (sensor placement / SID
disabled) or link C (field name drift), NOT rule logic.


## 2. Two-stage validation: offline first, then online

Validate the Snort layer deterministically OFFLINE before touching the wire.
This isolates rule-logic failures from sensor-placement failures.

  snort -c /etc/snort/snort.lua -r sample.pcap -A alert_json   # offline: does YOUR config fire on this pcap
  tail -n50 alert_json.txt | jq .                              # inspect what fired (sid, class, priority, addrs)

If it fires offline but not live, the problem is link A (interface, span port,
DAQ) — not your rules. If it fires live too, links A+B are good; move to Graylog.

  tcpreplay -i <mon-iface> sample.pcap                         # online: does the LIVE sensor see it
  tail -f /var/log/snort/alert_json.txt | jq '.sid, .msg'     # watch alerts arrive in real time

SAFETY: replay malicious pcaps only on the sensor's monitoring segment or a
lab/test VLAN, never onto a production data path. Offline (-r) touches no wire.


## 3. Stimulus catalog — one trigger per rule

Map every rule to a concrete, repeatable stimulus. A rule with no stimulus
cannot be validated, which in practice means it has never been proven to work.

The rows below are examples of the shape a stimulus takes. Replace them with
one row per rule you actually have.

  RULE                      STIMULUS (one-liner)
  ------------------------  ----------------------------------------------------
  priority-1 alerts         tcpreplay a pcap with a known high-priority SID
                              tcpreplay -i <mon> high-prio.pcap
  dangerous classifications pcap with a target classtype (e.g. trojan-activity)
                              snort -r trojan-activity.pcap -A alert_json   # confirm class first
  port-scan fan-out         nmap SYN scan across many ports
                              nmap -sS -p1-1000 <lab-target>
  repeated-SID hammering     loop one trigger N times past the threshold
                              for i in $(seq 1 50); do nmap -sS -p22 <t>; done
  internal C2 sourcing       beacon from an internal host to a flagged dst
                              while true; do curl -s http://<c2-lab>/beacon; sleep 30; done
  crown-jewel watchlist      any alert-worthy traffic to/from a watchlisted asset
                              nmap -sS -p1-100 <crown-jewel-ip>

Curated pcaps: malware-traffic-analysis.net, Snort registered-rule test pcaps.
Keep a `tests/pcaps/` dir in the repo, one pcap per rule, named by rule id.

A stimulus can also come out of a triage case: a confirmed-malicious domain
or hash from `../triage/stage-3-verdict/` is a ready-made test for the rule
you are about to write against it.


## 4. Per-detection validation loop (symptom -> check -> fix)

Run this for each rule. Record PASS/FAIL in the validation log (section 7).

Step 1 — identify the SID(s) the rule depends on:
  grep -i 'snort_sid' rules/<rule-id>.yml          # what SIDs / conditions the rule keys on

Step 2 — fire the stimulus (section 3).

Step 3 — confirm Snort alerted (link A/B):
  tail -f /var/log/snort/alert_json.txt | jq 'select(.sid==<SID>)'   # did the SID fire at all

Step 4 — confirm it parsed in Graylog (link C). Search the raw stream:
  snort_sid:<SID>                                  # Graylog search: event present with parsed sid
  # verify snort_class / snort_priority / snort_src_addr are populated, not missing

Step 5 — confirm the Sigma search matches (link D). Run the rule's translated query:
  make sigma-to-graylog RULE=<rule-id>             # emit the SIEM query, paste into search
  # it should return the event(s) from your stimulus

Step 6 — record the result and the detection latency (stimulus -> alert).

Common failures:
  Fires offline, not live      -> sensor not on the traffic path / wrong mon-iface (link A)
  In Graylog raw, not in rule  -> field-name drift; rule uses snort_class, pipeline emits snort_classification (link C)
  Nothing in Graylog at all    -> parse_json pipeline rule not attached to the stream / input down


## 5. False-positive baselining

A rule that fires on the stimulus but also 40x/day on normal traffic is not
production-ready. Baseline every rule against a CLEAN window before shipping.

Step 1 — run the rule's query over a quiet, known-good window:
  <rule query> AND NOT source:<your-test-host>     # Graylog, timerange = last 7d normal ops
Step 2 — aggregate to a rate:
  # Graylog: add a "count" aggregation grouped by day (or by snort_src_addr)
Step 3 — triage the hits: each one is either a true signal you missed, or an FP.
Step 4 — record FP/day per rule. Tune (add exclusions, raise thresholds) and re-baseline.

  RULE                    FP/day (pre-tune)   FP/day (post-tune)   Verdict
  <rule-id>               [ ]                 [ ]                  [ship/hold]

Rule of thumb: >1 FP/analyst/day per rule erodes trust in the whole program.


## 6. Coverage measurement

If you generate an ATT&CK Navigator layer from your rules, extend it to a
two-tone map. If you do not generate one yet, this is the reason to start.

  - has-rule          (a detection exists for this technique)
  - has-passing-test  (that detection is validated per section 4)

  make attack-layer                                 # techniques you have rules for
  make attack-layer VALIDATED=1                     # only techniques with a PASS in the log

A technique that is has-rule but NOT has-passing-test is your real coverage gap:
you think you're covered but have never proven it. Those are the priority.


## 7. CI vs scheduled purple exercise (know what belongs where)

You cannot replay live traffic inside CI. Split validation into two cadences.

  CI (every commit — add these to the build gate):
    make lint         # sigma syntax valid
    make fields       # every snort_ field a rule references exists in the pipeline schema
    make offline-test # snort -r tests/pcaps/<rule>.pcap fires the expected SID (deterministic, no wire)

  Scheduled purple exercise (monthly/quarterly):
    full section 4 loop on the live sensor -> Graylog -> Sigma chain
    update the validation log + FP baselines
    regenerate the coverage layer

The offline-test target is the high-value add: it makes SID-level firing a
commit-blocking check, catching rule/pcap drift automatically.


## 8. Validation log

Keep `tests/validation-log.md`, one row per rule per exercise. This is the
evidence that you run a program rather than keeping a pile of rules — and it
is what an auditor, a new analyst, or your own future self needs in order to
trust any given detection.

Fill it in honestly. A log with no FAIL rows in it has almost certainly not
been used.

  DATE        RULE                STIMULUS           CHAIN A/B/C/D        LATENCY  FP/DAY  RESULT
  2026-09-04  <rule-id>           nmap -sS -p1-1000  PASS PASS PASS PASS  4s       0.2     PASS
  2026-09-04  <other-rule-id>     curl beacon x120   PASS PASS FAIL ----  --       --      FAIL (link C: field drift)


## Next steps

  1. Create `tests/pcaps/` and drop one pcap per rule.
  2. Add the offline-test build target (biggest return, fully automatable).
  3. Run section 4 once by hand on every rule; fill the log honestly.
  4. Baseline FP/day (section 5); tune the noisy ones.
  5. Regenerate the coverage layer with the VALIDATED flag.


## Feeds back to

  A validated rule with a known FP rate is what makes a future triage verdict
  trustworthy. When a rule fires, the alert becomes the start of a new case:

    ../START-HERE.txt                pick the entry stage for what you have
    detection-engineering.txt        step 9, the alert triage playbook

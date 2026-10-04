# Hunting Malware with Containers — Expanded Notes

Source: "Hunting Malware with Containers", a workshop by **CompSec Direct**
([@compsecdirect](https://www.youtube.com/@compsecdirect) on YouTube, also
streamed on Twitch). These are notes on that material, not a substitute for
it — the vendor's own documentation is the authority on anything here.

## Core idea

Run malware analysis in **containers** rather than full VMs to lower cost, speed up reset cycles, and make labs reproducible. Key concerns when building a test environment:

- **Cost** — containers are cheaper than always-on VMs.
- **Speed** — containers spin up in seconds, snapshot/reset is trivial.
- **Reliability** — declarative images mean every analyst gets the same lab.
- **Internet egress** — C2 callbacks can trace back to your home/corporate IP. Always route through VPN, Tor, or an isolated network.
- **Don't run the container as root.** ← repeated for emphasis in the workshop.

---

## Tools & resources covered

### Cyber range — Kleared4 / SecureRange

- **Kleared4 cyber range:** <https://compsecdirect.com/kleared4-cyber-range/>
- **Lab portal:** <https://one.securerange.com/>
- Hosted, browser-accessible lab environments built by CompSec Direct for hands-on malware/IR training.
- Access is issued per workshop or per subscription — request your own from the
  vendor. (Shared workshop credentials were removed from these notes: they
  belong to someone else's platform and do not belong in a shared runbook.)

### Apache Guacamole *(notes spelled it "Gaucamolio")*

- Clientless remote desktop gateway. Gives RDP / SSH / VNC access through a browser via WebSockets — no client install on the analyst's machine. The Kleared4 portal uses it under the hood.
- Project: <https://guacamole.apache.org/>

### Sample sourcing — MalwareBazaar

- <https://bazaar.abuse.ch/> — abuse.ch's free malware sample feed. Search by SHA256 / family / tag, daily fresh samples. Downloads are zipped with password `infected`.
- API example (one-line): `curl -X POST https://mb-api.abuse.ch/api/v1/ -d "query=get_recent&selector=time"`

### CompSec Direct's puller — uctamas

- <https://github.com/compsecdirect/uctamas>
- Python script that automates pulling samples from MalwareBazaar (the instructor's own tool, demoed live).
- Typical use: `python uctamas.py` to fetch the latest batch into your container's working dir.

### Interactive sandbox — ANY.RUN

- <https://app.any.run/>
- Online interactive sandbox: detonate a sample in a Windows VM, click around it live, watch process tree / network / file / registry events. Free tier is public-only (your task is visible to others).

### Automated unpacking — UnpacMe

- <https://www.unpac.me>
- Submits a packed binary, returns the unpacked artifact(s). Detects common packers (UPX, Themida, ASPack, custom). Useful before running YARA on packed malware.

### YARA rule generation — yarGen

- <https://github.com/Neo23x0/yarGen> — by **Florian Roth**, "yarGen is a generator for YARA rules" (current v0.23.2).
- Workflow: drop samples into a folder → yarGen extracts strings/opcodes that don't appear in goodware → emits a draft `.yar` rule.
- One-line example (create rules from a folder of office samples):
  `yarGen.py -c --opcodes -i office -g /opt/packs/office2013`
- First run needs the goodware DB: `python yarGen.py --update` (~913 MB).

---

## Techniques mentioned

- **Scripted execution** — drive the malware run from inside the container (cmd / PowerShell / shell script) so the lab is repeatable and unattended.
- **Remote task managers** — observe process state on the detonation container from the analyst host (Procmon-like tools, container-aware monitors).
- **Async calls** — fan out multiple sample detonations in parallel across containers; orchestrate with `asyncio` / a queue rather than running serially.

---

## Safety checklist (from the workshop)

1. Container **NOT** running as root.
2. Network egress through a controlled exit (VPN / dedicated cloud egress IP) — never your home WAN.
3. Snapshot or rebuild the container between samples.
4. Strip / never share API keys baked into your detonation image.
5. Treat unpacked / extracted artifacts as still-live malware until proven otherwise.

---

*Source: notes taken during the CompSec Direct "Hunting Malware with Containers" workshop, expanded from public project documentation and confirmed where reachable. Shared workshop credentials that appeared in the raw notes have been removed.*

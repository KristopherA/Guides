# Diagnostic Scripts — macOS Network / Wi-Fi Troubleshooting

Read-only automation for the data-collection and first-pass classification parts of the troubleshooting playbooks in this folder. **No script changes any setting** — they collect, measure, classify, and recommend. Configuration changes (forget network, toggle IPv6, VPN split-tunnel), physical walk tests, and final diagnosis stay human, by design.

## What's here

| Script | Does | Playbook |
|---|---|---|
| `net-triage-snapshot.sh` | Full triage collection (Wi-Fi link, IP/DHCP, DNS, reachability, performance), then classifies the likely fault layer and writes a report (+ optional JSON). The workhorse. | Triage Decision Tree; Wi-Fi guide §1–§4 |
| `zoom-rtc-check.sh` | Real-time media readiness: latency/jitter/loss, upload capacity, UDP-vs-TCP-fallback, VPN state → call-readiness verdict. | Video Conferencing Troubleshooting |
| `net-baseline.sh` | Captures known-good values (signal, throughput, DHCP/DNS/gateway) for the Baseline Register. | Ops Templates §3; Wi-Fi guide §7.1 |
| `net-monitor.sh` | Continuous CSV logger (RSSI, latency, jitter, loss over time) to catch intermittent problems. | Wi-Fi guide §4.4 |

## Windows laptops

PowerShell versions of all four scripts live in **`windows/`** (`*.ps1` + `Networking Guide/Wifi Guide/scripts/windows/README-windows.md`) for a mixed fleet — same read-only design and comparable report/verdict output. Two Windows platform limits are documented there: signal is a percentage (not dBm/SNR), and there's no `networkQuality`/RPM test (jitter/loss are measured instead). The macOS `.sh` scripts below are for Macs.

## Requirements

- macOS 12+ (uses `wdutil`, `networkQuality`, `ipconfig`, `scutil`, `dig`, `ping`, `lsof` — all built in).
- **Run with `sudo`** for Wi-Fi signal fields (RSSI/SNR). macOS 14+ redacts these without elevated privileges; the scripts still run and note what was unavailable.
- **SSID shows `<redacted>`?** `wdutil` hides SSID/BSSID unless the *terminal app itself* has Location Services permission — `sudo` alone isn't enough. Grant it once: System Settings → Privacy & Security → Location Services → enable for Terminal/iTerm. The scripts also try `networksetup -getairportnetwork` as a fallback.
- Make executable once: `chmod +x *.sh`

## Usage

```
sudo ./net-triage-snapshot.sh -j            # full snapshot + JSON, report to ~/Desktop
sudo ./net-triage-snapshot.sh -o /tmp       # choose output dir
     ./zoom-rtc-check.sh                     # run DURING a call for the UDP/TCP check
sudo ./net-baseline.sh -l "HQ-3rdFloor"     # label the baseline by location
sudo ./net-monitor.sh -i 5 -d 30            # sample every 5s for 30 min (Ctrl-C to stop)
```

`net-triage-snapshot.sh` returns an **exit code = classified layer** for scripted routing:
`0` healthy · `2` RF/coverage · `3` DHCP · `4` local-segment · `5` path/ISP · `6` DNS · `7` performance · `9` incomplete.
`zoom-rtc-check.sh`: `0` ready · `1` marginal · `2` poor · `9` incomplete.

## Runtime — don't interrupt

`net-triage-snapshot.sh` and `zoom-rtc-check.sh` each take ~35–40 seconds because they run multiple `ping` samples plus `networkQuality`, and they only write the report **when finished**. They now print per-step progress to the terminal (`[1/3] measuring...`) so you can see they're working — let them run to completion. Stopping early (Ctrl-C) means no report is written. `net-baseline.sh` takes ~15 s; `net-monitor.sh` runs until its duration elapses or you Ctrl-C (and writes each sample live).

## Reading the output

Each script writes a timestamped, human-readable report to the output dir (default `~/Desktop`). The triage script leads with a **VERDICT** naming the likely layer and pointing to the matching playbook section. For a Zoom ticket, still open **Zoom → Settings → Statistics** for per-stream Send/Receive numbers — the script measures the path, the app measures the actual streams.

## Deploying via MDM (fleet use)

- Push as a script/policy (Jamf, Kandji, Intune). They run non-interactively and need no input.
- Run as root so signal fields populate; direct output somewhere collectable, e.g. `-o /Users/Shared` or a managed path, and have the MDM harvest the report/JSON.
- The triage script's exit code lets an MDM smart group or workflow route tickets by layer automatically.
- For a WFH self-service flow, hand a user `net-triage-snapshot.sh` (or `zoom-rtc-check.sh`) and have them return the report — the same role the inline §6.4 script plays in the Wi-Fi guide, but with classification built in.

## Limits (by design)

- **Read-only.** No remediation. Recommended fixes are printed as text; a human applies them.
- **Path, not RF truth.** Scripts measure what the client sees; they don't replace a WiFi Explorer scan, spectrum analysis, or a physical survey.
- **Classification is a first pass**, not a diagnosis — it narrows the search; judgment confirms it.
- **UDP/TCP media check needs a live call** to be meaningful.

## Verify before trusting on a new OS

macOS occasionally changes tool output formats. After a major macOS update, run each script once on a known-good machine and confirm the fields still populate (see Wi-Fi guide §7.7 "keep skills fresh"). The parsers are built around current `wdutil`/`networkQuality`/`ping` output.

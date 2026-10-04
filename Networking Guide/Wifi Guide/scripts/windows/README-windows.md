# Diagnostic Scripts — Windows / PowerShell

PowerShell counterparts to the macOS scripts one folder up, for diagnosing Windows laptops on a mixed fleet. Same read-only philosophy, same report/verdict structure, so a Windows report and a Mac report are directly comparable. **No script changes any setting.**

## What's here

| Script | Does | macOS twin |
|---|---|---|
| `net-triage-snapshot.ps1` | Full triage collection + layer classification + report (optional `-Json`). | `Networking Guide/Wifi Guide/scripts/net-triage-snapshot.sh` |
| `zoom-rtc-check.ps1` | Real-time media readiness: jitter/loss, Zoom UDP-vs-TCP path, VPN state, verdict. | `Networking Guide/Wifi Guide/scripts/zoom-rtc-check.sh` |
| `net-baseline.ps1` | Captures known-good values for the Baseline Register. | `Networking Guide/Wifi Guide/scripts/net-baseline.sh` |
| `net-monitor.ps1` | Continuous CSV logger (signal, latency, jitter, loss over time). | `Networking Guide/Wifi Guide/scripts/net-monitor.sh` |

## Requirements

- Windows 10 or 11. Works with **Windows PowerShell 5.1** (built in) or **PowerShell 7**.
- Built-in cmdlets only: `netsh`, `Get-NetRoute`, `Get-NetIPAddress`, `Get-DnsClientServerAddress`, `Resolve-DnsName`, `Get-NetTCPConnection`, `Get-NetUDPEndpoint`, `Get-NetAdapter`, `Get-VpnConnection`, and `ping.exe`.
- Runs as a standard user. Run **as Administrator** for the most complete process/adapter data (the Zoom UDP/TCP check reads your own processes, so it works unelevated).

## Running (execution policy)

Scripts aren't signed, so launch them bypassing policy for that one process — no permanent policy change:

```
powershell -ExecutionPolicy Bypass -File .\net-triage-snapshot.ps1 -Json
powershell -ExecutionPolicy Bypass -File .\zoom-rtc-check.ps1
powershell -ExecutionPolicy Bypass -File .\net-baseline.ps1 -Label "HQ-3rdFloor"
powershell -ExecutionPolicy Bypass -File .\net-monitor.ps1 -IntervalSec 5 -DurationMin 30
```

Reports write to the Desktop by default; use `-OutDir C:\path` to redirect. `net-triage-snapshot.ps1` returns an **exit code = classified layer** (`0` healthy, `2` RF, `3` DHCP, `4` local-segment, `5` path/ISP, `6` DNS, `7` performance, `9` incomplete) for RMM/scripted routing.

## Two real platform differences from macOS

These are Windows limitations, not script bugs — the reports call them out inline:

1. **Signal is a percentage, not dBm, and there's no SNR.** `netsh wlan show interfaces` reports `Signal : 92%`. The scripts show the percent and an **approximate** dBm (`percent/2 − 100`, so 92% ≈ −54 dBm) for rough comparison with Mac readings — treat it as an estimate. Windows exposes no noise floor, so SNR isn't available. For true dBm/SNR/spectrum, use WiFi Explorer's remote sensor or a Windows analyzer (e.g., a WLAN adapter tool); the guide's RF thresholds still apply once you have real dBm.
2. **No `networkQuality`/RPM equivalent.** Windows has no built-in responsiveness/bufferbloat test, so there's no RPM score. The scripts instead measure **jitter and loss** (via `ping`), which are the metrics that actually break real-time media. For throughput and bufferbloat, run a manual speed test or `iperf3`, or a browser bufferbloat test (e.g., waveform.com). The triage script's performance verdict therefore keys off gateway jitter/loss rather than RPM.

Everything else maps cleanly: APIPA (169.254) detection, gateway/internet reachability, configured-vs-public DNS resolution, Zoom UDP-8801-8810-vs-TCP/443 media-path detection, and VPN/adapter detection all have direct Windows equivalents.

## Deploying via MDM / RMM (Intune, ConfigMgr, etc.)

- Push as a PowerShell script/policy; they run non-interactively.
- Intune "Platform scripts" or a proactive remediation can run these and collect the report/JSON; the triage exit code lets a remediation flag machines by fault layer.
- Direct output to a collectable path with `-OutDir`, e.g. `-OutDir C:\ProgramData\NetTriage`.

## Validating the scripts

Two levels, and they're different:

**1. Syntax + lint (works on macOS/Linux/Windows).** Run the included `validate-ps.ps1` — it parse-checks every script with PowerShell's own parser and lints them with PSScriptAnalyzer, in one command:

```
brew install --cask powershell          # macOS, if not already installed
cd scripts/windows
pwsh -NoProfile -File ./validate-ps.ps1              # parse + lint
pwsh -NoProfile -File ./validate-ps.ps1 -SkipLint    # parse only (no module install)
```

It exits `0` clean, `1` on parse errors, `2` on lint findings, and prints per-file results. This proves the code is syntactically valid and lint-clean anywhere.

**2. Runtime behavior (needs Windows).** `validate-ps.ps1` deliberately does **not** run the scripts, because they call Windows-only cmdlets (`Get-NetRoute`, `netsh`, `Get-NetTCPConnection`, …) that don't exist on PowerShell for macOS/Linux. To confirm they actually work — fields populate, UDP/TCP detection fires, verdict is right — run each once on a real Windows laptop, a Windows 11 VM (Parallels/UTM on Apple silicon), or a `windows-latest` GitHub Actions runner, exactly as you validated the macOS set. Note `netsh` field labels can vary slightly by Windows build (e.g., "Receive rate (Mbps)"); adjust the labels in `Get-WifiInfo` if yours differ.

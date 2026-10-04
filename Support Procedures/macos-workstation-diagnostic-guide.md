# macOS Workstation Diagnostic Collector 2.1

The collector is read-only. It does not repair disks, restart services, install
updates, change security settings, or upload the report. A normal run does not
require administrator access.

## Standard collection

```zsh
./macos-workstation-diagnostic.zsh --output "$HOME/Desktop"
```

The script creates a private report folder and a ZIP archive. Start with
`summary.txt`; use `report.txt` for the full evidence. `report.json`,
`findings.tsv`, and `metrics.tsv` support ticketing and RMM integrations.

Result codes:

- `0`: no warning or critical threshold detected
- `1`: one or more warnings
- `2`: one or more critical findings
- `3`: a required collector was unavailable or the collection was incomplete

## Focused profiles

```zsh
./macos-workstation-diagnostic.zsh --performance
./macos-workstation-diagnostic.zsh --network --network-test
./macos-workstation-diagnostic.zsh --battery
./macos-workstation-diagnostic.zsh --storage --verify-volume
./macos-workstation-diagnostic.zsh --login --include-logs
./macos-workstation-diagnostic.zsh --security
./macos-workstation-diagnostic.zsh --application "Microsoft Teams" --include-logs
```

`--performance` samples for 60 seconds by default. Change this with
`--sample-seconds` and `--sample-interval`.

## Ticket metadata

```zsh
./macos-workstation-diagnostic.zsh \
  --ticket INC-12345 \
  --technician AB \
  --symptom "Intermittent Wi-Fi after waking" \
  --affected-app "Microsoft Teams"
```

## Baselines and change detection

Create a known-good baseline:

```zsh
./macos-workstation-diagnostic.zsh --save-baseline "$HOME/Desktop/known-good-state.tsv"
```

Compare a later run:

```zsh
./macos-workstation-diagnostic.zsh --baseline "$HOME/Desktop/known-good-state.tsv"
```

The state includes macOS/build/model, security controls, applications and
versions, system extensions, network services, and enrollment state. A baseline
can come from the same Mac, an approved standard build, or a comparable peer.

## Enterprise checks

Copy and edit `macos-diagnostic-enterprise.example.conf`, then run:

```zsh
./macos-workstation-diagnostic.zsh \
  --enterprise-config ./company-mac-checks.conf \
  --network-test
```

The configuration supports expected processes, paths, applications, system
extensions, certificate expiry, and service URLs. It is parsed as data and is
never executed as shell code.

## Privacy levels

- `internal`: retains workstation identifiers useful to an internal help desk
- `vendor`: redacts usernames, home paths, serials, UUIDs, MAC addresses, and IPs
- `public`: adds email, Wi-Fi, organization, enrollment, and computer-name redaction

`vendor` is the default. Recent unified logs, app inventories, process names,
and diagnostic filenames can still disclose sensitive context. Review every
report before sharing it.

## Slower or sensitive options

- `--include-logs` captures capped recent error/fault entries.
- `--network-test` generates external traffic and checks available updates.
- `--verify-volume` runs read-only volume verification and can take time.
- `--extended` adds peripherals, applications, system extensions, Bluetooth,
  audio, camera, installation history, and background items.
- `--security` adds firewall details, installed profiles, launch persistence,
  listening TCP services, current-user scheduled tasks, and browser-extension
  locations. It inventories evidence; it does not declare the Mac compliant.
- Running the script as root can expose additional Wi-Fi radio information, but
  routine collection should remain unprivileged.

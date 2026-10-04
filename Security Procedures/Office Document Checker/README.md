# Office Document Checker

A read-only macOS triage script for inspecting a suspicious Office
document (Word/Excel/PowerPoint) without opening it.

## Overview

`office_malware_check.sh` hashes the file, checks it against VirusTotal
and ClamAV, scans for malicious VBA macros, and unpacks OOXML
containers to look for external links and embedded objects. It writes
everything to a timestamped report folder. This guide gets a machine
set up to run it from scratch. Follow the steps in order — each tool the
script uses is installed in Step 3.

## Quick Facts

| Field         | Value                                              |
|---------------|----------------------------------------------------|
| Platform      | macOS (Apple Silicon or Intel)                     |
| Runs as       | Normal user account (no sudo needed to run it)     |
| Depends on    | Homebrew, ClamAV, ripgrep, oletools, vt-cli        |
| Network       | Only VirusTotal step sends data (hash only, no file) |
| Output        | ./office_check_<file>_<timestamp>/report.txt       |

## How It Works

You run the script against one file. It creates a working folder in the
current directory, runs each check in sequence, and appends the results
to `report.txt` inside that folder. Nothing is installed or changed on
the suspect file — it is read-only triage.

  suspect.docx -> hash -> VirusTotal (hash lookup) -> ClamAV scan
               -> olevba (macros) -> unzip OOXML -> scan XML for links/objects
               -> report.txt

The only step that touches the internet is the VirusTotal lookup, and
it sends **only the SHA-256 hash**, never the document itself.

================================================================

## Setup / Installation

### Step 1 — Open Terminal

Press Cmd+Space, type `Terminal`, press Enter. All commands below are
typed into this window, one line at a time, pressing Enter after each.

### Step 2 — Install Homebrew (skip if already installed)

Homebrew is the package manager that installs the other tools. Check if
it is already there:

```
brew --version          # if you see a version number, skip to Step 3
```

If that says "command not found", install it:

```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After it finishes, close and reopen Terminal, then run `brew --version`
again to confirm.

### Step 3 — Install the required tools

```
brew install clamav ripgrep vt-cli          # AV engine, fast search, VirusTotal CLI
python3 -m pip install --user oletools       # macro/embedded-object scanner (olevba, oleobj)
```

`unzip`, `shasum`, `md5`, `file`, and `xattr` already ship with macOS —
nothing to install for those.

Confirm each tool is visible:

```
clamscan --version      # expect: ClamAV 1.x
rg --version            # expect: ripgrep x.x
vt version              # expect: a version string
olevba --version        # expect: olevba x.x
```

If `olevba` says "command not found", your pip user-scripts folder isn't
on PATH. Add it (Apple Silicon path shown), then reopen Terminal:

```
echo 'export PATH="$HOME/Library/Python/3.9/bin:$PATH"' >> ~/.zprofile
```

(The `3.9` may differ — run `ls ~/Library/Python` to see your version.)

### Step 4 — Download the ClamAV virus definitions

ClamAV installs with an **empty** database and will error until you
download definitions. Create its config, remove the block line, then
fetch the database:

```
cp "$(brew --prefix)/etc/clamav/freshclam.conf.sample" "$(brew --prefix)/etc/clamav/freshclam.conf"   # create config
sed -i '' '/^Example/d' "$(brew --prefix)/etc/clamav/freshclam.conf"                                   # remove the block line
freshclam                                                                                              # download DB (~300 MB, takes a minute)
```

Confirm the definitions landed:

```
ls "$(brew --prefix)/var/lib/clamav"    # expect: main.cvd, daily.cld/.cvd, bytecode.cvd
```

Re-run `freshclam` before each triage session, or run it on a schedule,
to keep definitions current.

### Step 5 — Set up your VirusTotal API key

Get a free API key from https://www.virustotal.com (sign in → profile →
API key), then:

```
vt init                 # paste the API key when prompted
```

Free tier allows 4 lookups per minute — fine for one-off triage.

### Step 6 — Make the script runnable

Clone this repository (or copy `office_malware_check.sh`) somewhere
permanent (e.g. `~/tools/`), then mark it executable:

```
chmod +x ~/tools/office_malware_check.sh    # allow it to run
```

### Step 7 — Test run

Point it at any harmless Office file to confirm everything works:

```
~/tools/office_malware_check.sh ~/Documents/some-normal.docx
```

You should see sections scroll by (INTAKE, BASIC IDENTIFICATION,
VIRUSTOTAL, CLAMAV, OLETOOLS...) ending with `DONE. Report: ...`.
Open the report folder it names to read the full output.

================================================================

## Operations (Day-2)

### Running it on a suspect file

```
cd ~/Desktop                                  # go where you want the report saved
~/tools/office_malware_check.sh /path/to/suspicious.docx
```

The report folder is created in whatever directory you're currently in.

### Reading the result

Open the `report.txt` it points to and check the SUMMARY FLAGS section
at the bottom. Escalate if you see any of:
- `AutoOpen` / `Document_Open` / `Workbook_Open` in the olevba output
- Suspicious keywords: Shell, CreateObject, powershell, curl, wget, base64
- `TargetMode="External"` in a `.rels` file
- Unexpected URLs or UNC paths (`\\server\...`)
- A VirusTotal `malicious` count above 0

## Troubleshooting

### Symptom: `LibClamAV Error: cli_loaddbdir: No supported database files found`
- Cause: virus definitions never downloaded.
- Fix: run Step 4. Verify with `ls "$(brew --prefix)/var/lib/clamav"`.

### Symptom: VirusTotal step says "not found" or 404
- Cause: VirusTotal has never seen that file's hash. This is normal for
  a brand-new or internal document — it is not an error.
- Note: absence from VT is not a clean bill of health; keep reading the
  macro and link sections of the report.

### Symptom: `vt` errors with an auth/key message
- Cause: API key not set.
- Fix: run `vt init` (Step 5) and paste your key.

### Symptom: `olevba: command not found`
- Cause: pip user-scripts folder not on PATH.
- Fix: run the PATH line in Step 3, then close and reopen Terminal.

### Symptom: `Permission denied` when running the script
- Cause: not marked executable.
- Fix: `chmod +x ~/tools/office_malware_check.sh` (Step 6).

### Symptom: `ripgrep (rg) not found` in the report
- Cause: ripgrep missing; the external-link scan is skipped.
- Fix: `brew install ripgrep`.

## Security Notes

- The VirusTotal step sends the SHA-256 hash only — never the document.
  Do not switch it to `vt file scan <path>` (which uploads the file) for
  confidential documents.
- Run triage on a non-privileged account. The script only reads the
  suspect file; it never executes it.
- Keep ClamAV definitions fresh (`freshclam`) or scans give false
  "clean" results against new threats.

## References

- ClamAV docs: https://docs.clamav.net
- oletools (olevba/oleobj): https://github.com/decalage2/oletools
- VirusTotal CLI: https://github.com/VirusTotal/vt-cli
- ripgrep: https://github.com/BurntSushi/ripgrep

## Disclaimer

This script is provided as-is for triage assistance only. A clean
report is not proof that a document is safe. Handle suspect files in an
isolated environment and follow your own incident response process.

## License

MIT — see `LICENSE`.

---
Review after any tool or macOS major upgrade.

# CLI Security Tools — Quick Start Guide
*Covers all tools from the App Audit "Install via Homebrew/pip" section*

---

## Install Everything First

```bash
brew install nuclei trivy syft grype trufflehog gitleaks testssl zeek hashcat lynis
pip3 install volatility3
```

---

## Nuclei — Vulnerability Scanner
**What it does:** Sends targeted HTTP/network probes based on community templates. Finds CVEs, exposed admin panels, misconfigs, default credentials.

```bash
# Update templates (do this first, then weekly)
nuclei -update-templates

# Scan a single host
nuclei -u https://target.example.com

# Scan for critical/high severity only
nuclei -u https://target.example.com -s critical,high

# Scan a list of hosts
nuclei -l hosts.txt -s critical,high -o results.txt

# Scan internal network (combine with Nmap output)
nmap -iL targets.txt -oG - | awk '/open/{print $2}' > live.txt
nuclei -l live.txt -t network/
```

**Template categories:** `cves/`, `exposures/`, `misconfiguration/`, `default-logins/`, `network/`
Templates live at: `~/.local/nuclei-templates/`

---

## Trivy — Container & Code Scanner
**What it does:** Scans Docker images, filesystems, git repos, and IaC configs for CVEs and secrets.

```bash
# Scan a Docker image
trivy image nginx:latest

# Scan only critical CVEs
trivy image --severity CRITICAL,HIGH nginx:latest

# Scan a local directory (filesystem mode)
trivy fs /path/to/project

# Scan a git repo
trivy repo https://github.com/org/repo

# Scan Infrastructure-as-Code (Terraform, Helm, etc.)
trivy config ./terraform/

# Output as JSON for piping
trivy image --format json nginx:latest | jq '.Results[].Vulnerabilities[]'
```

---

## Syft — SBOM Generator
**What it does:** Generates a Software Bill of Materials (SBOM) — an inventory of every package inside an image or directory.

```bash
# Generate SBOM for a Docker image
syft nginx:latest

# Generate SBOM for a local directory
syft dir:/path/to/project

# Output as SPDX JSON (standard format)
syft nginx:latest -o spdx-json > sbom.json

# Output as CycloneDX
syft nginx:latest -o cyclonedx-json > sbom-cdx.json
```

Feed the SBOM output directly into Grype for vulnerability matching.

---

## Grype — Vulnerability Scanner (SBOM-aware)
**What it does:** Matches packages from Syft SBOMs (or scans directly) against CVE databases.

```bash
# Scan a Docker image directly
grype nginx:latest

# Scan a Syft SBOM file
grype sbom:./sbom.json

# Pipe Syft output directly into Grype
syft nginx:latest -o json | grype

# Show only fixable vulnerabilities
grype nginx:latest --ignore-states wont-fix

# Update the vulnerability database
grype db update
```

---

## Trufflehog — Secrets Scanner
**What it does:** Finds API keys, tokens, and credentials in git history, S3 buckets, filesystems, and more. Verifies secrets are live before alerting.

```bash
# Scan a local git repo (scans full history)
trufflehog git file://./path/to/repo

# Scan a remote GitHub repo
trufflehog github --repo https://github.com/org/repo

# Scan an entire GitHub org
trufflehog github --org orgname

# Scan a filesystem directory
trufflehog filesystem /path/to/directory

# Scan an S3 bucket
trufflehog s3 --bucket mybucket

# Only report verified (live) secrets
trufflehog git file://. --only-verified

# JSON output
trufflehog git file://. --json
```

---

## Gitleaks — Git Secrets Scanner
**What it does:** Faster, simpler secrets scanner focused on git repos. Good for pre-commit hooks and CI.

```bash
# Scan current git repo
gitleaks detect

# Scan a specific path
gitleaks detect --source /path/to/repo

# Scan git history (all commits)
gitleaks detect --log-opts="--all"

# Output report as JSON
gitleaks detect --report-path report.json --report-format json

# Install as a pre-commit hook (run once per repo)
gitleaks protect --staged -v

# Scan a specific commit range
gitleaks detect --log-opts="HEAD~10..HEAD"
```

**Trufflehog vs Gitleaks:** Trufflehog verifies secrets are live; Gitleaks is faster and better for CI hooks. Use both.

---

## testssl.sh — TLS Auditor
**What it does:** Tests any HTTPS endpoint for weak ciphers, protocol versions, certificate issues, HSTS, HPKP, and more.

```bash
# Full audit of a host
testssl https://target.example.com

# Check certificate only
testssl --certinfo https://target.example.com

# Check for known vulnerabilities (BEAST, POODLE, Heartbleed, etc.)
testssl --vulnerable https://target.example.com

# Check cipher support
testssl --cipher-per-proto https://target.example.com

# Scan all standard HTTPS ports on a host
testssl --starttls smtp mail.example.com:587

# Save HTML report
testssl --htmlfile report.html https://target.example.com
```

---

## Zeek — Network Analysis Framework
**What it does:** Parses live traffic or PCAP files into structured logs (conn.log, dns.log, http.log, ssl.log, files.log, etc.). Feeds into Brim (already installed) or ELK.

```bash
# Process an existing PCAP file
zeek -r capture.pcap

# Run on live interface (requires sudo)
sudo zeek -i en0

# Load standard policy scripts (recommended)
zeek -r capture.pcap local

# Output goes to current directory as .log files
ls *.log
# conn.log  dns.log  http.log  ssl.log  files.log  weird.log ...

# Open logs in Brim (already installed) for visual exploration
# Drag any .log file into Brim, or:
# Brim also accepts .pcap files and runs Zeek automatically

# Extract all HTTP requests from a PCAP
zeek -r capture.pcap && cat http.log | zeek-cut id.orig_h method host uri

# Find all DNS queries
cat dns.log | zeek-cut ts id.orig_h query qtype_name answers
```

**Key log files:**
- `conn.log` — all connections (IP, port, duration, bytes)
- `dns.log` — all DNS queries and responses
- `http.log` — HTTP requests (even inside TLS if you have keys)
- `ssl.log` — TLS handshakes and certificate details
- `files.log` — file transfers and hashes
- `weird.log` — protocol anomalies (start here for threat hunting)

---

## Hashcat — Password / Hash Cracker
**What it does:** GPU-accelerated hash cracking. Runs on Apple Silicon via Metal.

```bash
# Identify hash type first (use haiti or hashid)
brew install haiti
haiti '$2y$10$...'   # → bcrypt

# Basic dictionary attack (mode -a 0)
hashcat -a 0 -m 0 hashes.txt /usr/share/wordlists/rockyou.txt

# Common hash modes:
# -m 0    = MD5
# -m 100  = SHA1
# -m 1800 = sha512crypt (Linux /etc/shadow)
# -m 3200 = bcrypt
# -m 1000 = NTLM (Windows)
# -m 22000 = WPA2 (wifi)

# Rule-based attack (adds mutations to wordlist)
hashcat -a 0 -m 1000 hashes.txt rockyou.txt -r rules/best64.rule

# Brute force (mask attack, mode -a 3)
# ?u=uppercase ?l=lowercase ?d=digit ?s=symbol
hashcat -a 3 -m 0 hashes.txt ?u?l?l?l?d?d?d?s

# Crack WPA2 handshake (capture with Wireshark/Airtool 2 first)
hashcat -a 0 -m 22000 handshake.hccapx rockyou.txt

# Show cracked results
hashcat --show hashes.txt
```

**Apple Silicon note:** Hashcat uses Metal GPU acceleration automatically on M-series chips. Significantly faster than CPU-only mode.

---

## Volatility 3 — Memory Forensics
**What it does:** Analyzes RAM dumps for running processes, network connections, injected code, registry hives, credentials, and malware artifacts.

### Acquire a memory image first
- **macOS:** `osxpmem` (https://github.com/google/rekall) or via Parallels/UTM snapshot export
- **Windows VM:** Use DumpIt, WinPmem, or Magnet RAM Capture inside the VM
- **Linux:** `/dev/mem` or LiME kernel module

```bash
# List available plugins for a given OS
vol -f memory.dmp windows.info         # confirm OS/architecture
vol -f memory.dmp mac.pslist           # macOS process list

# ── Windows Plugins ────────────────────────────────────────────
# Running processes
vol -f memory.dmp windows.pslist
vol -f memory.dmp windows.pstree       # tree view showing parent/child

# Network connections
vol -f memory.dmp windows.netstat

# Loaded DLLs per process
vol -f memory.dmp windows.dlllist

# Find injected code (hollow processes, shellcode)
vol -f memory.dmp windows.malfind

# Dump a suspicious process by PID
vol -f memory.dmp windows.dumpfiles --pid 1234

# Registry hives and keys
vol -f memory.dmp windows.registry.hivelist
vol -f memory.dmp windows.registry.printkey --key "SOFTWARE\Microsoft\Windows\CurrentVersion\Run"

# Extract browser history/credentials
vol -f memory.dmp windows.hashdump     # SAM hashes

# ── macOS Plugins ─────────────────────────────────────────────
vol -f memory.dmp mac.pslist
vol -f memory.dmp mac.netstat
vol -f memory.dmp mac.malfind
vol -f memory.dmp mac.bash             # bash history from memory

# ── Linux Plugins ─────────────────────────────────────────────
vol -f memory.dmp linux.pslist
vol -f memory.dmp linux.bash
vol -f memory.dmp linux.netstat
```

### Typical IR workflow
```bash
# 1. Confirm image is readable
vol -f memory.dmp windows.info

# 2. Get process list — look for unusual names, missing parents, or cmd.exe spawned by Word
vol -f memory.dmp windows.pstree | tee processes.txt

# 3. Check network connections — look for unexpected outbound on 443/80/4444
vol -f memory.dmp windows.netstat | tee netconns.txt

# 4. Run malfind — flags regions with RWX permissions and MZ headers (injected PEs)
vol -f memory.dmp windows.malfind | tee malfind.txt

# 5. Dump suspicious PIDs for static analysis in Ghidra/Binary Ninja
vol -f memory.dmp windows.dumpfiles --pid <suspicious_pid>
```

---

## Lynis — System Hardening Auditor
**What it does:** Audits your macOS or Linux system against CIS/NIST benchmarks. Produces a scored report with specific remediation steps.

```bash
# Full system audit (sudo for complete results)
sudo lynis audit system

# Audit without colors (for logging)
sudo lynis audit system --no-colors | tee lynis-$(date +%F).txt

# Quick scan (no pauses)
sudo lynis audit system --quick

# Audit only specific category
sudo lynis audit system --tests-from-group authentication
sudo lynis audit system --tests-from-group malware
sudo lynis audit system --tests-from-group networking
sudo lynis audit system --tests-from-group storage

# Compare two audits (baseline vs. current)
# Run with --report-file to save
sudo lynis audit system --report-file /tmp/lynis-baseline.dat

# Show hardening index score only
sudo lynis audit system --quick | grep "Hardening index"
```

**Reading the output:**
- `[OK]` — passes the check
- `[WARNING]` — potential issue, review
- `[SUGGESTION]` — hardening opportunity
- **Hardening index: XX/100** — your overall score

Aim for 80+ on a production system. Fresh macOS typically scores 55–65.

---

*Quick Start Guide | Generated June 27, 2026*

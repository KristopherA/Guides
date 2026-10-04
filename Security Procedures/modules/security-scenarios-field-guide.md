# Incident Response & Security Scenarios — Field Guide
*Quick-reference commands for common sysadmin and security scenarios*

---

## Scenario Index
1. [Suspicious Process / Malware on Endpoint](#1-suspicious-process--malware-on-endpoint)
2. [Unusual Network Traffic / Beaconing](#2-unusual-network-traffic--beaconing)
3. [Web Application Attack](#3-web-application-attack)
4. [Exposed Credentials / Secrets in Code](#4-exposed-credentials--secrets-in-code)
5. [Compromised User Account](#5-compromised-user-account)
6. [Ransomware Response](#6-ransomware-response)
7. [Phishing Email Investigation](#7-phishing-email-investigation)
8. [Container / Supply Chain Compromise](#8-container--supply-chain-compromise)
9. [Wi-Fi / Wireless Attack](#9-wi-fi--wireless-attack)
10. [macOS Persistence Investigation](#10-macos-persistence-investigation)
11. [Memory Forensics — Live or Post-Mortem](#11-memory-forensics--live-or-post-mortem)
12. [Vulnerability Scan Before / After Patching](#12-vulnerability-scan-before--after-patching)
13. [TLS / Certificate Issue](#13-tls--certificate-issue)
14. [System Hardening Audit](#14-system-hardening-audit)

---

## 1. Suspicious Process / Malware on Endpoint

### Tools: TaskExplorer, Red Canary Mac Monitor, KnockKnock, FileMonitor, Ghidra, die.app, Malcat

**Step 1 — Identify what's running**
```bash
# List all processes with open network connections (macOS built-in)
lsof -i -n -P | grep ESTABLISHED

# Show process with its parent, command line, and code signing status
ps aux | grep <suspicious_name>
codesign -dvvv /path/to/binary 2>&1   # check signing identity
spctl -a -vv /path/to/binary          # check Gatekeeper assessment
```
*Example: You see `com.apple.mdworker_shared` making outbound connections to 185.x.x.x — pull the binary path from lsof and verify the signing identity.*

**Step 2 — Check file identity before opening**
```bash
# Identify packer, language, and format (die.app or CLI)
diec /path/to/suspicious_binary

# Get file hashes for VirusTotal lookup
shasum -a 256 /path/to/suspicious_binary
md5 /path/to/suspicious_binary
```
*Example: `diec` shows "Packed: UPX" — immediately suspicious for a macOS binary. Submit SHA256 to VirusTotal before detonating.*

**Step 3 — Check persistence (use KnockKnock GUI or CLI)**
```bash
# KnockKnock CLI scan — dumps all persistence items with signing info
/Applications/KnockKnock.app/Contents/MacOS/KnockKnock -json | jq '.[]'

# Manual persistence check — common locations
ls -la ~/Library/LaunchAgents/
ls -la /Library/LaunchAgents/
ls -la /Library/LaunchDaemons/
ls -la /Library/Application\ Support/*/
```
*Example: KnockKnock shows an unsigned LaunchAgent pointing to `/tmp/.cache/updater` — that binary doesn't exist on a clean system.*

**Step 4 — Static analysis if needed**
Open in Ghidra or Binary Ninja. For quick triage:
```bash
# Strings from binary
strings /path/to/binary | grep -E "(http|https|/tmp|passwd|exec|curl|wget)"

# Check linked libraries
otool -L /path/to/binary

# Check for embedded IP/domain patterns
strings /path/to/binary | grep -E "\b([0-9]{1,3}\.){3}[0-9]{1,3}\b"
```

---

## 2. Unusual Network Traffic / Beaconing

### Tools: Wireshark, Zeek, Brim, Stratoshark, Little Snitch, Sniffnet, Netstat

**Step 1 — Capture traffic**
```bash
# Capture on primary interface (replace en0 with your interface)
sudo tcpdump -i en0 -w /tmp/capture-$(date +%F-%H%M).pcap

# Capture only traffic to/from a suspicious host
sudo tcpdump -i en0 host 185.x.x.x -w /tmp/suspect.pcap

# Capture DNS only (detect C2 via DNS)
sudo tcpdump -i en0 port 53 -w /tmp/dns.pcap
```

**Step 2 — Analyse with Zeek (produces structured logs)**
```bash
zeek -r /tmp/capture.pcap local

# Connections sorted by bytes (exfiltration candidates)
cat conn.log | zeek-cut id.orig_h id.resp_h id.resp_p proto orig_bytes | sort -k5 -rn | head -20

# DNS queries to unusual TLDs or long subdomains (DNS tunneling)
cat dns.log | zeek-cut ts query | awk 'length($2) > 40'

# All unique external IPs contacted
cat conn.log | zeek-cut id.resp_h | sort -u | grep -v "^10\.\|^192\.168\.\|^172\."
```
*Example: You see the same host beaconing to `telemetry.updates-cdn.net` every 60 seconds with a 200-byte payload — classic C2 heartbeat. Pull the domain from dns.log and check IP on VirusTotal.*

**Step 3 — Open in Brim for visual investigation**
Drag the `.pcap` or Zeek `.log` files into Brim. Use the query bar:
```
# Brim ZQL queries
_path="conn" | count() by id.resp_h | sort -r count   # top talkers
_path="dns" | count() by query | sort -r count         # top DNS queries
_path="http" | uri contains ".exe" or uri contains ".ps1"  # suspicious downloads
_path="ssl" | server_name != "" | count() by server_name | sort -r count
```

*Example: A workstation is uploading to `storage.googleapis.com` with no user activity — Brim's conn.log shows 400MB out over 2 hours.*

---

## 3. Web Application Attack

### Tools: Burp Suite / Caido (proxy), ZAP (scanner), Wireshark, testssl, Zeek

**Reconnaissance — scan target before engagement**
```bash
# Full vulnerability scan with Nuclei
nuclei -u https://target.example.com -s critical,high

# TLS audit — check for weak ciphers, expired certs
testssl https://target.example.com

# Tech stack fingerprint
nuclei -u https://target.example.com -t technologies/
```
*Example: Nuclei hits `CVE-2023-44487` (HTTP/2 Rapid Reset) on an nginx instance — patch or mitigate before going further.*

**Proxy-based testing (Burp Suite / Caido)**
1. Set browser proxy to `127.0.0.1:8080`
2. Browse target to populate sitemap
3. Send interesting requests to Repeater for manual manipulation

```
# Common manual checks in Repeater/Replay:
- SQLi: Add ' or 1=1-- to input fields
- IDOR: Change /api/user/1234 → /api/user/1235
- Auth bypass: Remove/modify Authorization header
- SSTI: Input {{7*7}} — if response contains 49, template injection confirmed
```

**Automated scan with ZAP**
```bash
# Headless scan with report output
docker run -v $(pwd):/zap/wrk/:rw ghcr.io/zaproxy/zaproxy:stable \
  zap-baseline.py -t https://target.example.com \
  -r zap-report.html -J zap-report.json
```
*Example: ZAP flags a reflected XSS in the search parameter — `?q=<script>alert(1)</script>` returns unescaped.*

**Investigating an attack after the fact**
```bash
# Parse web server access logs for SQLi patterns
grep -E "('|--|%27|%3D|UNION|SELECT|INSERT|DROP)" /var/log/nginx/access.log

# Spike in 4xx/5xx — possible scanning or brute force
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn

# IPs making > 100 requests/min
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20
```

---

## 4. Exposed Credentials / Secrets in Code

### Tools: Trufflehog, Gitleaks, 1Password

**Scan a repo for secrets**
```bash
# Scan local repo (full git history)
trufflehog git file://. --only-verified

# Scan remote repo without cloning
trufflehog github --repo https://github.com/org/repo --only-verified

# Scan entire GitHub org
trufflehog github --org your-org-name --only-verified

# JSON output for piping into a ticket/SIEM
trufflehog git file://. --only-verified --json | jq '{file:.SourceMetadata.Data.Git.file, line:.SourceMetadata.Data.Git.line, type:.DetectorName}'
```
*Example: Trufflehog finds a live AWS key in a commit from 6 months ago, even though it was deleted in a later commit — git history preserves it. Revoke the key immediately.*

**Pre-commit hook — catch secrets before they land**
```bash
# Install Gitleaks hook in any repo (run once)
gitleaks protect --staged -v

# Add to .git/hooks/pre-commit for automatic blocking
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/sh
gitleaks protect --staged -v
EOF
chmod +x .git/hooks/pre-commit
```

**After finding a secret — triage checklist**
```bash
# 1. Is it still active? (Trufflehog --only-verified confirms this)
# 2. When was it committed?
git log --all -S "AKIA" --oneline    # search for AWS key prefix in history

# 3. Who has cloned this repo?
# Check GitHub → Insights → Traffic (requires repo access)

# 4. Revoke immediately, then rotate dependent services
# 5. Purge from git history (only after revoking — don't rely on this alone)
git filter-repo --path secrets.txt --invert-paths
```

---

## 5. Compromised User Account

### Tools: AD Check, Little Snitch, 1Password, lsof, last

**Immediate triage**
```bash
# Who is logged in right now
w
last | head -20

# Check for unexpected sudo usage
grep sudo /var/log/auth.log | tail -50          # Linux
log show --predicate 'process == "sudo"' --last 1h  # macOS

# Active network connections from user processes
lsof -i -u username

# Check SSH authorized_keys for backdoors
cat ~/.ssh/authorized_keys
cat /root/.ssh/authorized_keys
```
*Example: `last` shows a login from an unfamiliar IP at 3am. Cross-reference with Little Snitch logs to see what that session accessed.*

**macOS-specific account checks**
```bash
# List all local users
dscl . list /Users | grep -v "^_"

# Check if any user has unexpected admin rights
dscl . read /Groups/admin GroupMembership

# Recent file access by user (last 24h)
find /Users/username -newer /tmp/24h-marker -ls 2>/dev/null

# Audit Active Directory account status
# Use AD Check.app GUI or:
id username@domain
```

**Contain: lock the account**
```bash
# macOS — disable local user
sudo dscl . change /Users/compromised_user UserShell /bin/bash /usr/bin/false

# Active Directory (requires AD admin rights)
# In AD Check.app → search user → Disable Account
```

---

## 6. Ransomware Response

### First 60 seconds — contain before investigating

```bash
# 1. Isolate — kill network access immediately
sudo ifconfig en0 down
# Or use Little Snitch: switch to "Silent Mode - Deny All Connections"

# 2. Identify the encrypting process
ps aux | grep -E "(crypt|cipher|lock|ransom)"
lsof | grep "\.encrypted\|\.locked\|\.ransom" | head -20

# 3. Kill the process (get PID from above)
sudo kill -9 <PID>

# 4. Snapshot the system state before anything else
sudo fs_usage > /tmp/fs_snapshot.txt &
sudo tcpdump -i en0 -w /tmp/ransom-traffic.pcap &
```

**Preserve evidence**
```bash
# Capture memory image (if tools available — do this FIRST)
# Use osxpmem or a VM snapshot

# Document encrypted file extensions
find / -name "*.encrypted" -o -name "*.locked" -o -name "*.ransom" 2>/dev/null | head -50

# Find ransom note
find / -name "README*.txt" -o -name "HOW_TO_DECRYPT*" -o -name "RESTORE_FILES*" 2>/dev/null

# Check what was modified in the last hour
find / -newer /tmp/1h-marker -type f 2>/dev/null | grep -v proc | head -100
```

**Identify the strain (use die.app + Malcat + VirusTotal)**
```bash
shasum -a 256 /path/to/ransomware_binary
# Submit hash to: https://www.virustotal.com
# Check: https://id-ransomware.malwarehunterteam.com (upload ransom note or encrypted file)
```

---

## 7. Phishing Email Investigation

### Tools: TextSniper (extract text from screenshots), BBEdit, Wireshark, Caido/Burp

**Extract and analyse the email**
```bash
# If you have the raw .eml file:
# Parse headers (look for SPF/DKIM/DMARC failures, relay hops)
cat suspicious.eml | grep -E "^(Received|From|To|Subject|X-Originating-IP|Authentication-Results):"

# Check SPF/DKIM from command line
dig TXT sender-domain.com | grep "v=spf"
dig TXT _dmarc.sender-domain.com

# Extract all URLs from email body
cat suspicious.eml | grep -oE "(https?://[^\"' >]+)" | sort -u
```
*Example: Email claims to be from `@your-org.example` but `Authentication-Results` shows `spf=fail` — spoofed sender.*

**Investigate a suspicious link safely**
```bash
# Expand shortened URLs without visiting
curl -sI "https://bit.ly/xxxxx" | grep -i location

# Fetch headers only (no payload) to fingerprint the server
curl -sI "https://suspicious-link.com" | head -20

# Use Caido or Burp with interception — visit through proxy to capture full request/response
# Set proxy → navigate → review in Caido's History tab

# Screenshot the page without running JS
# Use: curl -s https://suspicious-url.com | grep -E "(title|form|input|action)"
```

**Check attachments**
```bash
# Hash and check before opening
shasum -a 256 attachment.docx
# Submit to VirusTotal: https://www.virustotal.com

# Check for macros in Office docs (without opening)
olevba attachment.docx          # install: pip3 install oletools
mraptor attachment.docx         # flags macro risk level

# For PDFs — check with die.app, then:
pdfinfo attachment.pdf          # install: brew install poppler
strings attachment.pdf | grep -E "(JavaScript|JS|OpenAction|Launch|URI)"
```

---

## 8. Container / Supply Chain Compromise

### Tools: Trivy, Syft, Grype, Trufflehog, Docker

**Audit an image before deployment**
```bash
# Full CVE scan
trivy image your-org/app:latest

# Generate SBOM — know every package in the image
syft your-org/app:latest -o json > sbom.json

# Match SBOM against CVE database
grype sbom:./sbom.json

# Scan for secrets baked into the image layers
trufflehog docker --image your-org/app:latest --only-verified
```
*Example: `trivy image` flags a critical CVE in an outdated `log4j` jar inside a vendor image. The vendor's public tag is "latest" but contains a 2-year-old base layer.*

**Investigate a running container**
```bash
# What's actually running inside
docker exec -it <container_id> ps aux
docker exec -it <container_id> netstat -tulnp

# Diff the container filesystem against its image (shows what changed at runtime)
docker diff <container_id>

# Inspect image history — look for suspicious RUN commands
docker history --no-trunc your-org/app:latest

# Export running container filesystem for analysis
docker export <container_id> > container-fs.tar
tar tf container-fs.tar | grep -E "(cron|bashrc|profile|ssh|passwd)"
```

**Scan IaC configs**
```bash
# Terraform, Helm, Kubernetes manifests
trivy config ./terraform/
trivy config ./k8s/

# Common findings: open security groups, no encryption at rest,
# privileged containers, capabilities like NET_ADMIN
```

---

## 9. Wi-Fi / Wireless Attack

### Tools: WiFi Explorer Pro 3, Airtool 2, Wireshark, EAPTest, WireGuard

**Rogue AP / Evil Twin detection**
```bash
# WiFi Explorer Pro GUI: look for:
# - Two SSIDs with identical names but different BSSIDs
# - Your known SSID on an unexpected channel
# - APs with unusually strong signal appearing suddenly

# CLI: scan for all BSSIDs advertising your SSID
/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -s | grep "YourSSID"
```

**Capture and analyse Wi-Fi traffic (Airtool 2 → Wireshark)**
1. Open Airtool 2 → select channel → Start Capture
2. Open `.pcap` in Wireshark

```
# Wireshark display filters for wireless investigation:
wlan.fc.type_subtype == 0x08          # Beacon frames (all APs)
wlan.ssid == "TargetSSID"             # Specific SSID
eapol                                  # WPA handshake frames
wlan.fc.type_subtype == 0x0b          # Authentication frames
wlan.fc.type_subtype == 0x0c          # Deauthentication (deauth attack)
```
*Example: Filter `wlan.fc.type_subtype == 0x0c` shows a flood of deauth frames from a spoofed BSSID — a deauth attack forcing clients to reconnect to a rogue AP.*

**Test 802.1X/EAP configuration**
Use EAPTest.app to:
- Verify PEAP/EAP-TLS is enforcing server certificate validation
- Test that clients reject rogue RADIUS servers
- Confirm EAP method negotiation

**Protect your own traffic on untrusted Wi-Fi**
```bash
# Bring up WireGuard tunnel immediately on connection to unknown network
wg-quick up wg0

# Verify tunnel is active
wg show
curl https://ifconfig.me   # should return VPN exit IP
```

---

## 10. macOS Persistence Investigation

### Tools: KnockKnock, FileMonitor, ProcessMonitor, Red Canary Mac Monitor, PlistEdit Pro

**Full persistence sweep**
```bash
# KnockKnock JSON dump — all persistence mechanisms
/Applications/KnockKnock.app/Contents/MacOS/KnockKnock -json | \
  jq '.[] | select(.signature.status != "Apple") | {name:.name, path:.path, signed:.signature.status}'

# Manual sweep — all LaunchAgent/Daemon plists
for dir in ~/Library/LaunchAgents /Library/LaunchAgents /Library/LaunchDaemons /System/Library/LaunchDaemons; do
  echo "=== $dir ==="
  ls -la "$dir" 2>/dev/null
done

# Login items (newer macOS)
osascript -e 'tell application "System Events" to get the name of every login item'
sfltool dump | grep -A3 "BackgroundItems"

# Cron jobs
crontab -l
sudo crontab -l
ls /etc/cron* 2>/dev/null
```

**Investigate a specific plist**
```bash
# Read LaunchAgent plist (or open in PlistEdit Pro)
plutil -p ~/Library/LaunchAgents/com.suspicious.plist

# Key fields to check:
# ProgramArguments — what binary/command runs
# RunAtLoad — runs on login
# StartInterval — runs on schedule (note the interval in seconds)
# WatchPaths / QueueDirectories — triggers on file change
```
*Example: A plist has `RunAtLoad=true` pointing to `/tmp/.helper` which doesn't exist yet — it's a dropper waiting to download the real payload.*

**Monitor for new persistence in real-time**
```bash
# FileMonitor — watch LaunchAgents directory for new files
# Open FileMonitor.app → filter path contains "LaunchAgents"

# Or use built-in FSEvents
fswatch ~/Library/LaunchAgents /Library/LaunchAgents /Library/LaunchDaemons | while read path; do
  echo "[ALERT] Change detected: $path"
  plutil -p "$path" 2>/dev/null
done
```

---

## 11. Memory Forensics — Live or Post-Mortem

### Tools: Volatility 3, Autopsy, TaskExplorer

**Acquire memory (macOS VM in Parallels/UTM)**
```bash
# Take a snapshot in Parallels — freeze VM state, then access .mem file
# In UTM: Machine → Pause → the .utm bundle contains the memory state

# For a Windows VM, use WinPmem inside the VM:
winpmem_mini_x64.exe memory.dmp
```

**Quick IR triage with Volatility 3**
```bash
# Verify image is readable
vol -f memory.dmp windows.info

# --- Process Analysis ---
# Full process tree (spot orphaned or injected processes)
vol -f memory.dmp windows.pstree

# Processes with no parent or unusual parent (e.g. Word spawning cmd.exe)
vol -f memory.dmp windows.pstree | grep -E "(cmd.exe|powershell|wscript|mshta)"

# --- Network ---
# All connections at time of capture
vol -f memory.dmp windows.netstat

# --- Injection Detection ---
# Regions with RWX permissions containing PE headers = injected code
vol -f memory.dmp windows.malfind | tee malfind.txt

# Dump suspicious process for static analysis
vol -f memory.dmp windows.dumpfiles --pid <PID>
# Then open dumped file in Ghidra or submit hash to VirusTotal

# --- Credentials ---
vol -f memory.dmp windows.hashdump      # SAM hashes
vol -f memory.dmp windows.lsadump       # LSA secrets
```
*Example: `windows.pstree` shows `WINWORD.EXE` → `cmd.exe` → `powershell.exe -enc <base64>` — a macro-based payload. Dump the powershell process and decode the base64.*

```bash
# Decode PowerShell encoded command from memory strings
echo "<base64string>" | base64 -d | iconv -f UTF-16LE -t UTF-8
```

---

## 12. Vulnerability Scan Before / After Patching

### Tools: Nuclei, Zenmap, testssl, Trivy

**Before patching — establish baseline**
```bash
# Network host discovery
nmap -sn 192.168.1.0/24 -oG hosts-up.txt

# Port scan live hosts
nmap -sV -sC -iL hosts-up.txt -oN baseline-$(date +%F).txt

# Vulnerability scan with Nuclei
nuclei -l hosts-up.txt -s critical,high,medium -o nuclei-baseline-$(date +%F).txt

# TLS audit of all HTTPS services
cat hosts-up.txt | grep "443" | awk '{print $2}' | while read host; do
  testssl --quiet https://$host >> tls-baseline.txt
done
```

**After patching — diff the results**
```bash
nuclei -l hosts-up.txt -s critical,high,medium -o nuclei-post-$(date +%F).txt

# Compare — what's been fixed, what's new
diff nuclei-baseline-2026-06-27.txt nuclei-post-2026-06-28.txt

# Container images — before/after
grype image:old-tag > grype-before.txt
grype image:new-tag > grype-after.txt
diff grype-before.txt grype-after.txt | grep "^[<>]"
```

---

## 13. TLS / Certificate Issue

### Tools: testssl, Wireshark, Caido/Burp

**Diagnose TLS problems**
```bash
# Full TLS audit — covers everything
testssl https://target.example.com

# Check certificate expiry only (fast)
testssl --certinfo https://target.example.com | grep -E "(expires|Not After|days)"

# Check for specific vulnerabilities
testssl --vulnerable https://target.example.com

# Test STARTTLS services (SMTP, IMAP, etc.)
testssl --starttls smtp mail.example.com:587
testssl --starttls imap mail.example.com:143

# Quick cert check via openssl (no testssl needed)
echo | openssl s_client -connect target.example.com:443 2>/dev/null | openssl x509 -noout -dates -subject -issuer
```
*Example: testssl shows `LUCKY13` and `RC4` cipher support still enabled on a server — configuration issue, not a patch issue. Shows exact cipher strings to remove from nginx/Apache config.*

**Capture and inspect TLS in Wireshark**
```
# If you have the server's private key (lab/test env):
# Wireshark → Preferences → Protocols → TLS → RSA Keys → add key
# Then filter: tls && http   (decrypted HTTP inside TLS)

# Without the key — inspect handshake metadata
tls.handshake.type == 1    # ClientHello (what the client offers)
tls.handshake.type == 2    # ServerHello (what the server chose)
tls.handshake.ciphersuite  # filter by specific cipher
```

---

## 14. System Hardening Audit

### Tools: Lynis, Little Snitch, KnockKnock, Nuclei

**Full Lynis audit**
```bash
# Run and save output
sudo lynis audit system --quick --no-colors | tee lynis-$(date +%F).txt

# Key sections to review in output:
grep -A2 "\[WARNING\]" lynis-$(date +%F).txt
grep -A2 "\[SUGGESTION\]" lynis-$(date +%F).txt | head -60
grep "Hardening index" lynis-$(date +%F).txt
```

**macOS-specific hardening checks**
```bash
# Firewall status
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getstealthmode

# FileVault encryption
fdesetup status

# SIP (System Integrity Protection) status
csrutil status

# Gatekeeper
spctl --status

# Secure boot (Apple Silicon)
sudo bputil -d | grep -E "(Secure Boot|Kernel Extensions)"

# Find world-writable files (potential privilege escalation)
sudo find / -perm -o+w -type f -not -path "*/proc/*" 2>/dev/null | head -20

# SUID binaries (privilege escalation vectors)
sudo find / -perm -4000 -type f 2>/dev/null
```

**Compare against a previous audit**
```bash
# Save with date stamp each time
sudo lynis audit system --report-file lynis-report-$(date +%F).dat

# Check hardening index trend
grep "^hardening_index" lynis-report-*.dat
```
*Example: Hardening index drops from 72 to 58 after a software install — Lynis flags a new world-writable directory and a new SUID binary introduced by the installer.*

---

## Quick Reference — Tool→Scenario Matrix

| Scenario | Primary Tools |
|----------|--------------|
| Malware triage | die.app, KnockKnock, TaskExplorer, Ghidra |
| Network anomaly | Wireshark, Zeek, Brim, Sniffnet |
| Web app attack | Burp/Caido, ZAP, Nuclei, testssl |
| Secrets in code | Trufflehog, Gitleaks |
| Account compromise | AD Check, lsof, Little Snitch, `last` |
| Ransomware | lsof, FileMonitor, Little Snitch, Autopsy |
| Phishing | BBEdit, curl, olevba, Caido |
| Container security | Trivy, Syft, Grype, docker diff |
| Wi-Fi attack | WiFi Explorer Pro, Airtool 2, Wireshark, EAPTest |
| macOS persistence | KnockKnock, FileMonitor, PlistEdit Pro |
| Memory forensics | Volatility 3, Autopsy, TaskExplorer |
| Vuln scanning | Nuclei, Zenmap/Nmap, testssl |
| TLS issues | testssl, Wireshark, Caido |
| Hardening audit | Lynis, csrutil, spctl, fdesetup |

---

*IR & Security Scenarios Field Guide | June 27, 2026*

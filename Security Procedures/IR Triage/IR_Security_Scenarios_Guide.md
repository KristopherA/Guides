# Incident Response & Security Scenarios — Field Guide
*Quick-reference commands for common sysadmin and security scenarios*

---

## Section 0 — Before you start

The scenario sections assume you have already done the four things in this section. In a real
incident they are what determine whether the response holds up afterward, and they are the parts
that cannot be reconstructed later.

### 0.1 Scope of this guide

Commands are written for **macOS analyst workstations and macOS/Linux hosts**. Windows endpoints are
referenced only where a tool spans both (Volatility, Defender XDR). Windows-specific triage is not
covered here — if Windows endpoints are in scope for an incident, that procedure needs writing
before you need it, not during.

This guide covers technical triage. It is not a substitute for the incident response plan, legal
advice, or the insurer's requirements.

### 0.2 Contacts — fill this in before the first incident

An escalation table with blanks in it is the most common reason a response stalls at 2am. Complete
this, print it, and keep a copy that does not depend on the network or the mail system being up.

```text
Incident lead / on-call:
Backup on-call:
Executive sponsor (decisions on disclosure, payment, shutdown):
Legal counsel (external, 24h):
Privacy officer:
Cyber-insurance carrier + policy number + 24h claims line:
IT/infrastructure escalation:
Mail/identity platform vendor support + contract ID:
Communications lead (member/staff notification):
Law enforcement (RCMP cybercrime / local police):
Out-of-band comms channel (assume email and chat are compromised):
```

**Assume your normal channels are hostile.** If the incident involves mail or identity compromise,
do not coordinate the response over corporate email, Teams or Slack. Agree the out-of-band channel
in advance — personal phones with a pre-shared group is sufficient and is better than deciding
during the event.

### 0.3 Severity classification

Set severity in the first fifteen minutes and write down why. It drives who gets woken up, and
re-rating later is normal and expected.

| Level | Definition | Response |
|-------|-----------|----------|
| **SEV1 — Critical** | Active attacker, encryption in progress, confirmed exfiltration of personal information, or a business-critical system down | Immediate escalation, incident lead assigned, executive sponsor and counsel notified, insurer notified, out-of-band channel opened |
| **SEV2 — High** | Confirmed compromise, contained or containable; credential theft with evidence of use; targeted campaign against multiple users | Escalate same business day; incident lead assigned; privacy officer notified if personal information may be involved |
| **SEV3 — Moderate** | Single-user compromise with no evidence of lateral movement; malware detected and blocked; a user interacted with a phish (clicked, opened, scanned) but entered no credentials | Normal working hours; ticket with full case notes |
| **SEV4 — Low** | Reported phish with no interaction; policy violation; scanning or nuisance traffic | Ticket, trend tracking, no escalation |

Escalate a level when any of these is true, regardless of the initial rating: personal information
of members or staff may have been accessed; money has moved or been redirected; an administrator or
executive account is involved; backups are affected; or you cannot determine scope.

### 0.4 Evidence handling and chain of custody

Section 7.0 has the detailed handling rules for mail samples. They generalise, and the principles
below apply to every scenario in this guide.

- **Work from copies. Preserve originals read-only.** Hash the original on acquisition, record the
  hash, and verify it again before any analysis that will be relied on.
- **Never use a synced folder** — iCloud, OneDrive, Dropbox — as a case workspace. It leaks evidence
  and it destroys timestamps.
- **Record actions as you take them, with timestamps and timezone.** Reconstruction after the fact
  is unreliable and, if the matter becomes a legal or insurance question, close to worthless.
- **Volatile first** (RFC 3227 order): CPU/cache and routing, ARP and network connection state, then
  the process table, then a full memory image, then disk. Network and process state are cheap and
  disappear fastest; a memory image takes tens of minutes and, if you do it first, the state you
  most wanted is already gone. Section 6 follows this order — suspend, record state, then image.
- **Note every change you make to the system**, including the containment steps. An undocumented
  analyst action is indistinguishable from attacker activity in a later review.

```text
Chain of custody — one row per item, kept with the case
Item / description:
Acquired from (host, path, mailbox):
Acquired by / date / time / timezone:
Acquisition method and tool version:
Hash (SHA-256):
Storage location:
Transfers (who, when, why) — append only:
```

Retention: keep incident evidence for the period set by the organization's retention schedule and
by counsel. Do not destroy anything while any legal, insurance or regulatory matter is open — hold
notices override the normal schedule.

### 0.5 Privacy breach notification

**This section states the regulator's published requirements. It is not legal advice, and which
statute applies to this organization must be confirmed with counsel and the privacy officer and
recorded here before an incident.** the provincial Office of the Information and Privacy Commissioner
administers three separate regimes with different tests and different deadlines.

| Regime | Applies to | Test | Deadline |
|--------|-----------|------|----------|
| **PIPA** (s.34.1) | Private sector organizations | "Real risk of significant harm" (RROSH) to an individual | Notify the Commissioner **without unreasonable delay** |
| **HIA** (s.60.1(2)) | Health custodians | "Risk of harm" to an individual — a lower bar than RROSH | Notify the Commissioner **as soon as practicable**; also notify the Minister and the affected individuals (s.60.1(3)) |
| **POPA** (s.10(2)) | Public bodies — replaced FOIP on 11 June 2025 | RROSH | Notify affected individuals, the Commissioner, **and** the Minister. Confirm the current timing requirement on the form — POPA is new and the guidance is still settling |

Under PIPA the Commissioner may require an organization to notify affected individuals (s.37.1), and
an organization may also notify on its own initiative (s.37.1(7)) — in practice most do, before the
report is filed. Self-initiated notifications should still meet the content requirements in s.19.1
of the PIPA Regulation.

**What starts the assessment.** A breach means loss of, unauthorized access to, or unauthorized
disclosure of personal information. It does not require an attacker, a successful intrusion, or
malice — a misdirected email and a lost laptop both qualify. Ransomware with exfiltration qualifies.
Credential theft qualifies where the account could reach personal information.

**What to do:**

1. Notify the privacy officer as soon as personal information may be involved. Do not wait for
   confirmation, and do not make the RROSH determination alone — it is the privacy officer's call
   with counsel.
2. Preserve what is needed to assess scope: whose information, what fields, how many individuals,
   whether it was accessed or merely exposed.
3. Record the time the organization became aware. The clock and the record of diligence both start
   there, and "without unreasonable delay" is judged against it.
4. File using the regulator's current form. Do not draft notification content without counsel.

**Log every breach, including the ones you decide not to report.** The reporting threshold and the
record-keeping obligation are separate. Organizations subject to PIPEDA must keep a record of every
breach of security safeguards for 24 months (s.10.3) and produce them to the Commissioner on
request, and a decision not to notify is only defensible if the assessment behind it was written
down at the time. Keep a single breach register with, for each entry: date, description, personal
information involved, number of individuals, the harm assessment, the decision, and who made it.

Forms and guidance: [FILL IN: your privacy commissioner's breach-reporting page]

Federal PIPEDA may apply instead of or alongside provincial law depending on the organization's
activities. Confirm with counsel which regime governs, and record the answer here rather than
deciding it mid-incident.

---

## Scenario Index

0. [Before you start — contacts, severity, evidence, notification](#section-0--before-you-start)
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
15. [Business Email Compromise / Payment Fraud](#15-business-email-compromise--payment-fraud)
16. [Lost or Stolen Device](#16-lost-or-stolen-device)
17. [Insider Threat / Departing Employee](#17-insider-threat--departing-employee)
18. [After the Incident](#18-after-the-incident)

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

### Tools: AD Check, Little Snitch, 1Password, lsof, last, identity-provider admin console

Most account compromise now happens in the identity provider, not on the endpoint. An attacker with
a stolen session cookie never touches a password and never appears in `last`. Work 5.1 and 5.2
together unless you have positive evidence the account is local-only.

**Order matters.** Revoke sessions *after* resetting the password, not before — otherwise the
attacker's live session survives the reset and can simply set a new password. Full sequence in 5.3.

### 5.1 Identity and cloud triage — do this first

The attacker's persistence is here, and it survives a password reset. Every item below has been used
to retain access after the "account secured" ticket was closed.

| Check | What you are looking for |
|-------|--------------------------|
| Sign-in logs | Impossible travel, unfamiliar ASN, legacy/basic auth, non-interactive token refreshes |
| Registered MFA methods | A method added around the time of compromise — phone, authenticator, FIDO key |
| OAuth / app consent grants | Third-party apps granted mail or file scopes; consent to an unverified publisher |
| Inbox and forwarding rules | Auto-forward externally, or auto-delete of mail matching "security", "phish", "password" |
| Mailbox delegates and permissions | Full-access or send-as granted to another account |
| App passwords / legacy tokens | Long-lived credentials that bypass MFA entirely |
| New device or app registrations | Enrolled devices the user does not recognise |
| Conditional access exclusions | The account added to a bypass group |

```bash
# Microsoft 365 — read-only review. Requires Exchange Online / Entra admin permissions.
# Run in PowerShell with the ExchangeOnlineManagement and Microsoft.Graph modules installed.
#
#   Get-InboxRule -Mailbox user@example.org | Format-List Name,Enabled,ForwardTo,
#       ForwardAsAttachmentTo,RedirectTo,DeleteMessage,MoveToFolder
#   Get-Mailbox user@example.org | Format-List ForwardingAddress,ForwardingSmtpAddress,
#       DeliverToMailboxAndForward
#   Get-MailboxPermission user@example.org | Where-Object { $_.User -notlike "NT AUTHORITY*" }
#   Get-MailboxFolderPermission user@example.org:\Inbox    # folder-level Default/Anonymous
#       grants are a common persistence path that Get-MailboxPermission does NOT return
#   Get-MgUserAuthenticationMethod -UserId user@example.org
#   Get-MgUserOauth2PermissionGrant -UserId user@example.org
#
# The audit log is the primary evidence source — the cmdlets above show current
# configuration, not what happened:
#   Search-UnifiedAuditLog -StartDate <d> -EndDate <d> -UserIds user@example.org
#   Entra sign-in logs: export from the portal or Get-MgAuditLogSignIn
#
# CAVEAT: Get-InboxRule returns only rules the Outlook rules engine exposes. Rules
# created directly through MAPI/EWS can be hidden from it entirely — which is exactly
# the persistence the example below describes. Cross-check against the audit log for
# New-InboxRule / UpdateInboxRules operations.
#
# Google Workspace — Admin console: Security > Investigation tool, plus
#   Apps > Google Workspace > Gmail > check user's filters and forwarding
#   Security > API controls > App access control (third-party app grants)
```

Preserve the output before you remove anything. A forwarding rule is evidence of scope as well as a
persistence mechanism, and deleting it first destroys the record of what the attacker was reading.

*Example: password reset completed and the ticket closed. Three weeks later the same mailbox is
still relaying to an external address, because a rule named `.` — a single period, invisible at a
glance in the rules list — was never removed.*

### 5.2 Endpoint and on-prem triage

```bash
# Who is logged in right now
w
last | head -20

# Check for unexpected sudo usage
grep sudo /var/log/auth.log | tail -50              # Debian/Ubuntu
grep sudo /var/log/secure | tail -50                # RHEL/Fedora/Amazon Linux
journalctl _COMM=sudo --since '24 hours ago'        # systemd-journal-only hosts
log show --predicate 'process == "sudo"' --last 1h  # macOS

# Active network connections from user processes.
# -a is required: without it lsof ORs the criteria and you get every internet
# connection on the box PLUS every file the user has open.
sudo lsof -a -i -u username

# Check SSH authorized_keys for backdoors — on every host the account can reach.
# Use the absolute path; ~ expands to whoever is running the shell, i.e. you.
sudo cat /Users/username/.ssh/authorized_keys      # macOS
sudo cat /home/username/.ssh/authorized_keys       # Linux
sudo cat /root/.ssh/authorized_keys
```

*Example: `last` shows a login from an unfamiliar IP at 3am. Cross-reference with Little Snitch logs
to see what that session accessed.*

```bash
# List all local users
dscl . list /Users | grep -v "^_"

# Check if any user has unexpected admin rights
dscl . read /Groups/admin GroupMembership

# Recently MODIFIED files in the user's home. Create the reference marker first —
# it does not exist by default, and -newer against a missing file errors out.
touch -t "$(date -v-1d +%Y%m%d%H%M)" /tmp/24h-marker   # macOS/BSD date
# GNU/Linux equivalent: touch -d '24 hours ago' /tmp/24h-marker
find /Users/username -newer /tmp/24h-marker -ls 2>/dev/null

# Or skip the marker entirely
find /Users/username -mtime -1 -ls 2>/dev/null

# These test MODIFICATION time, not access. For access time use -newerat / -anewer —
# but treat atime as unreliable on APFS, which does not update it by default.
# If you need to know what was READ rather than changed, the answer is in the
# platform audit log (5.1), not on the filesystem.

# Active Directory account status — `id` returns uid/gid/groups only and says
# nothing about enabled/disabled/locked/expired. Use the AD Check.app GUI, or query
# the directory directly from a bound host:
dscl "/Active Directory/DOMAIN/All Domains" -read /Users/username \
  userAccountControl accountExpires pwdLastSet 2>/dev/null
id username@domain      # group membership only — not a status check
```

### 5.3 Contain — in this order

1. **Block sign-in / disable the account** at the identity provider. Necessary, but understand its
   limit: in Entra ID and Google Workspace, blocking sign-in does **not** invalidate access or
   refresh tokens already issued. A stolen session keeps working until the token expires — commonly
   up to an hour, longer for apps that are not continuous-access aware. Step 4 is what actually
   stops the attacker.
2. **Remove attacker-added MFA methods** *before* the reset. An attacker-controlled method can drive
   self-service password reset and retake the account in the gap between reset and cleanup.
3. **Reset the password** to a value the user does not choose and has never used.
4. **Revoke all active sessions and refresh tokens.** This is the step that ends live access. Doing
   it before step 3 accomplishes nothing — the attacker simply signs back in with the password they
   still have.
5. **Re-enrol the user's MFA** on a known-good device, verified out of band.
6. **Remove unauthorized OAuth grants, app passwords, delegates, forwarding and inbox rules** —
   after the output from 5.1 is preserved in the case record.
7. **Revoke or reissue any credential the account could reach**: SSH keys, API tokens, VPN profiles,
   service accounts whose secrets sat in that mailbox or drive.
8. **Re-enable only after** steps 1–7 are complete and the user is reachable on a verified channel.

```bash
# macOS — disable a local user.
#
# The documented `dscl . change` trick requires the OLD value to match exactly, and
# the macOS default shell has been /bin/zsh since 10.15 — the /bin/bash form fails
# with eDSAttributeValueNotFound and leaves the account fully enabled while looking
# like it succeeded. Set the value outright instead:
sudo dscl . -create /Users/compromised_user UserShell /usr/bin/false

# This blocks new SHELL logins only. GUI loginwindow, screen sharing and SMB are
# unaffected, and it does not terminate existing sessions or invalidate issued tokens.
# To actually disable the account:
sudo dscl . -create /Users/compromised_user AuthenticationAuthority ";DisabledUser;"
sudo pkill -u compromised_user     # terminate live sessions

# Active Directory (requires AD admin rights)
# In AD Check.app → search user → Disable Account
```

### 5.4 Scope before you close

Compromise of a single account is rarely the objective. Before closing, establish:

- What the account could read — mailboxes, shared drives, HR or member data, finance systems.
- Whether anything was actually accessed or downloaded, from the provider's audit log, not inference.
- Whether mail was sent *from* the account (check Sent Items and the message trace, not just the
  mailbox — attackers delete from Sent).
- Whether other accounts received lures from it, which turns this into Section 7.
- Whether payment or banking instructions were altered, which turns this into Section 15.
- Whether personal information was exposed, which starts the clock in 0.5.

---

## 6. Ransomware Response

### Before the first command — three decisions

- **Suspend, don't kill; isolate, don't power off.** Shutting down destroys memory. Killing the
  process destroys its address space. `kill -STOP` does neither — it stops the encryption *and*
  preserves the process for imaging. That is the move, and it is what the sequence below does.
- **Notify the cyber-insurance carrier early, and engage counsel first.** Many policies require
  notice before incident-response, legal or negotiation costs are incurred, and many require panel
  providers — engaging your own responder first can prejudice or void coverage under those policies.
  Confirm what *this* policy requires; it is not uniform. Counsel is generally engaged before the
  carrier so the investigation runs under privilege. Policy number and 24-hour claims line go in 0.2.
- **Write evidence to external media**, not to the volume being encrypted. Every capture path below
  assumes `/Volumes/IR` is attached external storage. `/tmp` is on the disk under attack and is
  purged on reboot.

### First 60 seconds — contain

```bash
# 0. Attach external media for captures.
EV=/Volumes/IR/$(date +%Y%m%d-%H%M)-case; mkdir -p "$EV"

# 1. Identify and SUSPEND the encrypting process. This stops the damage immediately
#    while keeping the process imageable. Do not kill it yet.
ps aux | grep -E "(crypt|cipher|lock|ransom)"
sudo lsof -c <suspect_process> | grep -E '\.encrypted|\.locked|\.ransom' | head -20
#    Note: BSD grep has no \| alternation — use grep -E. Bare `lsof` with no
#    filter takes minutes on macOS; scope it to the process or use -p <PID>.
sudo kill -STOP <PID>

# 2. Start capturing. fs_usage and tcpdump are live traces — they record from now
#    forward, not retroactively — so start them BEFORE isolating the interface.
sudo fs_usage -w > "$EV/fs_trace.txt" &
sudo tcpdump -i any -w "$EV/ransom-traffic.pcap" &

# 3. NOW isolate. Do NOT power off. Capture first, or the pcap is empty.
sudo networksetup -setairportpower en0 off    # Wi-Fi: configd will re-up `ifconfig down`
sudo ifconfig en1 down                        # wired — check `ifconfig -l` for ALL interfaces
#    Thunderbolt bridges, USB adapters and utun* VPN interfaces stay up unless
#    you bring them down too. `ifconfig -l` lists what actually exists.
#
#    WARNING: this ends your own session on a machine you administer remotely.
#    An upstream switch-port or firewall block is the only safe alternative —
#    Little Snitch deny-all filters incoming connections too and will drop you as well.
```

**Preserve evidence — the process is suspended, so you have time**
```bash
# Capture memory. The process is stopped, so nothing is encrypting while this runs.
#
#   Apple Silicon / current macOS: there is no working full-memory acquisition tool.
#   osxpmem depends on the MacPmem kext, which is unmaintained, unsigned for current
#   macOS, and blocked by SIP. Do not plan a SEV1 around it.
#   What does work: pause the VM and take the .mem file from the bundle (Parallels/UTM),
#   or dump the single suspended process's address space:
sudo vmmap <PID> > "$EV/vmmap.txt"
sudo lldb -p <PID> -o "process save-core \"$EV/proc-<PID>.core\"" -o detach -o quit

# Only after capture: terminate.
sudo kill -9 <PID>

# Document encrypted file extensions.
find / \( -name "*.encrypted" -o -name "*.locked" -o -name "*.ransom" \) -print 2>/dev/null | head -50

# Find ransom note
find / \( -name "README*.txt" -o -name "HOW_TO_DECRYPT*" -o -name "RESTORE_FILES*" \) -print 2>/dev/null
#   Parentheses group the -o branches so an explicit action (-print, -ls, -exec, -delete)
#   applies to all of them. With no action at all the implicit -print covers the whole
#   expression anyway — but write the parentheses, because the moment you add -ls or
#   -delete without them the action silently binds to the last branch only.

# Check what was modified in the last hour — note this is modification time, not access time.
touch -t "$(date -v-1H +%Y%m%d%H%M)" /tmp/1h-marker     # macOS/BSD date
# GNU/Linux equivalent: touch -d '1 hour ago' /tmp/1h-marker
find / -xdev -newer /tmp/1h-marker -type f -print 2>/dev/null | head -100

# Or without the marker
find / -xdev -mmin -60 -type f -print 2>/dev/null | head -100
#   -xdev keeps this on one volume. Without it, macOS traverses the firmlinked
#   /System/Volumes/Data and every mount under /Volumes — including your evidence disk.
```

**On key recovery from memory.** Some historical families left usable key material in RAM, and it is
worth capturing for that reason. Do not plan on it: most current families use per-file keys wrapped
with an attacker-held public key and zero the plaintext after each file, which makes what is left in
memory useless without the private key. Capture because it is cheap once the process is suspended —
not because it is a likely recovery path, and never at the cost of delaying containment.

**Identify the strain (use die.app + Malcat + VirusTotal)**
```bash
shasum -a 256 /path/to/ransomware_binary
# Submit the hash to: https://www.virustotal.com
# Strain identification: https://id-ransomware.malwarehunterteam.com
#
# CAUTION — same rule as 7.3: submit hashes, not content. An encrypted file is
# organizational data and a ransom note usually carries a victim/campaign ID that
# identifies you to the operator. Uploading either is a disclosure decision, not a
# technical one; get counsel and insurer sign-off first.
```

### Check for exfiltration — assume double extortion

Most current operators steal data before encrypting, so a clean restore does not end the incident.
Encryption is the ransom lever; the data theft is the legal exposure.

```bash
# Large outbound transfers in the days before encryption — from firewall/proxy logs, not the host
# Look for: cloud storage endpoints, rclone/megatools/WinSCP user agents, sustained upload volume

# Archive staging on the host — attackers compress before they exfiltrate
find / \( -name "*.7z" -o -name "*.rar" -o -name "*.zip" \) -mtime -14 -size +100M 2>/dev/null

# Exfiltration tooling left behind
ls -la /tmp /var/tmp ~/Downloads 2>/dev/null | grep -iE "rclone|mega|winscp|filezilla|ngrok"
```

If data left the environment, the incident includes a privacy breach. Go to 0.5 and start the
assessment — the notification obligation is independent of whether you recover the files.

### Recovery — verify backups before you trust them

```bash
# 1. Are the backups reachable from the compromised environment?
#    If yes, assume they are also encrypted or deleted. Attackers target backups first.
#    Only offline, immutable, or separately-credentialed copies can be assumed intact.

# 2. Identify the earliest known-good restore point — this must predate initial access,
#    not just the encryption event. Dwell time is commonly weeks.

# 3. Test-restore to an isolated environment and verify before any production restore.

# 4. Rebuild rather than clean. Restoring data onto a host that was compromised
#    reinstates the attacker's persistence along with the files.
```

Restore order: identity and directory services, then core infrastructure, then business systems by
documented priority. Reconnect to the network only after credentials have been rotated environment-
wide — every account, service account and API key the attacker could have reached.

### On paying

The decision to pay is a business and legal decision, never a technical one, and it is not the
responder's call. Record who made it and on what advice. Points that belong in the record:

- Payment does not reliably produce a working decryptor, and does not stop publication of stolen data.
- Sanctions screening must happen before any payment, and counsel determines which regime applies.
  A Canadian organization is governed by SEMA and Criminal Code listings; OFAC applies where there
  is a US nexus. The regimes differ in scope — do not assume one covers the other.
- The insurer and counsel must be engaged before any contact with the operator.
- A free decryptor may already exist — check <https://www.nomoreransom.org> before anything else.

---

## 7. Phishing Email Investigation

**Purpose.** Preserve the original message, determine whether anyone interacted with it, identify
the full delivery and collection chain, contain affected accounts or endpoints, and leave behind a
defensible case record. A clean reputation result is not a verdict; many campaigns use new domains,
compromised sites, or legitimate sharing services.

**Use this section when** a user reports suspicious email, a mail gateway generates a phishing alert,
or an incident starts from a mail-delivered URL, attachment, QR code, callback number, or calendar
invitation.

**Stop and escalate immediately** if ransomware, active malware execution, confirmed credential or
MFA entry, session-cookie theft, payment diversion, sensitive-data disclosure, or an executive/VIP
account is involved. Preserve evidence before remediation where doing so does not prolong harm.

### Tools and access

Core tools: Python 3, `ripmime`, `jq`, `curl`, `dig`, `whois`, `openssl`, `file`, `7z`, `zbarimg`,
Poppler, oletools, Didier Stevens' PDF tools, and an intercepting proxy such as Caido or Burp.

Access that may be required: the original mailbox or journal, mail-gateway search, Zimbra MTA logs,
DNS/proxy/firewall/EDR logs, identity-provider audit logs, and authority to quarantine messages or
disable accounts. Commands that change state are labelled **containment** and should be peer-checked.

```bash
# macOS analyst workstation: install the common static-analysis utilities
brew install ripmime jq bind whois p7zip zbar poppler
python3 -m pip install --user oletools extract-msg publicsuffix2

# Record the installed versions in the case notes; flags and output can vary by release.
python3 --version
curl --version | head -1
olevba --version
```

Example:

```text
Python 3.12.6
curl 8.7.1 (arm64-apple-darwin23.0) libcurl/8.7.1 OpenSSL/3.3.1
olevba 0.60.2 on Python 3.12.6
```

A modern phish is rarely one link. The mail contains a wrapped or obfuscated URL, that URL redirects
two to five times through compromised sites and open redirects, the landing page is picked by
server-side cloaking, and the credentials are POSTed somewhere different again. The goal of this
section is to walk that chain end to end and identify the landing page and collection endpoint —
the parts most useful for precise blocking, hunting, containment, and reporting.

---

### 7.0 Ground rules before you touch a link

```bash
# Make a private case directory; do not put evidence in a shared sync folder.
umask 077
CASE="IR-2026-####"
mkdir -p "$CASE"/{evidence,work,output,notes}

# Preserve the original, record basic metadata, hash it, and work only from a copy.
cp -p suspicious.eml "$CASE/evidence/original.eml"
stat -x "$CASE/evidence/original.eml" > "$CASE/notes/original-stat.txt"
shasum -a 256 "$CASE/evidence/original.eml" | tee "$CASE/notes/SHA256SUMS"
cp "$CASE/evidence/original.eml" "$CASE/work/work.eml"
cd "$CASE/work"

# Defang anything you paste into a ticket, chat or notes
echo "$URL" | sed -e 's|https\?://|hxxp://|' -e 's/\./[.]/g'
```

Expected evidence record:

```text
8c2f8c...9a71  IR-2026-####/evidence/original.eml
```

- Never resolve or fetch a phishing URL from a corporate IP. Use an isolated VM on an out-of-band
  connection. The actor sees your source address, and hits from your egress range tell them the
  campaign has been caught.
- The URL path or fragment almost always encodes the recipient (base64 or hex of their email).
  Decode it for scoping, but treat every fetch as potentially one-time-use — some kits burn the
  link after the first visit, so capture each response body as you go.
- Do not submit tokenised URLs to public sandboxes. A public urlscan.io scan exposes your user's
  address and is monitored by the actors themselves. Use `"visibility":"unlisted"` or private.
- Do not submit canary credentials. It confirms a live target, and it contaminates your own
  detection logic later.
- Never paste live tokens, API keys, session cookies, or full recipient-bearing URLs into tickets.
  Store the raw value in the restricted evidence location and use a redacted or defanged copy in
  normal notes.
- Record all times in UTC and note the source timezone. Mail `Date:` headers are sender-controlled;
  trusted `Received:` timestamps and platform audit logs carry more evidentiary weight.

#### First-five-minute triage

Ask the reporter these questions before doing deep analysis:

1. Did you open the message, click a link, scan a QR code, call a number, or open an attachment?
2. Did you enter a password, approve MFA, provide a code, install software, or enable macros?
3. What device and browser did you use, and approximately when (include timezone)?
4. Is the message still in the mailbox, and can it be forwarded **as an attachment**?
5. Is there a business transaction, payroll change, gift-card request, or sensitive disclosure in play?

| User action | Initial severity | Immediate action |
|---|---:|---|
| Viewed only; no remote content | Low | Preserve and analyse |
| Clicked; no data entered | Medium | Hunt DNS/proxy/EDR, inspect destination |
| Opened active attachment or ran code | High | Isolate endpoint and start endpoint IR |
| Entered password, MFA code, or approved push | High | Disable/sign-out, reset, inspect persistence |
| Payment or sensitive data sent | Critical | Escalate to fraud/privacy/legal processes |

---

### 7.1 Parse the email and decode every part

```bash
# Headers first — authentication failures and relay path
grep -E "^(Received|From|Reply-To|Return-Path|Subject|X-Originating-IP|Authentication-Results):" work.eml

# Sender domain policy — is the real domain even protected?
dig +short TXT sender-domain.com | grep "v=spf"
dig +short TXT _dmarc.sender-domain.com
```

Raw `grep` is a quick look, not authoritative parsing: RFC 5322 headers can continue on indented
lines. Use the standard library to unfold them and retain repeated headers in order:

```bash
python3 - <<'PY'
from email import policy
from email.parser import BytesParser

wanted = {
    "received", "from", "reply-to", "return-path", "subject", "date", "message-id",
    "authentication-results", "arc-authentication-results", "arc-seal", "x-originating-ip"
}
with open("work.eml", "rb") as fh:
    msg = BytesParser(policy=policy.default).parse(fh, headersonly=True)
for name, value in msg.raw_items():
    if name.lower() in wanted:
        print(f"{name}: {' '.join(value.splitlines())}")
PY
```

Example:

```text
Return-Path: <bounce@mailer-example.net>
Authentication-Results: mx.example.org; spf=fail smtp.mailfrom=mailer-example.net;
 dkim=none; dmarc=fail header.from=university.example
From: "IT Support" <helpdesk@university.example>
Reply-To: resetdesk@outlook.example
Message-ID: <20260916174211.4832@mailer-example.net>
```

Interpret authentication carefully:

- SPF checks the envelope sender, not the visible `From:` address.
- DKIM validates a signing domain and message integrity; it does not prove the sender is trustworthy.
- DMARC asks whether authenticated SPF or DKIM aligns with the visible `From:` domain.
- Forwarding can break SPF. Mailing lists can alter the body and break DKIM. ARC may preserve the
  intermediary's account of earlier results, but trust ARC only from an intermediary your
  organization has chosen to trust.
- A message can pass SPF, DKIM, and DMARC and still be malicious when the attacker controls the
  sending domain or a legitimate account.

*Example: mail claims `@university.ca`, `Authentication-Results` shows `spf=fail dkim=none`, and
`Return-Path` is a Brazilian hosting domain — spoofed sender, not a compromised mailbox.*

Raw `grep` on the `.eml` will miss most links. Bodies are quoted-printable or base64, and
quoted-printable inserts a soft line break (`=` at end of line) that splits URLs in half. Decode
first, always.

```bash
# Decode every text MIME part into a single readable body
python3 -c "import email,sys;[sys.stdout.write(p.get_payload(decode=True).decode('utf-8','ignore')) for p in email.message_from_binary_file(open('work.eml','rb')).walk() if p.get_content_maintype()=='text']" > body.txt

# Split out attachments and inline images as files
ripmime -i work.eml -d ./parts/          # install: brew install ripmime

# Outlook .msg instead of .eml
python3 -m extract_msg --extract-embedded suspicious.msg   # install: pip3 install extract-msg
```

Inventory every MIME part and compare the declared type with the actual file type:

```bash
python3 - <<'PY'
from email import policy
from email.parser import BytesParser

with open("work.eml", "rb") as fh:
    msg = BytesParser(policy=policy.default).parse(fh)
for n, part in enumerate(msg.walk()):
    print(f"{n:02d}  {part.get_content_type():35}  {part.get_filename() or '-'}")
PY
find parts -maxdepth 2 -type f -exec file --brief --mime-type {} \; -print
```

Example mismatch:

```text
application/pdf  parts/Invoice-8841.pdf
application/x-dosexec
parts/Invoice-8841.pdf
```

The filename says PDF, but the bytes identify a Windows executable. Treat the detected type as the
truth.

#### 7.1a When the report is an inline forward

An inline Zimbra or Outlook forward preserves quoted body text but usually replaces the original
transport headers. A line such as `From: "Bob" <bob@example.net>` inside the body is a claim, not
header evidence. Authentication results on the forwarded message describe the internal forward.

```bash
# If all Received hosts are internal or loopback, this is likely only the forwarding envelope.
grep -c '^Received:' work.eml
grep '^Received: from' work.eml | grep -vE \
  '127\.0\.0\.1|10\.|192\.168\.|172\.(1[6-9]|2[0-9]|3[01])\.|mail\.example\.org'

# A References or In-Reply-To value may retain the original Message-ID.
grep -Ei '^(References|In-Reply-To):' work.eml
```

Example:

```text
6
References: <CAExampleMsgID0000000000000000000@mail.gmail.com>
```

Recover the original in this order:

1. Ask for **Forward as Attachment** so the original arrives as a `message/rfc822` child part.
2. Export/download the original `.eml` from the user's mailbox or mail journal.
3. Pivot on the original Message-ID in the MTA logs and mailbox search.

```bash
# Zimbra: search compressed and current MTA logs for the exact Message-ID.
zgrep -hF 'CAExampleMsgID0000000000000000000@mail.gmail.com' /var/log/zimbra.log*

# Zimbra: search a mailbox; verify locally supported syntax with `zmmailbox help search`.
zmmailbox -z -m user@example.org search -t message -l 25 \
  'subject:"Member Questions" after:2026/09/10 before:2026/09/13'
```

Sample search output:

```text
num: 1, more: false
Id       Type   From                    Subject
257931   mess   sender81@gmail.example  Member Questions
```

Do not treat an inline forward as proof that the claimed sender or visible URL was the one delivered.
Mark unavailable fields as `unknown`, not `pass` or `clean`.

---

### 7.2 Extract every URL, not just the obvious ones

```bash
# Decode HTML entities before matching — kits write &#x68;&#x74;&#x74;&#x70;s:// to defeat grep
python3 -c "import html,sys;sys.stdout.write(html.unescape(sys.stdin.read()))" < body.txt \
  | grep -oE "https?://[^\"'<> )]+" | sort -u > urls.txt

# Anchor text vs real target — the classic mismatch, shown side by side
python3 -c "
import re,html
h=html.unescape(open('body.txt').read())
for m in re.finditer(r'<a\b[^>]*href=[\"\']([^\"\']+)[\"\'][^>]*>(.*?)</a>',h,re.S|re.I):
    print(m.group(1),'||',re.sub(r'<[^>]+>','',m.group(2)).strip()[:60])
" | sort -u
```

Also check for links that are not `href` at all:

```bash
# Meta refresh, form targets, background loads, and javascript: handlers in the mail body
grep -oiE "<meta[^>]+refresh[^>]*>|<form[^>]+action=\"[^\"]+\"|javascript:[^\"']+|url\([^)]+\)" body.txt | sort -u
```

**Unwrap security-gateway rewrites.** If the mail passed through a gateway, every link is rewritten
and the original is hidden inside it. Unwrap it offline rather than clicking through the gateway.

```bash
# Microsoft Safe Links — original sits in the urlencoded 'url' parameter
python3 -c "import sys,urllib.parse as u;q=u.parse_qs(u.urlparse(sys.argv[1]).query);print(u.unquote(q['url'][0]))" "$WRAPPED"

# Proofpoint URLDefense v3 — original sits between the double underscores
echo "$WRAPPED" | sed -E 's#.*/v3/__(.*)__;.*#\1#'

# Proofpoint v2 — 'u' parameter, with _ meaning / and - meaning :
python3 -c "import sys,urllib.parse as u;q=u.parse_qs(u.urlparse(sys.argv[1]).query);print(u.unquote(q['u'][0]).replace('_','/').replace('-',':'))" "$WRAPPED"
```

Mimecast and Barracuda tokens are opaque and cannot be reversed offline — pull the pre-rewrite copy
from the gateway's own message log instead of resolving the token.

**Compare registrable domains, not prefixes.** A hostname such as
`confidential-mail.google.com.msg-view.example` belongs to `msg-view.example`, not Google. Do not
use a simple “last two labels” rule because public suffixes such as `.co.uk` and `.com.au` break it.

```bash
# Public-suffix-aware extraction; update the package during workstation maintenance, not mid-case.
python3 - <<'PY'
from publicsuffix2 import get_sld
from urllib.parse import urlsplit

for raw in open("urls.txt", encoding="utf-8"):
    raw = raw.strip()
    if not raw:
        continue
    host = (urlsplit(raw).hostname or "").rstrip(".").lower()
    print(f"{host}\t{get_sld(host) or '[IP-or-invalid]'}")
PY
```

Example:

```text
confidential-mail.google.com             google.com
confidential-mail.google.com.msg-view.co msg-view.co
files.example.co.uk                      example.co.uk
```

Then resolve and record all answers. CDNs and anycast make ownership a supporting clue, not a
verdict.

```bash
HOST="confidential-mail.google.com"
dig +noall +answer A "$HOST"
dig +noall +answer AAAA "$HOST"
IP="$(dig +short A "$HOST" | tail -1)"
whois "$IP" | grep -iE '^(OrgName|netname|origin|CIDR|country):' | head
```

Example:

```text
confidential-mail.google.com. 300 IN A 142.250.72.14
OrgName:        Google LLC
CIDR:           142.250.0.0/15
Country:        US
```

**Look beyond web links.** Capture telephone numbers, email addresses, cryptocurrency wallets, and
calendar locations. Callback phishing often has no URL, and calendar-invite phishing may place the
lure in `DESCRIPTION`, `LOCATION`, or `ATTACH`.

```bash
# Telephone numbers and mail addresses in the decoded body (review false positives manually).
grep -Eo '\+?[0-9][0-9() .-]{7,}[0-9]|[[:alnum:]._%+-]+@[[:alnum:].-]+\.[A-Za-z]{2,}' \
  body.txt | sort -u

# Inspect iCalendar content without importing it into a calendar application.
grep -Ei '^(ORGANIZER|ATTENDEE|SUMMARY|DESCRIPTION|LOCATION|URL|ATTACH)(;[^:]*)?:' \
  parts/*.ics 2>/dev/null
```

**QR codes (quishing).** If the body is an image with no links at all, that is the finding.

```bash
# Read the URL out of a QR code image without a phone
zbarimg --raw parts/image001.png          # install: brew install zbar

# QR embedded in a PDF — pull the images out first
pdfimages -png attachment.pdf qr && zbarimg --raw qr-000.png
```

For multiple extracted images, keep the source filename next to each result:

```bash
find parts -type f \( -iname '*.png' -o -iname '*.jpg' -o -iname '*.jpeg' \) -print0 |
while IFS= read -r -d '' image; do
  result="$(zbarimg --quiet --raw "$image" 2>/dev/null)"
  [ -n "$result" ] && printf '%s\t%s\n' "$image" "$result"
done | tee qr-urls.txt
```

Sample:

```text
parts/image003.png    https://login-check.example/tenant/user@example.org
```

Treat the recipient value in that path as sensitive and redact it from general tickets.

---

#### 7.2a Check the registrable domain, not the prefix

Subdomain prefixing is the oldest trick still working, and it is invisible in a truncated mail
client display. Reduce every extracted URL to its registrable domain before you judge it.

```bash
# Print each hostname beside its public-suffix-aware registrable domain.
# Prerequisite: python3 -m pip install --user publicsuffix2
python3 - <<'PY'
from publicsuffix2 import get_sld
from urllib.parse import urlsplit
for raw in open('urls.txt', encoding='utf-8'):
    host = (urlsplit(raw.strip()).hostname or '').rstrip('.').lower()
    if host:
        print(f'{host}\t{get_sld(host) or "[IP-or-invalid]"}')
PY
```

```text
confidential-mail.google.com              google.com
confidential-mail.google.com.msg-view.co  msg-view.co
```

The registrable domain reveals control boundaries, not intent. A compromised site can have a
perfectly legitimate registrable domain, and an unfamiliar domain is not automatically malicious.

```bash
# Record resolution and network ownership as supporting evidence, not as a verdict.
dig +short confidential-mail.google.com
whois "$(dig +short confidential-mail.google.com | tail -1)" | grep -iE "orgname|netname|origin"
```

### 7.3 Pull links out of attachments

Hash and check before anything else, then extract links statically — never open the file in its
native application.

```bash
shasum -a 256 parts/attachment.docx | tee -a ioc-hashes.txt
# Submit the hash (not the file, if it may contain internal data) to https://www.virustotal.com
```

**Office (docx/xlsx/pptx are ZIP archives)**

```bash
# Every external reference in an OOXML file is declared in a .rels part — this catches all of them
unzip -o parts/attachment.docx -d docx/ >/dev/null && grep -rhoE 'Target="https?://[^"]+"' docx/ | sort -u

# Remote template injection — this URL is fetched automatically the moment the doc is opened
unzip -p parts/attachment.docx word/_rels/settings.xml.rels

# Legacy OLE (.doc/.xls) — extracts embedded objects and remote-template URLs
oleobj parts/attachment.doc              # install: pip3 install oletools

# Macro URLs, including strings reassembled from Chr() and StrReverse
olevba --decode parts/attachment.docm

# DDE links — executes without macros being enabled
msodde parts/attachment.docx
```

*Example: a docx with no macros and a clean VT score, but `settings.xml.rels` points at
`hxxp://45[.]61[.]x[.]x/tpl.dotm` — the document is only a downloader stub.*

**PDF**

```bash
# Count the risky structures first — /OpenAction, /JS, /Launch, /EmbeddedFile
pdfid.py parts/attachment.pdf            # install: pip3 install pdfid pdfparser

# Dump every link action object with its target
pdf-parser.py --search /URI parts/attachment.pdf | grep -oE "https?://[^)>]+" | sort -u

# Text-layer URLs (often a different set from the annotation URLs)
pdftotext -raw parts/attachment.pdf - | grep -oE "https?://[^ ]+" | sort -u
```

**HTML attachment / HTML smuggling**

```bash
# A large base64 blob in an .html attachment is the payload — decode it, don't open it
grep -oE "base64,[A-Za-z0-9+/=]{200,}" parts/attachment.html | cut -d, -f2 | base64 --decode > smuggled.bin && file smuggled.bin

# Where the page sends what it collects, and where it redirects to
grep -oiE "action=\"[^\"]+\"|location(\.href)?\s*=\s*[\"'][^\"']+|fetch\([\"'][^\"']+" parts/attachment.html | sort -u
```

**SVG, LNK and container files**

```bash
# SVG attachments can carry <script> and remote <image> pulls
grep -oiE "<script|xlink:href=\"[^\"]+\"|href=\"[^\"]+\"" parts/attachment.svg | sort -u

# .lnk — the target and arguments are where the download URL lives
python3 -m LnkParse3 parts/attachment.lnk | grep -iE "command|arguments|target"   # install: pip3 install LnkParse3

# ISO/IMG/ZIP wrappers — list contents without mounting
7z l parts/attachment.iso
```

**OneNote, disk images, archives, and password-protected payloads**

```bash
# Inventory an archive without extracting it. The -slt form includes paths, sizes and encryption.
7z l -slt parts/attachment.zip | grep -E '^(Path|Size|Encrypted|Method) ='

# Extract only inside the case workspace, never into Downloads or a synced folder.
mkdir -p extracted/attachment-zip
7z x -oextracted/attachment-zip parts/attachment.zip
find extracted/attachment-zip -type f -exec shasum -a 256 {} \; -exec file {} \;

# OneNote files: first identify strings, embedded PE files and script/URL indicators statically.
file parts/*.one
strings -a parts/attachment.one | grep -Ei 'https?://|powershell|cmd\.exe|mshta|rundll32|wscript' | head -50
```

If the password is supplied in the email, document that fact and use it only in the isolated case
workspace. Password protection prevents gateway inspection; it is not evidence of malware by itself.
Never upload a confidential attachment to a public multi-scanner without authorization.

**Executable or script content**

```bash
# Detect actual types regardless of extension and flag common active formats.
find parts extracted -type f -print0 2>/dev/null | while IFS= read -r -d '' f; do
  printf '%s\t' "$f"
  file --brief --mime-type "$f"
done | tee file-types.tsv

grep -Ei 'application/x-dosexec|application/x-mach-binary|application/x-executable|text/x-shellscript' \
  file-types.tsv
```

Example:

```text
extracted/attachment-zip/Payroll_Adjustment.pdf    application/x-dosexec
```

At this point, stop the mail-only workflow. Isolate any endpoint that opened or ran the file and
continue with the suspicious-process/malware scenario.

Add every URL found here to `urls.txt`. Attachment links and body links are usually different
stages of the same chain.

---

### 7.4 Walk the redirect chain hop by hop

Do not use `curl -L` alone — it hides the intermediate hosts, and `-I` (HEAD) is frequently answered
differently from a real GET. Step through one hop at a time and keep every body.

```bash
UA="Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/126.0 Safari/537.36"
u="$URL"; i=0
while [ -n "$u" ] && [ "$i" -lt 10 ]; do
  i=$((i+1))
  curl -sS --proto '=http,https' --connect-timeout 10 --max-time 30 -A "$UA" \
    -D "hop$i.hdr" -o "hop$i.body" \
    -w "status=%{http_code} type=%{content_type} bytes=%{size_download}\n" "$u" \
    | tee "hop$i.meta"
  next_raw="$(awk 'tolower($1)=="location:"{$1="";sub(/^ /,"");print;exit}' \
    "hop$i.hdr" | tr -d '\r')"
  next="$(python3 -c 'import sys;from urllib.parse import urljoin;print(urljoin(sys.argv[1],sys.argv[2]))' \
    "$u" "$next_raw")"
  printf 'hop%d: %s -> %s\n' "$i" "$u" "${next_raw:+$next}" | tee -a chain.txt
  [ -n "$next_raw" ] || break
  u="$next"
done
```

Example:

```text
status=302 type=text/html; charset=UTF-8 bytes=164
hop1: https://short.example/A7Q -> https://compromised.example/track?id=991
status=200 type=text/html bytes=18432
hop2: https://compromised.example/track?id=991 ->
```

The loop resolves relative `Location:` values correctly, caps runtime, and saves a header, body, and
transfer summary for every hop. It still makes live requests: run it only from the authorised
isolated analysis network.

```bash
# Resolve every host in the chain to an IP so you can see the hosting pivot
cut -d/ -f3 chain.txt | grep -E "^[a-z0-9.-]+$" | sort -u | while read -r h; do echo "$h -> $(dig +short "$h" | tail -1)"; done
```

**Test for cloaking.** Kits serve a benign page to scanners and the real page to victims, keyed on
User-Agent, geo-IP, referer or the token. Different results mean you have not seen the real landing
page yet.

```bash
# Same URL, three identities — compare final URL, status and response size
for ua in "curl/8.4.0" "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0 like Mac OS X)" "$UA"; do
  curl -s -L -A "$ua" -o /dev/null -w "%{size_download}\t%{http_code}\t%{url_effective}\t$ua\n" "$URL"
done
```

*Example: `curl/8.4.0` gets a 200 with a Google homepage clone, the iPhone UA gets a 302 into the
real kit — the operator is filtering out desktop scanners.*

**Follow the redirects curl cannot see.** Meta refresh, JS assignment and auto-submitting forms are
client-side, so the hop loop will stop on a 200 that is not really the end.

```bash
# Check each captured body for a client-side hand-off before concluding you've reached the end
grep -oiE "<meta[^>]+refresh[^>]*url=[^\"'>]+|(window\.|document\.)?location(\.href|\.replace\()?\s*=?\s*[\"'][^\"']+|<form[^>]+action=\"[^\"]+\"" hop*.body

# Obfuscated hand-off — decode the base64 that gets passed to atob()
grep -oE "atob\(['\"][A-Za-z0-9+/=]+" hop*.body | cut -d\" -f2 | cut -d\' -f2 | base64 --decode
```

If the chain still dead-ends in packed JavaScript, use a headless browser in the isolated VM or an
approved browser-analysis service whose privacy level matches the evidence.

```bash
# Prefer private. Unlisted scans can still be visible to vetted third parties.
curl -s -X POST https://urlscan.io/api/v1/scan/ -H "API-Key: $URLSCAN_KEY" \
  -H "Content-Type: application/json" \
  --data "$(jq -n --arg url "$URL" '{url:$url,visibility:"private",tags:["phishing","incident-response"]}')"

# Then pull the full request chain the browser actually performed
curl -s "https://urlscan.io/api/v1/result/$UUID/" | jq -r '.data.requests[].request.request.url'
```

Do not submit a recipient-bearing or otherwise confidential URL unless the service, account tier,
and organizational policy provide the required visibility. “Unlisted” is not the same as private.

#### 7.4a Chains that begin behind a legitimate host

Google Drive/Gmail, SharePoint/OneDrive, Dropbox, Adobe, DocuSign, Canva, and similar services can be
the delivery wrapper for malicious content. The vendor domain may be genuine while the sender,
shared object, or next-stage link is malicious.

Recognise the pattern:

- Hop 0 is the vendor's real registrable domain and certificate.
- Static fetching returns a login/interstitial and no useful `Location:` header.
- Access is recipient-bound or requires the target identity.
- Reputation and pre-delivery scanning are clean because the visible host is legitimate.

Response:

1. Verify the sender through a known-good channel. Ask them to resend the content plainly when
   practical.
2. Do not sign in from a normal corporate workstation merely to inspect it.
3. If opening is necessary and authorised, use an isolated VM, out-of-band egress, a fresh browser
   profile, and only the minimum-purpose test or target identity approved for the case.
4. Assume the service and sender can observe access. Record the decision before opening.
5. Capture the first inner URL or file as a new artefact, hash it, and restart the workflow there.
6. Block the malicious object, sender, or tenant where supported; do not block the entire vendor
   domain.

Example disposition:

```text
Carrier:    Gmail confidential-mode notice
Hop 0:      hxxps://confidential-mail[.]google[.]com/... (Google-owned)
Pre-open:   Recipient-bound interstitial; no second-stage content available
Headers:    Inline forward; original envelope not preserved
Verdict:    Inconclusive statically; sender identity not verified
Action:     Verify sender out of band; request plain-text resend; do not sign in from production
```

---

### 7.5 Identify the collection endpoint / credential sink

The last URL in the redirect chain is the page the user sees. The collection endpoint is where that
page sends captured data. It may be attacker-controlled infrastructure or an abused legitimate API;
do not label a legitimate service itself as attacker C2.

```bash
# The form target and any background POST endpoints on the final landing page
grep -oiE "action=\"[^\"]+\"|fetch\([\"'][^\"']+|XMLHttpRequest[^;]*open\([^,]+,\s*[\"'][^\"']+|ws{1,2}s?://[^\"']+" final.html | sort -u

# Absolute exfil endpoints hiding in the bundled JS
grep -rhoE "https?://[^\"' )]+" final.html *.js | sort -u | grep -v "$(echo "$URL" | cut -d/ -f3)"
```

Markers worth recognising in the output:

- `post.php`, `next.php`, `submit.php`, `login2.php` — stock phishing-kit handlers.
- `api.telegram.org/bot<token>/sendMessage` — a common exfil path. The bot token and chat ID are
  sensitive, high-confidence campaign artefacts and are reportable to Telegram; redact them from
  general tickets.
- A POST target on a *different* domain from the landing page — investigate it as the collection
  endpoint rather than automatically classifying the whole domain as malicious.
- Page assets loading from the legitimate vendor while only the login form is local, plus a
  websocket back to the phishing host — that is adversary-in-the-middle (Evilginx, Tycoon, EvilProxy).
  Treat it as session-token theft, not password theft, and jump straight to 7.7.

```bash
# Some kits leave an archive beside deployed files. This request is visible to the operator;
# perform it only when the investigation authorizes active interaction.
curl -sS --proto '=https' --connect-timeout 10 --max-time 20 -I \
  "https://$HOST/$(basename "$PATHDIR").zip" | head -1
```

---

### 7.6 Pivot on the infrastructure

Once you have an attacker-controlled host, expand it into the rest of the campaign so you block a
cluster, not a single name. Do not run infrastructure pivots against a shared legitimate service as
though all sibling hosts were malicious.

```bash
# Registration and nameserver detail — a domain registered three days ago is the tell
curl -s "https://rdap.org/domain/$DOMAIN" | jq '{registered: .events, ns: [.nameservers[].ldhName]}'

# Sibling hostnames from certificate transparency — kits provision many subdomains at once
curl -s "https://crt.sh/?q=%25.$DOMAIN&output=json" | jq -r '.[].name_value' | sort -u

# Every SAN on the live certificate — often lists the operator's other phishing domains
openssl s_client -connect "$HOST:443" -servername "$HOST" </dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep -A1 "Subject Alternative Name"

# Favicon hash — pivot in Shodan (http.favicon.hash:) or Censys to find the same kit elsewhere
python3 -c "import mmh3,base64,sys,urllib.request as r;print(mmh3.hash(base64.encodebytes(r.urlopen(sys.argv[1]).read())))" "https://$HOST/favicon.ico"
```

---

### 7.7 Record the chain and close the loop

Write the chain down as evidence — the hop table is what makes the report defensible and what feeds
the blocklist.

```
Hop  URL (defanged)                         IP / ASN              Role
0    hxxps://safelinks…/?url=…              n/a                   gateway rewrite
1    hxxps://compromised-site[.]org/x.php    203.0.113.10 / AS64500  open redirect
2    hxxps://t[.]co-verify[.]top/auth        198.51.100.7 / AS64501  cloaking gate
3    hxxps://login[.]co-verify[.]top/m365    198.51.100.7 / AS64501  landing page
4    hxxps://api[.]telegram[.]org/bot…       legitimate service     abused collection API
```

```bash
# Flatten the artefacts into one IOC list for blocking and hunting
{ cut -d' ' -f2 chain.txt; cat ioc-hashes.txt; } | sort -u > iocs.txt
```

Then hunt, in this order:

1. **Mail** — search the gateway for the same sender IP, subject or any chain domain to find every
   recipient, not just the one who reported it.
2. **Proxy / DNS / firewall logs** — search for *every* hop domain and IP, not only the final one.
   Users who stopped at hop 2 still clicked.
3. **Identity** — for anyone who reached the landing page, treat it as credential compromise:
   revoke sessions and refresh tokens, reset the password, then check for newly registered MFA
   methods, new app consents, and new inbox rules (auto-forward or auto-delete of "security" mail).
   Those three are the standard post-AiTM persistence steps and they outlive a password reset.
4. **Detections** — turn the chain into a rule rather than a one-time block: alert on DNS resolution
   or proxy requests to any chain domain, on newly registered domains in mail links, and on inbox
   rule creation that forwards externally.

*Reporting: abuse@ the hosting ASN and registrar for each hop, plus the CDN if one is fronting the
kit. Compromised intermediate sites (hop 1 above) usually belong to a real business that does not
know — a short note to their abuse contact removes a link in the chain for everyone.*

#### Search examples

Use the syntax supported by the organization's own products. The following are patterns to adapt,
not universal copy-and-paste queries.

```text
# Splunk: any DNS request for known campaign domains
index=dns earliest=-30d (query="login.co-verify.example" OR query="t.co-verify.example")
| stats earliest(_time) AS firstSeen latest(_time) AS lastSeen values(query) BY src_ip,user

# Microsoft Defender XDR advanced hunting
DeviceNetworkEvents
| where Timestamp > ago(30d)
| where RemoteUrl in~ ("login.co-verify.example", "t.co-verify.example")
| project Timestamp, DeviceName, InitiatingProcessAccountUpn, RemoteUrl, RemoteIP, ActionType

# Zeek DNS log
zeek-cut ts id.orig_h query < dns.log \
  | awk 'tolower($3) ~ /^(login|t)\.co-verify\.example$/ {print}'
```

Example result:

```text
2026-09-16T17:42:09Z  mac-014  user@example.org  login.co-verify.example  198.51.100.7  ConnectionSuccess
```

Record both hits and confirmed non-hits, including the query window and data source. “No evidence”
means only that the searched telemetry did not contain a match.

### 7.8 Containment by observed user action

Do not wait for complete infrastructure attribution before containing a user or device with clear
exposure.

| Observed action | Required response |
|---|---|
| Message delivered, not opened | Quarantine/purge matching copies; block precise IOCs; notify recipients if needed |
| Link opened, no data entered | Hunt DNS/proxy/EDR; inspect downloads; monitor identity events |
| Password entered | Disable or block sign-in; revoke sessions; reset password; inspect mailbox and identity persistence |
| MFA code entered or push approved | Treat as session compromise; revoke sessions/tokens and inspect sign-ins immediately |
| Attachment executed | Isolate endpoint; collect volatile evidence if appropriate; start malware workflow |
| Payment or data sent | Preserve transaction records and escalate to finance/privacy/legal and leadership |

**Microsoft 365 example — inspect before changing state:**

```powershell
# Read-only review; requires the appropriate Exchange Online permissions.
Get-InboxRule -Mailbox user@example.org -IncludeHidden |
  Format-Table Name,Enabled,ForwardTo,RedirectTo,DeleteMessage,MoveToFolder -Auto

Get-Mailbox user@example.org |
  Format-List ForwardingAddress,ForwardingSmtpAddress,DeliverToMailboxAndForward
```

Sample suspicious result:

```text
Name             Enabled ForwardTo                  DeleteMessage
----             ------- ---------                  -------------
Invoice archive  True    externalbox@outlook.example          True
```

Do not bulk-delete rules. Export or record the exact rule, confirm it is unauthorized, and use
`Remove-InboxRule -WhatIf` before removal. Changes to Inbox rules can affect Outlook client-side
rules, so follow the current Microsoft procedure and your change-control policy.

**Zimbra example — scope matching messages:**

```bash
# Search, but do not delete, until the query has been reviewed.
zmmailbox -z -m user@example.org search -t message -l 100 \
  'from:sender81@gmail.example subject:"Member Questions" after:2026/09/10'

# Preserve the query and results in the case record before quarantine or deletion.
```

Containment order for confirmed identity compromise:

1. Block sign-in or disable the account if business impact permits.
2. Revoke active sessions and refresh tokens using the current identity-provider procedure.
3. Reset the password from a known-clean administrative session.
4. Review and remove unauthorized MFA methods, app passwords, OAuth grants, delegates, mailbox
   forwarding, and inbox rules.
5. Review sign-ins, audit events, sent/deleted mail, and data access from the earliest plausible
   compromise time.
6. Re-enable only after persistence is removed and the user has a clean device and new credentials.

### 7.9 Verdicts and confidence

Use one of these outcomes so “unknown” is not accidentally recorded as “safe.”

| Verdict | Meaning |
|---|---|
| Malicious | Evidence shows credential theft, malware, fraud, impersonation, or hostile infrastructure |
| Suspicious | Multiple risk indicators exist, but intent or payload is not proven |
| Benign | Expected business message verified through reliable evidence |
| Inconclusive | Required evidence is unavailable or access would create disproportionate risk |

State confidence separately (`high`, `medium`, or `low`) and list what would change the verdict.

Example:

```text
Verdict:    Suspicious (medium confidence)
Basis:      Unsolicited free-mail sender, recipient-bound Google confidential-mode link,
            sender not found in the member system, original transport headers unavailable
Unknowns:   Content behind access wall; original sending IP and authentication results
To resolve: Verify sender using the known member phone number or obtain a plain-text resend
```

### 7.10 Case-note template

```markdown
# Phishing investigation — IR-YYYY-NNNN

## Summary
- Reported by / time (UTC):
- Analyst / case owner:
- Verdict / confidence:
- Business impact:

## User interaction
- Viewed / clicked / opened / executed:
- Credentials, MFA, payment, or data supplied:
- Device, browser, and approximate time:

## Message evidence
- Original `.eml` obtained: yes/no
- SHA-256:
- Message-ID:
- Envelope sender / visible From / Reply-To:
- SPF / DKIM / DMARC / ARC:
- Original recipient(s):

## Artefacts and chain
| Hop | Defanged URL or file | SHA-256 / IP / ASN | Role | First seen |
|---:|---|---|---|---|

## Scope and hunting
- Mailboxes searched / query / time window:
- DNS, proxy, firewall, and EDR sources searched:
- Identity sources searched:
- Positive matches:
- Data gaps:

## Actions
- Quarantines / precise blocks:
- Accounts contained:
- Endpoints isolated:
- Notifications / abuse reports:

## Closure
- Residual risk:
- Detection or process improvement:
- Evidence location and retention date:
```

### 7.11 Common failure modes

- Analysing an inline forward as though it contains original headers.
- Declaring a message safe because SPF/DKIM/DMARC pass.
- Clicking only the display text rather than extracting the actual target.
- Using `curl -L` without retaining intermediate headers and bodies.
- Assuming a Google/Microsoft/Dropbox hostname makes the shared object benign.
- Uploading tokenised URLs or private attachments to public services.
- Blocking an entire shared service instead of the malicious object, account, tenant, or inner URL.
- Resetting a password without revoking sessions and removing persistence.
- Recording `clean` where the evidence only supports `inconclusive`.
- Deleting matching messages before recording the scope query and preserving a sample.

### 7.12 Reference checks

Verify platform-specific commands against current vendor documentation before a live incident:

- Microsoft Safe Links overview: <https://learn.microsoft.com/en-us/defender-office-365/safe-links-about>
- Microsoft compromised-account response: <https://learn.microsoft.com/en-us/defender-office-365/responding-to-a-compromised-email-account>
- Exchange `Get-InboxRule`: <https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-inboxrule>
- Google Gmail confidential mode: <https://support.google.com/mail/answer/7674059>
- urlscan API and visibility levels: <https://urlscan.io/docs/api/>
- Zimbra `zmmailbox` command reference: <https://wiki.zimbra.com/wiki/Zmmailbox>

---

### 7.13 Variant — callback phishing with no URL

Some campaigns contain only a phone number and a plausible invoice, subscription renewal, or fraud
warning. URL extraction and reputation checks will find nothing.

```bash
# Extract phone-number candidates from the decoded message and OCR text.
grep -Eo '\+?[0-9][0-9() .-]{7,}[0-9]' body.txt | sed 's/[() .-]//g' | sort -u

# Search all decoded parts for callback language.
grep -RniE 'call (us|now)|support line|renewal|subscription|refund|invoice|unauthori[sz]ed' parts body.txt
```

```text
body.txt:37:If you did not authorize this renewal, call Support at +1 (800) 555-0199.
```

Do not call from a personal or production phone. Check the number against internal telemetry and
approved reputation sources, compare it with the vendor's independently obtained contact details,
and preserve the PDF or image containing the number. Treat requests to install remote-support
software, read back a one-time code, or move money as malicious. If money moved, switch to
Section 15.

### 7.14 Variant — QR-only mail and image-based lures

The only actionable content may be inside an inline image. Extract it without scanning it with a
personal phone.

```bash
ripmime -i work.eml -d parts/
for f in parts/*; do
  file "$f"
  zbarimg --quiet --raw "$f" 2>/dev/null && printf 'source=%s\n' "$f"
done
```

```text
https://m365-auth.example/user@example.org
source=parts/image003.png
```

Redact recipient identifiers in normal case notes. Preserve the full tokenised URL only in the
restricted evidence store.

### 7.15 Variant — calendar invitation phishing

An `.ics` attachment can put a lure directly on a calendar even if the email body is empty. Inspect
it as text; do not import it into a production calendar.

```bash
grep -Ei '^(METHOD|ORGANIZER|ATTENDEE|SUMMARY|DESCRIPTION|LOCATION|URL|ATTACH)(;[^:]*)?:' \
  parts/*.ics 2>/dev/null
```

```text
METHOD:REQUEST
ORGANIZER;CN=Payroll:mailto:payroll-update@external.example
SUMMARY:Mandatory salary confirmation
LOCATION:https://hr-login.example/confirm
```

Check whether the sender is authorized to invite the recipient, then search the mail and calendar
platform for the organizer, UID, subject, and link. Removal must cover both the delivered message
and any calendar object it created.

### 7.16 Variant — compromised-vendor and thread-hijack mail

Authentication may pass and the message may be inside a real conversation. Look for a change in
intent rather than only a change in sender:

- unexpected bank-account or payment instructions;
- a new file-sharing service or attachment type;
- changed reply-to, writing style, timezone, or urgency;
- a display-name or signature mismatch within the thread;
- a request to continue on a new address, phone number, or chat service.

```bash
# Show the conversation identifiers and reply-routing fields together.
python3 - <<'PY'
from email import policy
from email.parser import BytesParser
msg = BytesParser(policy=policy.default).parse(open('work.eml','rb'), headersonly=True)
for h in ('From','Reply-To','Message-ID','In-Reply-To','References','Date','Subject'):
    for v in msg.get_all(h, []): print(f'{h}: {v}')
PY
```

Verify payment or identity changes through a known contact path from the vendor master record, not
the message itself. A DMARC pass does not lower the response priority when the account is genuinely
compromised. Where the objective is a payment, this is Section 15 — work both sections together.

### 7.17 Variant — password-protected archives

Attackers provide an archive password in the message specifically to bypass gateway inspection.
Business users also do this legitimately, so the password is a risk indicator rather than a verdict.

```bash
# List without extraction and record whether encryption is present.
7z l -slt parts/attachment.zip | grep -E '^(Path|Size|Encrypted|Method) ='
```

```text
Path = Remittance_0916.iso
Size = 7340032
Encrypted = +
Method = ZipCrypto Deflate
```

Extract only in the isolated case workspace. If a user opened or ran the payload, stop treating the
case as mail-only and move immediately to endpoint containment (Section 1) and malware triage.

### 7.18 Report-intake text for the help desk

Use this as a macro when asking for a better sample:

> Thank you for reporting this. Please do not click, reply, scan the QR code, call the number, or
> open the attachment again. In Zimbra/Outlook, use **Forward as Attachment** and send the original
> message to the security queue. Tell us whether you clicked or opened anything, whether you entered
> a password or approved an MFA prompt, which device you used, and the approximate time. Reporting
> quickly is the right action; you will not be blamed for the report.

Minimum ticket fields:

```text
Reporter / recipient:
Original message attached (.eml): yes / no
Received time and timezone:
Clicked / opened / scanned / called: yes / no / unknown
Password, MFA, payment, or data supplied: yes / no / unknown
Device and browser:
Message still in mailbox: yes / no
```

The standing instruction — "forward as attachment, do not paste or inline-forward" — removes the
whole 7.1a problem class at the source. It is the single highest-value process change available to
this section.

### 7.19 Worked example — the inline-forward sample

```text
Reported:   reception -> systems, 2026-09-11, inline forward
Carrier:    Gmail confidential mode notification, sender sender81@gmail.example (free account)
Link:       hxxps://confidential-mail[.]google[.]com/msg/<MSG_ID>…  -> eTLD+1 google.com, legitimate
Headers:    unusable — six internal Received: hops, no original envelope
Pivot:      References: <CAExample…@mail.gmail.com> — original Message-ID survived the forward
Scannable:  nothing. No attachment, no redirect chain, no second-stage URL visible pre-open.
Verdict:    cannot be resolved statically. Out-of-band sender verification is the cheapest path.
Secondary:  our own DKIM verdict on outbound mail is failing —
            "dkim=neutral reason=invalid (public key: OpenSSL error: too long) header.d=example.com"
            suggests a malformed or badly split p= value in the selector's TXT record.
            Unrelated to this report; worth a separate ticket, it weakens DMARC.
```

This case is the reference example for three of the failure modes in 7.11: a forwarded sample with
no usable headers, a legitimate registrable domain carrying the lure, and an investigation that
correctly ends at `inconclusive` rather than being forced to a verdict the evidence does not support.

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

## 15. Business Email Compromise / Payment Fraud

### Tools: message trace, identity provider audit logs, finance records, bank fraud line

BEC is a financial incident with an email delivery mechanism. The technical investigation is
Section 7 and Section 5; this section is the part that runs on a clock, because recovery of a
transfer depends almost entirely on how fast the bank is contacted.

### 15.1 If money has already moved — first hour

```text
1. Call the bank's fraud line immediately and ask for a recall. The mechanism depends
   on the payment type, so tell them what was sent, not just how much:
     International correspondent wire -> SWIFT recall / indemnity request
     Domestic wire (Lynx) or EFT       -> bank's own recall process, different rules
     Interac e-Transfer                -> usually unrecoverable once deposited
     Pre-authorized debit / ACH pull    -> reversal window, different again
   Recovery odds fall sharply after the first 24-72 hours and drop to near zero once
   funds are withdrawn or moved onward from the receiving account.
2. Report to local police — they are the ones who investigate — and to the
   Canadian Anti-Fraud Centre.
     Online (any time):  https://reportcyberandfraud.canada.ca
     Phone:              1-888-495-8501, Mon-Fri 10:00-16:45 Eastern, closed holidays
   Verify the portal URL before relying on it — the national reporting system has
   changed address during rollout. The phone line is NOT 24/7; outside those hours
   use the online report and the bank's own 24-hour fraud line, which is the one
   that can actually stop the money.
   File an FBI IC3 complaint (https://www.ic3.gov) where there is any US nexus.
   Note the Financial Fraud Kill Chain applies to wires ORIGINATING from a US
   financial institution and sent internationally, above a threshold and reported
   within roughly 72 hours — a Canadian-originated wire into a US account does not
   qualify. File anyway; just do not expect the FFKC mechanism to engage.
3. Notify the insurer — fraud and social-engineering coverage usually sits under a
   different policy section than cyber, with its own notice requirement.
4. Notify the executive sponsor and counsel. Do not notify the counterparty until
   you know whether their mailbox is the compromised one.
5. Freeze any further payments to that vendor pending verification.
```

Preserve everything before anyone starts cleaning up: the original message with full headers, the
payment instruction and any attachment, the approval trail, and the finance system record of the
change.

### 15.2 Determine which side was compromised

This is the question that decides the rest of the response, and the answer is often "the vendor".

```bash
# Our side: did the attacker have access to our mailbox?
#  - Sign-in logs for the approver and anyone in the payment chain (Section 5.1)
#  - Inbox rules that hide or divert vendor correspondence — a rule filing all mail
#    from the vendor domain into an obscure folder is the classic tell
#  - Sent Items and message trace for replies the user did not write

# Their side: was the thread hijacked from a compromised vendor mailbox?
#  - Authentication passes, thread history is genuine, only the instructions changed
#  - Reply-To differs from From, or the domain is a lookalike registered recently
#  - Compare the sending IP/ASN with previous genuine mail from that vendor
```

Where the vendor was compromised, our mailbox is clean and a password reset here changes nothing.
Notify the vendor through a phone number from the master record — not one from the email thread —
and expect that other customers of theirs are being targeted with the same thread.

### 15.3 Controls worth fixing afterward

BEC is a process failure more than a technical one, and each of these has stopped a real loss:

- **Out-of-band verification is mandatory for any banking change**, using a number from the vendor
  master record. Not the number in the email. Not a number in the signature block.
- **Dual authorization** above a threshold, with the second approver required to verify independently
  rather than confirming the first approver's work.
- **No payment changes on the strength of email alone**, regardless of who appears to have sent it,
  including executives. Urgency and confidentiality in a payment request are the pattern, not an
  exception to it.
- **External sender banner** on all inbound mail, and an alert on inbound mail whose display name
  matches an internal executive.
- **Alert on new inbox rules** that forward externally or that move mail matching vendor domains.

*Example: invoice arrives in a two-month-old genuine thread, correct PO number, correct signature
block, one changed IBAN. DMARC passes because the vendor's mailbox really did send it. Nothing in
the mail is technically wrong — only the instructions changed.*

---

## 16. Lost or Stolen Device

### Tools: MDM console, identity provider, FileVault/BitLocker recovery records

The question is not whether the device is gone. It is whether the data on it was readable and what
credentials left with it.

### 16.1 First hour

```text
1. Confirm encryption status from the MDM or inventory record — the device is gone, so
   this is a query against your records, not a command you can run on it. Full-disk
   encryption with the device powered off is a significant mitigating factor in the
   harm assessment. It is NOT a safe harbour: the applicable privacy law has no statutory encryption
   exemption, so encryption is evidence feeding the RROSH determination, not the
   determination itself. See 0.5 — this is the privacy officer's call, not yours.
2. Remote lock, via MDM Lost Mode or Find My. Remote wipe only after deciding whether
   the device is evidence — a wipe is irreversible and may destroy what you need for
   the scope assessment. Check whether Activation Lock is enabled and whether you hold
   the bypass code; without it a recovered device may be unusable either way.
3. Run the full 5.3 containment sequence on the account, including the password reset.
   Cached credentials and browser-stored passwords are the whole point of this scenario,
   so do not shortcut to session revocation alone.
4. Revoke device-bound credentials: certificates, VPN profiles, SSH keys, MFA registration,
   push-approval enrolment. A stolen laptop with a live MFA registration is an identity incident.
5. Record whether the device was powered on and unlocked when lost. An unlocked device
   defeats encryption entirely, and this single fact drives the harm assessment.
```

```bash
# For the fleet, ahead of time — this is what makes step 1 answerable:
#   - FileVault/BitLocker state reported into MDM for every device
#   - Recovery key escrow confirmed present, not assumed
#
# On a device still in hand, verify properly. `fdesetup status` reports "On" while
# encryption is still converting, so check completeness too:
fdesetup status
diskutil apfs list | grep -iE "filevault|encryption|conversion"
```

### 16.2 Scope

- What was stored locally versus what was only accessible through now-revoked credentials.
- Whether local caches contained personal information — mail, downloads, exports, screenshots.
- Whether browser-stored credentials or session cookies were present.
- Whether the device held member or HR data, which starts the 0.5 assessment.

Unencrypted, or encrypted but taken while unlocked, and holding personal information: treat as a
privacy breach and notify the privacy officer. Encrypted, powered off, credentials revoked promptly:
document the reasoning and the evidence for it — that record is what supports the determination.

### 16.3 File the police report

Get a file number. Insurers and, where applicable, the regulator will ask for it, and it is far
harder to obtain weeks later.

---

## 17. Insider Threat / Departing Employee

### Tools: DLP and audit logs, MDM, identity provider, HR records

Handle this jointly with HR and counsel from the first step. Acting alone on suspicion of an
employee creates employment-law and privacy exposure of its own, and in a unionised workplace it
will also engage the collective agreement. Monitoring an individual employee is not a decision for
the security team to make by itself.

### 17.1 Planned departure — before the last day

```text
- Identify the access list ahead of the date: accounts, shared drives, SaaS apps,
  service accounts they administered, shared credentials they knew.
- Preserve the mailbox and home drive before any cleanup or reassignment.
- Time the revocation to the HR-confirmed moment, not to the ticket being opened.
- Collect devices and confirm return in writing.
```

### 17.2 Indicators worth investigating

Bulk activity shortly before a resignation is the pattern; individually these are all normal.

```bash
# Unusual volume of downloads or exports from file storage, close to the departure date
# Large personal-cloud uploads (Dropbox, personal Google Drive, iCloud, WeTransfer)
# Mail auto-forwarding to a personal address — check Section 5.1 and mean it
# Mass USB writes
# Access to systems outside the person's normal role or working hours
# Repository cloning, full CRM or member-list exports, print spool spikes
```

```bash
# macOS — recently connected removable volumes.
# diskarbitrationd is a PROCESS, not a logging subsystem; the subsystem== form
# silently matches nothing, which reads as "no USB activity" when piped to /dev/null.
log show --predicate 'process == "diskarbitrationd"' --last 7d 2>&1 | head -50

# The unified log holds days to a couple of weeks, not 30 — volume dependent, and it
# rolls silently. A --last 30d query returns only what survived, with no warning that
# the window was truncated. Collect logs AT the point of notice, not weeks later.

# Locally staged archives near the departure date.
# Parentheses are required here: -ls is an explicit action and without them it
# would bind to the last -name branch only.
find /Users/username \( -name "*.zip" -o -name "*.7z" \) -mtime -30 -size +50M -ls 2>/dev/null

# This only finds what is still there. Anything already copied off and deleted leaves
# no trace here — which is why the audit and DLP logs above, not the filesystem, are
# the primary source for this scenario.
```

### 17.3 Handling

- **Preserve first, confront later.** Notify the employee only when HR and counsel decide, not when
  you find the indicator. An early confrontation is what triggers destruction of evidence.
- **Proportionality matters.** Collect what the specific concern justifies and no more. Record the
  authorization for the collection alongside the collection itself.
- **Member and client data leaving the organization is a privacy breach** whoever took it. Section
  0.5 applies to insiders exactly as it does to attackers.

---

## 18. After the Incident

Closing the ticket is not closing the incident. Two things happen after containment, and skipping
them is why the same incident recurs.

### 18.1 Review

Within two weeks, while people still remember. Blameless — the aim is the system that allowed it,
not the person who clicked.

```text
Post-incident review — IR-YYYY-NNNN

Timeline:          first activity, first detection, first response, containment, closure
Detection gap:     how long between compromise and detection, and why
What worked:
What did not:
Root cause:        the condition that allowed it, not the immediate trigger
Contributing factors:
Actions:           owner and date for each — an action without both is not an action
Guide changes:     what in this document was wrong, missing, or slowed the response
```

That last line matters. Every incident is the best available test of this guide, and the corrections
are worth more than the notes.

### 18.2 Carry forward

- Add confirmed IOCs to blocklists and to the hunting set; record how long they stay.
- Note detections that should have fired and did not, and raise them as engineering work.
- Update the contact table in 0.2 if anything in it proved wrong at 2am.
- Confirm any regulatory or insurance obligation is closed out, not just the technical one.

---

## Quick Reference — Tool→Scenario Matrix

| Scenario | Primary Tools |
|----------|--------------|
| Malware triage | die.app, KnockKnock, TaskExplorer, Ghidra |
| Network anomaly | Wireshark, Zeek, Brim, Sniffnet |
| Web app attack | Burp/Caido, ZAP, Nuclei, testssl |
| Secrets in code | Trufflehog, Gitleaks |
| Account compromise | Identity provider audit logs, AD Check, lsof, Little Snitch, `last` |
| Ransomware | lsof, FileMonitor, Little Snitch, Autopsy, lldb/vmmap, VM snapshot |
| Phishing | ripmime, curl, zbarimg, oletools, PDF tools, Caido/Burp |
| Container security | Trivy, Syft, Grype, docker diff |
| Wi-Fi attack | WiFi Explorer Pro, Airtool 2, Wireshark, EAPTest |
| macOS persistence | KnockKnock, FileMonitor, PlistEdit Pro |
| Memory forensics | Volatility 3, Autopsy, TaskExplorer |
| Vuln scanning | Nuclei, Zenmap/Nmap, testssl |
| TLS issues | testssl, Wireshark, Caido |
| Hardening audit | Lynis, csrutil, spctl, fdesetup |
| BEC / payment fraud | Message trace, identity audit logs, bank fraud line, CAFC |
| Lost / stolen device | MDM console, fdesetup, identity provider |
| Insider / departure | DLP and audit logs, MDM, HR records |

---

*IR & Security Scenarios Field Guide | Revised September 16, 2026*

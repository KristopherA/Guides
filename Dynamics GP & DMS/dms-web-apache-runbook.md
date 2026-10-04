# DMS Web (dms.example.com) — Apache Server Runbook

> Status: initial draft, 2026-10-04. Contains [FILL IN] markers — see INDEX.md.

## Overview

DMS Web is the browser front end to the organization's Data Management System (DMS) — the FileMaker-based system that holds member, local, employer, event, expense and labour-relations data. It is published at **https://dms.example.com** and is used by two outside audiences:

- **Members** — submit expense claims, update contact information, register for events, view forms (see `DMS Manual/DMSWebForMembers.pdf`, `DMS Manual/EventsMembers.pdf`).
- **Local executives** — everything members can do, plus the Local Admin tab (executive and position assignments, membership and employer lists, approving/rejecting member expense claims, local events and attendance, accounts-payable payment requests, local funding and rebates, annual budget, votes) and the PRC tab (see `DMS Manual/DMSWebForExec.pdf`, `DMS Manual/EventsExec.pdf`, `DMS Manual/PRCSummaries.pdf`, `DMS Manual/LocalBudgetGuide.pdf`).

Users sign in with their **OrgNet** account (historically their FirstClass credentials; the 2014 manuals still show the old address `https://dms.example.org`). Staff mostly work in the DMS FileMaker client rather than DMS Web, but staff also use DMS Web's **Systems** menu for administration (Global Settings, Site Messages, Sessions, Shadow Login).

The single most important fact for whoever is on call: **"DMS Web is down" almost never means the DMS Web container is down.** The recurring failure is the Apache web server bundled inside **FileMaker Server on erpdb.example.com** crashing and failing to relaunch. DMS Web still loads, but logins and every database lookup fail. That failure, and its fix, is the headline of this runbook ([Runbook 1](#runbook-1--fms-apache-on-erpdb-has-gone-down-headline-incident)).

This guide covers the architecture, the known crash mode, the planned-maintenance ("temp DMS Web and reboot production") procedure, certificates, logs, health checks, monitoring, backup, DR and security. It does not cover the Dynamics GP application itself (`Dynamics GP & DMS/dynamics-gp-2018-admin-guide.md`) or generic Apache/TLS patterns (`Linux & Servers/reverse-proxy-and-tls-automation-guide.md`).

## Quick Facts

| Field | Value |
|---|---|
| Owner | IT lead (it@example.com), IT Ops. DMS application/FileMaker development: [FILL IN: current DMS developer(s) — the crash fix was originally implemented by a consultant (per the `ServerAdmin` line in the vhost KB); confirm whether they are still the escalation contact] |
| Environment | prod. Non-production DMS Web tier: [FILL IN: does a test/staging DMS Web exist? `backup-restore-testing-procedure.md` asks the same question] |
| Public URL | `https://dms.example.com` (DMS Web). Legacy: `https://dms.example.org` (2014 manuals) — [FILL IN: does the old name still redirect?] |
| Web tier host | `port.example.com` — DMS Web runs in a Docker container; project directory `/mnt/data/orgdocker/dms.example.com`, site mount `mounts/sites/dmsweb`, deployment script `mantisdeployment.php` (source: "How to Temp DMS Web" KB) |
| Database tier host | `erpdb.example.com` — macOS, FileMaker Server (FMS), admin account `fmadmin` (prompt `fmadmin@erpdb` in source notes). FMS version: [FILL IN: run `fmsadmin -v` or check Admin Console > Information] |
| The Apache that crashes | FileMaker Server's bundled Apache on erpdb: `/Library/FileMaker Server/HTTPServer/` — control script `bin/httpdctl`, config `conf/httpd.conf`, launched as `/usr/sbin/httpd -k start -D FILEMAKER -f "/Library/FileMaker Server/HTTPServer/conf/httpd.conf"` |
| Access | SSH `orgadmin@port.example.com` (web tier — run the deploy script as `orgadmin`, never root/sudo). SSH/console to erpdb as `fmadmin` [FILL IN: SSH enabled on erpdb? Screen Sharing? From which network/VPN?]. FMS Admin Console: [FILL IN: URL — `https://erpdb.example.com/admin-console` on FMS 19+, or `https://erpdb.example.com:16000` on older FMS]. DMS Web admin: log in as a Systems user → Systems → Global Settings. |
| Dependencies | FMS Apache on erpdb (TLS with the `erpdb.example.com` certificate); FileMaker Server database engine and the DMS files ("DMS 1.0" main file and others); a MySQL database holding DMS Web site messages (`MySQL_SiteMessages` layout) [FILL IN: which MySQL host/database]; Dynamics GP on SQL Server for finance/pay-stub data ("Dynamics access" toggle) [FILL IN: GP SQL Server hostname]; AppDB2 (`appdb2.example.com`) for user identity data ("Update From AppDB2", "AppDB2 User" on the DMS user record) [CONFIRM: AppDB2's exact role]; OrgNet accounts for member login; Active Directory `fm__dms__*` groups for staff FileMaker auth (`Active Directory/FileMaker_AD_External_Authentication_Troubleshooting_Runbook.txt`); DNS; pfSense NAT; public TLS certs |
| Dependents | Members and local executives (expense claims, contact updates, event registration, local administration); OrgNet account activation and password reset (toggles live in DMS Web Global Settings); DMS mobile apps share the backend (the login toggle does *not* affect them) |
| Monitoring | **None.** DMS Web and erpdb have no uptime, HTTP or certificate-expiry check today — see `SysAdmin Procedures/monitoring-alerting-guide.md` ("Not monitored at all"). Users report the outage before IT knows. |
| Secrets | FMS admin, `fmadmin`/`orgadmin` passwords, DMS Developer account, private keys: IT vault [FILL IN: vault entry names]. Never in this document. |
| Last reviewed | 2026-10-04 (initial draft — UNVERIFIED) |

## How It Works

### Request path

```
Member / exec browser
   │  https://dms.example.com
   ▼
public DNS → pfSense WAN → [FILL IN: NAT/proxy in front of port.example.com — Apache/Caddy/Traefik on the host, or published container port?]
   ▼
port.example.com  ── Docker container "DMS Web" (PHP site, /mnt/data/orgdocker/dms.example.com/mounts/sites/dmsweb)
   │               ├─ reads site messages from MySQL (MySQL_SiteMessages)          [CONFIRM]
   │               └─ reads/writes DMS data over HTTPS ─────────────┐
   ▼                                                                 ▼
                                             erpdb.example.com:443  FileMaker Server bundled Apache (TLS: erpdb.example.com cert)
                                                                 │  FileMaker web publishing / Data API  [CONFIRM which API]
                                                                 ▼
                                                           FileMaker Server database engine → DMS files
                                                                 │
                                                                 └─ Dynamics GP (SQL Server) for finance data  [CONFIRM: queried by FMS or by DMS Web directly]
```

Two separate Apache instances are in play, and confusing them wastes time:

| Apache | Where | What it serves | Who manages it |
|---|---|---|---|
| FMS bundled Apache | erpdb.example.com, `/Library/FileMaker Server/HTTPServer/` | FMS Admin Console (remote), FileMaker web publishing / Data API that DMS Web uses for every login and query | FileMaker Server installer — **overwritten on every FMS upgrade/reinstall** |
| DMS Web site server | inside the DMS Web container on port.example.com | The DMS Web PHP pages at dms.example.com | Container image / `mantisdeployment.php` [FILL IN: is the container's web server Apache (php:apache image) or nginx/php-fpm? Check with `docker compose ps` / `docker exec <ctr> apachectl -v`] |

macOS system Apache (`/etc/apache2/httpd.conf`, `sudo apachectl restart`) is a third, unrelated instance. The vhost pattern in `[internal KB: Apache virtual host setup]` (vhosts `devdms.example.com` → `/Library/WebServer-mantis/Documents`, `devwasp.example.com` → `/Library/WebServer-wasp/Documents`, port 80, no TLS) is the historical **developer-workstation** layout for the "mantis" (DMS Web) code base. It is not how production is served today. Do not use `apachectl` against the system Apache on erpdb thinking it controls FMS's Apache — it does not.

### Why the failure looks the way it does

DMS Web's PHP pages are served from port.example.com, so the login page renders even when erpdb is unreachable. As soon as the page needs data (login, any lookup), it calls FMS on erpdb through FMS's Apache. If that Apache is down, the call fails and the user sees a login failure or an error page. The FMS Admin Console is also served through the same Apache, so it becomes unreachable remotely at the same moment — a useful confirming symptom.

### Where things live

| Item | Location |
|---|---|
| FMS Apache control script | `/Library/FileMaker Server/HTTPServer/bin/httpdctl` |
| FMS Apache config | `/Library/FileMaker Server/HTTPServer/conf/httpd.conf` (plus `extra/`) |
| Cert/key Apache actually uses | `/Library/FileMaker Server/HTTPServer/conf/server.pem`, `server.key` |
| The organization's erpdb certificate files | same `conf/` dir: `erpdb.example.com.pem`, `erpdb.example.com_intermediate.pem`, `erpdb.example.com_key.pem` |
| FMS-generated originals (kept for reference) | `conf/server.fm_created.pem`, `conf/server.fm_created.key` |
| FMS Apache logs | [CONFIRM: `/Library/FileMaker Server/HTTPServer/logs/` — `error_log`, `access_log`, `ssl_*`] |
| FMS logs | [CONFIRM: `/Library/FileMaker Server/Logs/` — `Event.log`, `Access.log`, `fmdapi.log`/`wpe.log`] |
| FileMaker's own TLS cert (for FileMaker Pro clients) | Separate from the Apache files above; managed in Admin Console > Configuration > SSL Certificate [CONFIRM: `/Library/FileMaker Server/CStore/`] |
| DMS Web compose project | `port.example.com:/mnt/data/orgdocker/dms.example.com` |
| DMS Web site files | `port.example.com:/mnt/data/orgdocker/dms.example.com/mounts/sites/dmsweb` |
| DMS Web runtime switches | DMS Web → Systems → Global Settings (stored in [FILL IN: FileMaker or MySQL]) |
| Maintenance banner | `MySQL_SiteMessages` layout in the DMS 1.0 main file |

---

## Runbook 1 — FMS Apache on erpdb has gone down (headline incident)

**Known history.** FMS's Apache on erpdb crashes on its own roughly every two weeks (observed 2022; current frequency [FILL IN]). Normally it relaunches itself. It *fails* to relaunch when `server.key` and `server.pem` in the FMS Apache `conf/` directory do not match each other — Apache refuses to start with a certificate/key mismatch. FMS (re)creates those two files on every **install, reinstall or upgrade of FileMaker Server**, which re-introduces the mismatch. The fix (replace them with the organization's matching `erpdb.example.com` certificate and key) does **not** stop the crashes; it only lets Apache restart cleanly. So the pattern is: FMS upgrade → a couple of weeks later → Apache crashes → cannot restart → DMS Web logins fail.

### Symptoms

- DMS Web at https://dms.example.com loads, but logins fail and/or pages that need data error out.
- FMS Admin Console is not reachable remotely.
- FileMaker Pro clients inside the office may still work (they use the FileMaker protocol on 5003, not Apache) — this is a strong indicator it is Apache, not the database.
- Often within weeks of an FMS update.

### 1. Confirm it is FMS Apache (2 minutes)

From any workstation:

```
curl -sI https://dms.example.com | head -1                         # expect HTTP 200/302 — web tier up
curl -skI https://erpdb.example.com/ | head -1                      # no response / connection refused = FMS Apache down
nc -vz erpdb.example.com 443                                        # "refused" or timeout = nothing listening on 443
nc -vz erpdb.example.com 5003                                       # succeeds = FileMaker engine still up, Apache-only problem
```

On erpdb (as `fmadmin`):

```
pgrep -fl "httpd.*FILEMAKER"                                   # no output = FMS Apache not running
sudo lsof -nP -iTCP:443 -sTCP:LISTEN                           # who (if anyone) owns 443
sudo lsof -nP -iTCP:80 -sTCP:LISTEN                            # same for 80
```

If FMS Apache **is** running and listening, this is not Runbook 1 — go to [Troubleshooting](#troubleshooting).

### 2. Try the documented restart

```
cd "/Library/FileMaker Server/HTTPServer/bin"                  # FMS Apache control directory (note the space)
sudo ./httpdctl start                                          # starts FMS Apache
```

Expected output ends with a timestamped `========== start ==========` banner followed by the launch line:

```
/usr/sbin/httpd -k start -D FILEMAKER -f /Library/FileMaker Server/HTTPServer/conf/httpd.conf
```

Then verify:

```
pgrep -fl "httpd.*FILEMAKER"                                   # should now list httpd processes
curl -skI https://erpdb.example.com/ | head -1                      # should return an HTTP status line
```

Log in to DMS Web with a test account. If it works, you are done — record the crash (date/time) in `SysAdmin Procedures/Change_Management_Log.txt` or the incident log so the frequency is tracked.

### 3. If it will not start: check the cert/key pair

```
cd "/Library/FileMaker Server/HTTPServer/conf"                 # FMS Apache config directory
ls -l server.* erpdb.example.com*                                   # current files and their dates
sudo /usr/sbin/httpd -t -D FILEMAKER -f "/Library/FileMaker Server/HTTPServer/conf/httpd.conf"   # config test; look for SSL key/cert errors
sudo openssl x509 -noout -pubkey -in server.pem | openssl md5       # public key from the cert
sudo openssl pkey -pubout -in server.key | openssl md5             # public key from the private key — must match the line above
sudo openssl x509 -noout -subject -issuer -enddate -in server.pem   # whose cert is this? FMS self-signed vs erpdb.example.com
tail -50 "/Library/FileMaker Server/HTTPServer/logs/error_log"     # [CONFIRM path] — look for "key values mismatch" / "SSLCertificateKeyFile"
```

If the two md5 lines differ, or the subject is not `erpdb.example.com`, or `server.fm_created.*` is older than `server.*` with a recent FMS upgrade in between, apply the fix.

### 4. The fix — replace FMS's cert/key with the organization's (needed after every FMS upgrade)

```
cd "/Library/FileMaker Server/HTTPServer/conf"                         # FMS Apache config directory
sudo cp -p server.key server.fm_created.key.$(date +%F)                # extra dated copy before overwriting anything
sudo cp -p server.pem server.fm_created.pem.$(date +%F)                # extra dated copy of the FMS cert
sudo mv server.key server.fm_created.key                               # keep what FMS generated (original procedure)
sudo mv server.pem server.fm_created.pem                               # keep what FMS generated (original procedure)
sudo cp erpdb.example.com_key.pem server.key                                # organization private key -> name Apache expects
sudo cp erpdb.example.com.pem server.pem                                    # organization certificate -> name Apache expects
sudo openssl x509 -noout -pubkey -in server.pem | openssl md5          # re-check the pair
sudo openssl pkey -pubout -in server.key | openssl md5                 # must match
sudo /usr/sbin/httpd -t -D FILEMAKER -f "/Library/FileMaker Server/HTTPServer/conf/httpd.conf"   # expect "Syntax OK"
cd "/Library/FileMaker Server/HTTPServer/bin"                          # back to the control script
sudo ./httpdctl start                                                  # start FMS Apache
```

Notes:

- The original procedure used plain `mv`/`cp` without `sudo` because it was run in a session that already had rights; use `sudo` if you get "Permission denied". Preserve the existing ownership and mode of the files — compare with `ls -l` before and after. [CONFIRM: correct owner/mode for `server.key` — expected root- or fmserver-owned, not world-readable.]
- If the `erpdb.example.com*` files themselves are missing (wiped by a full uninstall), restore them from the IT vault / certificate store — see [Certificates](#tls-certificates).
- [CONFIRM: does `server.pem` need the intermediate appended (`cat erpdb.example.com.pem erpdb.example.com_intermediate.pem > server.pem`) or does `httpd.conf`/`extra/httpd-ssl.conf` reference `erpdb.example.com_intermediate.pem` via `SSLCertificateChainFile`? Check with `grep -R "SSLCertificate" "/Library/FileMaker Server/HTTPServer/conf/"`, then verify the chain externally with the `openssl s_client` command in [Health checks](#health-checks).]
- Once fixed, it stays fixed until the next FMS install/upgrade (confirmed in the source notes). An earlier attempt made the files read-only so FMS could not overwrite them; it is not known whether that held. [CONFIRM: proposed default — do **not** rely on file flags (`chflags uchg`), which can make an FMS upgrade fail; instead make the cert swap a mandatory post-upgrade step (see [Runbook 2](#runbook-2--planned-maintenance-temp-dms-web-and-reboot-production), step 9).]

### 5. Still down?

- Port conflict: `sudo lsof -nP -iTCP:443 -sTCP:LISTEN` shows something other than FMS's httpd (e.g. macOS system Apache started by someone) → `sudo apachectl stop` for the *system* Apache, then retry `httpdctl start`.
- Stale PID file after a crash: `ls "/Library/FileMaker Server/HTTPServer/logs/"*.pid` [CONFIRM path] → if httpd is not running but a pid file exists, remove it and retry.
- FMS itself unhealthy: `sudo fmsadmin list files` (prompts for FMS admin credentials from the vault) — if the engine is down, this is a database incident, not an Apache one. Escalate per `SysAdmin Procedures/business-continuity-plan.md` Scenario 5.
- Last resort: a full reboot of erpdb using the controlled close in [Runbook 2](#runbook-2--planned-maintenance-temp-dms-web-and-reboot-production) steps 4–7. Never hard-power-cycle erpdb with databases open.

---

## Runbook 2 — Planned maintenance: temp DMS Web and reboot production

Use this for FMS updates, macOS updates on erpdb, Dynamics GP updates that touch DMS, or any planned reboot of erpdb. Source: "How to Temp DMS Web and reboot Production" KB (updated 2024-09-26).

### Lead time and communication

| When | Action |
|---|---|
| [CONFIRM: proposed default — 5 business days before] | Email **Staff Mail List**, **Regional Offices**, and **News** with date, start time, expected duration, and what will be unavailable (DMS Web, FileMaker client, mobile apps if affected). |
| Same time | Put the maintenance banner up on DMS Web (below) so members and executives see it at login. |
| Morning of | Reminder to staff list. Confirm finance/payroll is not mid-run if Dynamics is involved. |
| After | All-clear email to the same lists; take the banner down. |

[FILL IN: avoid windows — payroll run dates, dues/rebate processing, election/vote periods, AGM registration deadlines.]

### Step 1 — Display the maintenance message on DMS Web

1. Log into DMS (FileMaker client) as **Developer** (credentials in IT vault).
2. Go to the **MySQL_SiteMessages** layout in the **DMS 1.0** main file.
3. Check existing records — the first record is the usual one for downtime notices; reuse it if the wording fits.
4. Update the **Message** field.
5. Set **DateStart** to today and **DateEnd** to the maintenance date. Both are evaluated inclusively, so a notice posted several days early runs from today through the day of the work.
6. To remove the message later, set DateStart and DateEnd to dates in the past.

(DMS Web also has Systems → **Site Messages**, which appears to be the web-side editor for the same records [CONFIRM].)

### Step 2 — Optional: stop logins without temping

DMS Web → **Systems → Global Settings** → **DMS Web** → toggle **"DMS Web logins are allowed"** off. This blocks DMS Web logins only — it does **not** affect the mobile apps. Other switches on the same page:

| Switch | Effect when off | When to use |
|---|---|---|
| DMS Web logins are allowed | Web logins refused; mobile apps unaffected | Short maintenance where a full temp page is overkill |
| OrgNet: Account activations are allowed | New OrgNet activations blocked | Identity/directory maintenance |
| OrgNet: Password resets and changes are allowed | Resets/changes blocked | Identity/directory maintenance, or during a credential incident |
| Blobs can be accessed and uploaded | Document attachments unavailable | Storage maintenance on the attachment store |
| Dynamics access is enabled | DMS stops reaching Dynamics GP | **Before any Dynamics GP update or GP SQL Server maintenance** (this is what `DMS Settings for Dynamics update.png` documents) — turn back on afterwards |

### Step 3 — Temp DMS Web (maintenance page)

On the web tier — **as `orgadmin`, not root, not with sudo**:

```
ssh orgadmin@port.example.com                                        # web tier host
cd /mnt/data/orgdocker/dms.example.com                               # DMS Web compose project
ls -l mantisdeployment.php                                      # the deployment script should be here
./mantisdeployment.php temp mounts/sites/dmsweb                 # switch DMS Web to the temporary/maintenance site
curl -sI https://dms.example.com | head -1                           # confirm the site still answers (now the temp page)
```

[CONFIRM: exactly what `temp` does — swaps the document root to a holding page, or stops the app container? Record it here once read from the script.]

### Step 4 — Before an FMS upgrade: export backup schedules

An in-place FMS **update** preserves backup schedules; a full **uninstall/reinstall or major-version upgrade** will likely wipe them. Export or screenshot every schedule (Admin Console → Configuration → Schedules) before starting. [FILL IN: where exported schedules are kept.]

Also record, before the upgrade, anything else the installer resets: the FMS Apache `server.key`/`server.pem` (will be overwritten — see step 9), external-authentication settings for `fm__dms__*` groups, and the FileMaker SSL certificate.

### Step 5 — Limit connections

On erpdb, use **The Missing Admin** tool to set **Client Connections to zero** before closing databases. This stops open FileMaker Pro clients auto-reconnecting the moment files are closed. [FILL IN: where The Missing Admin is installed and how it authenticates.]

### Step 6 — Close databases

1. Log into the **FMS Admin Console**.
2. **Activity** → folder icon → **Close All**.
3. Specify a delay and a message that connected users will see.
4. Closing can take a while. Force-disconnect stragglers under the **Clients** tab if needed.

CLI equivalent on erpdb (prompts for FMS admin credentials):

```
sudo fmsadmin list clients                                      # who is still connected
sudo fmsadmin close -m "DMS maintenance — please save and exit" -t 300   # close all files after 5-minute warning [CONFIRM flags on installed FMS version]
sudo fmsadmin list files                                        # confirm all files are Closed
```

### Step 7 — Do the work / reboot

```
softwareupdate -l                                               # list pending macOS updates (if this is an OS patch window)
sudo shutdown -r now                                            # reboot erpdb only after all files are closed
```

### Step 8 — Bring it back

```
sudo fmsadmin list files                                        # files should auto-open if set to; otherwise open them
sudo fmsadmin open                                              # open all hosted files [CONFIRM on installed version]
pgrep -fl "httpd.*FILEMAKER"                                    # FMS Apache running after boot
curl -skI https://erpdb.example.com/ | head -1                       # FMS Apache answering
```

Restore **Client Connections** to the normal value in The Missing Admin ([FILL IN: normal value]).

### Step 9 — After any FMS install/upgrade: re-apply the cert/key fix

Mandatory. Skipping this is the direct cause of the next "Apache has gone down" call two weeks later.

```
cd "/Library/FileMaker Server/HTTPServer/conf"                  # FMS Apache config directory
ls -l server.* server.fm_created.*                              # newer server.* than fm_created = FMS rewrote them
sudo openssl x509 -noout -subject -in server.pem                # if not CN=erpdb.example.com, re-run Runbook 1 step 4
```

Re-run [Runbook 1, step 4](#4-the-fix--replace-fmss-certkey-with-the-organizations-needed-after-every-fms-upgrade) if required, then restart FMS Apache. Re-import the backup schedules if they were lost.

### Step 10 — Untemp, verify, announce

```
ssh orgadmin@port.example.com                                        # web tier host
cd /mnt/data/orgdocker/dms.example.com                               # DMS Web compose project
./mantisdeployment.php untemp mounts/sites/dmsweb               # restore the live DMS Web site
curl -sI https://dms.example.com | head -1                           # expect normal response
```

- Log in to DMS Web with a member test account and an executive test account ([FILL IN: test accounts]); open a member list and an expense claim.
- Re-enable any Global Settings switches you turned off (especially **Dynamics access**).
- Clear the maintenance message (DateStart/DateEnd in the past).
- Send the all-clear email.
- Log the change in `SysAdmin Procedures/Change_Management_Log.txt`.

---

## Configuration

### FMS Apache (erpdb)

FMS owns this Apache; treat `httpd.conf` as installer-managed. Changes made by hand are lost on upgrade, just like the cert files. Record any deliberate change here.

| Setting | Value | Reason |
|---|---|---|
| `server.pem` / `server.key` | Copies of `erpdb.example.com.pem` / `erpdb.example.com_key.pem` | FMS-generated pair mismatches and prevents Apache auto-relaunch after a crash |
| `server.fm_created.*` | FMS's originals, kept | Reference in case FMS needs them; lets us diff after an upgrade |
| Admin Console exposure | [FILL IN: reachable from internet, from office LAN only, or VPN only?] | Admin Console is served through this Apache; it should not be internet-reachable [CONFIRM: restrict to management network] |
| Ports | 443 (and 80 for redirect) [CONFIRM]; 5003 FileMaker protocol (not Apache) | 443 is what DMS Web uses to reach FMS |
| Other `httpd.conf` changes | [FILL IN: none known — diff against `httpd.conf.2.4` to check] | |

```
diff "/Library/FileMaker Server/HTTPServer/conf/httpd.conf" "/Library/FileMaker Server/HTTPServer/conf/httpd.conf.2.4"   # spot local edits vs the shipped copy
grep -R -n -e "SSLCertificate" -e "^Listen" "/Library/FileMaker Server/HTTPServer/conf/"                                # which cert files and ports Apache actually uses
```

### DMS Web container (port.example.com)

| Setting | Value | Reason |
|---|---|---|
| Compose project | `/mnt/data/orgdocker/dms.example.com` | Per KB |
| Site mount | `mounts/sites/dmsweb` | Target of `mantisdeployment.php temp/untemp` |
| Deploy user | `orgadmin` (not root, not sudo) | Script expects orgadmin's ownership/permissions; running as root leaves root-owned files the container cannot manage [CONFIRM reason] |
| Web server / vhost | [FILL IN: Apache vhost or other inside the container; `ServerName dms.example.com`; DocumentRoot] | |
| TLS for dms.example.com | [FILL IN: terminated where — host Apache/proxy on port.example.com, a container, or the Barracuda WAF?] | |
| FMS endpoint / credentials | [FILL IN: config file inside the site that names erpdb.example.com; credentials in vault, not here] | |

Inspect it:

```
cd /mnt/data/orgdocker/dms.example.com && docker compose ps            # container names and state
docker compose config --services                                  # services in the project
docker compose logs --since 1h                                    # recent container output
docker exec <container> apachectl -S                              # if Apache: show vhosts/ServerName/DocumentRoot [CONFIRM image uses Apache]
docker exec <container> apachectl -t                              # if Apache: config syntax test
```

### Historical vhost pattern (reference only)

`[internal KB: Apache virtual host setup]` documents enabling `Include /private/etc/apache2/extra/httpd-vhosts.conf` in macOS `/etc/apache2/httpd.conf` and defining port-80 vhosts with per-vhost `ErrorLog`/`CustomLog` named `<hostname>-error_log` / `<hostname>-access_log` under `/private/var/log/apache2/`, then `sudo apachectl restart`. That was the developer setup for `devdms.example.com` (mantis/DMS Web) and `devwasp.example.com`. Keep the log-naming convention for any new vhost; for the modern TLS/proxy pattern use `Linux & Servers/reverse-proxy-and-tls-automation-guide.md`.

---

## TLS certificates

Two certificates matter. Track both in the inventory in `Security & Hardening/certificate-pki-lifecycle-guide.md`.

| Certificate | Used by | Files | Issuer / renewal | Expiry |
|---|---|---|---|---|
| `erpdb.example.com` | FMS Apache on erpdb (DMS Web → FMS traffic, Admin Console) | `/Library/FileMaker Server/HTTPServer/conf/erpdb.example.com.pem`, `_intermediate.pem`, `_key.pem`, copied to `server.pem`/`server.key` | [FILL IN: public CA or internal CA; manual reissue (not certbot — macOS/FMS)] | [FILL IN] |
| `dms.example.com` | Public DMS Web site | [FILL IN: on port.example.com or WAF] | [FILL IN: Let's Encrypt/certbot vs wildcard `*.example.com`] | [FILL IN] |
| FileMaker SSL cert | FileMaker Pro clients → FMS (port 5003) | Admin Console → SSL Certificate | [FILL IN] | [FILL IN] |

Check expiry from anywhere:

```
echo | openssl s_client -connect erpdb.example.com:443 -servername erpdb.example.com 2>/dev/null | openssl x509 -noout -subject -issuer -enddate   # FMS Apache cert
echo | openssl s_client -connect dms.example.com:443 -servername dms.example.com 2>/dev/null | openssl x509 -noout -subject -issuer -enddate       # public DMS Web cert
echo | openssl s_client -connect erpdb.example.com:443 -servername erpdb.example.com -showcerts 2>/dev/null | grep -c "BEGIN CERTIFICATE"         # >=2 means the chain is being sent
```

### Renewing the erpdb.example.com certificate

1. Generate the CSR on erpdb so the key never leaves the host (`certificate-pki-lifecycle-guide.md` policy):
   ```
   cd "/Library/FileMaker Server/HTTPServer/conf"                                      # keep new files beside the old
   sudo openssl req -new -newkey rsa:2048 -nodes -keyout erpdb.example.com_key.new.pem -out erpdb.example.com.csr -subj "/CN=erpdb.example.com"   # new key + CSR
   ```
   [CONFIRM: subject fields and SANs required by the issuing CA.]
2. Submit the CSR; save the returned cert and intermediate as `erpdb.example.com.new.pem` / `erpdb.example.com_intermediate.new.pem`.
3. Verify the new pair before touching anything live:
   ```
   sudo openssl x509 -noout -pubkey -in erpdb.example.com.new.pem | openssl md5            # cert public key
   sudo openssl pkey -pubout -in erpdb.example.com_key.new.pem | openssl md5               # must match
   ```
4. Keep the old files (`*.old.$(date +%F)`), rename the new ones to the canonical `erpdb.example.com*` names, then re-copy to `server.pem`/`server.key` exactly as in Runbook 1 step 4. **Renewing the `erpdb.example.com*` files alone does nothing** — Apache reads `server.*`.
5. Restart FMS Apache in a quiet period. [CONFIRM: does `sudo ./httpdctl stop` then `sudo ./httpdctl start` work, or does `httpdctl` support `restart`/`graceful`? Run `sudo ./httpdctl` with no argument to see its usage.]
6. Run the `openssl s_client` check; log in to DMS Web.
7. Store the new key in the IT vault; update the inventory; add a calendar reminder 30 days before the next expiry.

If FileMaker Pro clients should also present the organization cert, import it via Admin Console → Configuration → SSL Certificate (or `fmsadmin certificate import`) — this is a separate store from the Apache files and requires an FMS restart [CONFIRM].

---

## Logs

| Log | Location | Use it for |
|---|---|---|
| FMS Apache error log | [CONFIRM: `/Library/FileMaker Server/HTTPServer/logs/error_log`] | Crash and failed-start reasons, SSL key mismatch |
| FMS Apache access log | [CONFIRM: same dir, `access_log` / `ssl_request_log`] | Whether DMS Web's requests are reaching FMS |
| `httpdctl` output | Terminal; [CONFIRM: also appended to a log in `HTTPServer/logs/`] | Start/stop timestamps |
| FMS event log | [CONFIRM: `/Library/FileMaker Server/Logs/Event.log`] | Database open/close, scheduled backups, errors |
| FMS access log | [CONFIRM: `/Library/FileMaker Server/Logs/Access.log`] | Client connects/disconnects, auth failures |
| macOS crash reports | `/Library/Logs/DiagnosticReports/` | `httpd` crash reports — evidence for the ~2-week crash cycle |
| DMS Web container | `docker compose logs` in `/mnt/data/orgdocker/dms.example.com` | PHP errors, failed calls to erpdb |
| DMS Web sessions | DMS Web → Systems → Sessions | Who is logged in; confirm drain before maintenance |
| DMS user change history | DMS FileMaker client → Database Admin → Users → Logs / Errors / Changes to this User | Per-user problems |

```
tail -f "/Library/FileMaker Server/HTTPServer/logs/error_log"                     # live FMS Apache errors [CONFIRM path]
ls -lt /Library/Logs/DiagnosticReports/ | grep -i httpd | head                    # recent httpd crash reports
log show --last 1h --predicate 'process == "httpd"' | tail -50                    # unified log entries for httpd
cd /mnt/data/orgdocker/dms.example.com && docker compose logs --since 30m --tail 200   # web tier, on port.example.com
```

[CONFIRM: proposed default — ship the FMS Apache error log and the DMS Web container log to Graylog (graylog01.example.com) via Sidecar so crashes are searchable.]

---

## Health checks

Quick sweep, outside in:

```
curl -sI https://dms.example.com | head -1                                          # web tier answers (200/302)
curl -s -o /dev/null -w '%{http_code} %{time_total}s\n' https://dms.example.com/    # status + response time
curl -skI https://erpdb.example.com/ | head -1                                      # FMS Apache answers
curl -sk https://erpdb.example.com/fmi/data/vLatest/productInfo                     # FMS Data API alive (no auth needed) [CONFIRM: FMS 19+ and Data API enabled]
nc -vz erpdb.example.com 443                                                        # TLS port open
nc -vz erpdb.example.com 5003                                                       # FileMaker protocol open (engine up)
```

On erpdb:

```
pgrep -fl "httpd.*FILEMAKER"                                                   # FMS Apache processes
sudo lsof -nP -iTCP -sTCP:LISTEN | grep -e httpd -e fmserver                   # listening ports by process
sudo /usr/sbin/httpd -t -D FILEMAKER -f "/Library/FileMaker Server/HTTPServer/conf/httpd.conf"   # config syntax
sudo fmsadmin list files -s                                                    # hosted files and status
df -h /                                                                        # disk space on the FMS volume
```

On port.example.com:

```
cd /mnt/data/orgdocker/dms.example.com && docker compose ps                         # container Up/healthy
docker stats --no-stream                                                       # CPU/memory per container
df -h /mnt/data                                                                # space on the Docker data volume
```

Functional check (the one that actually matters): log in to DMS Web with a test account and open a page that queries data. Login page rendering alone proves nothing (see How It Works).

---

## Troubleshooting

### Symptom: DMS Web loads, logins fail, Admin Console unreachable
- Likely cause: FMS Apache on erpdb down.
- Check: `curl -skI https://erpdb.example.com/ | head -1`   # no response = Apache down
- Fix: [Runbook 1](#runbook-1--fms-apache-on-erpdb-has-gone-down-headline-incident).

### Symptom: `httpdctl start` runs but Apache dies immediately
- Likely cause: `server.key`/`server.pem` mismatch after an FMS upgrade.
- Check: `sudo openssl x509 -noout -pubkey -in server.pem | openssl md5; sudo openssl pkey -pubout -in server.key | openssl md5`   # differing hashes = mismatch
- Fix: Runbook 1 step 4.

### Symptom: dms.example.com itself does not load (timeout / 502 / connection refused)
- Likely cause: web tier — container stopped, host proxy down, or site left in `temp`.
- Check: `cd /mnt/data/orgdocker/dms.example.com && docker compose ps`   # Exit/Restarting = container problem
- Fix: `docker compose up -d` in the project directory; if a maintenance page shows when it should not, `./mantisdeployment.php untemp mounts/sites/dmsweb` as orgadmin. For proxy 502s see `Linux & Servers/reverse-proxy-and-tls-automation-guide.md`.

### Symptom: browser certificate warning on dms.example.com
- Likely cause: expired or incomplete-chain cert on the public site.
- Check: `echo | openssl s_client -connect dms.example.com:443 -servername dms.example.com 2>/dev/null | openssl x509 -noout -enddate`
- Fix: renew per [TLS certificates](#tls-certificates) and `certificate-pki-lifecycle-guide.md`.

### Symptom: logins fail with a TLS/SSL error in the container log, FMS Apache is up
- Likely cause: `erpdb.example.com` cert expired, chain missing, or FMS self-signed cert back in `server.pem`.
- Check: `echo | openssl s_client -connect erpdb.example.com:443 -servername erpdb.example.com 2>/dev/null | openssl x509 -noout -subject -enddate`   # subject must be erpdb.example.com and not expired
- Fix: re-copy or renew the cert (Runbook 1 step 4 / renewal procedure).

### Symptom: a single user cannot log in or sees a broken/blank layout
- Likely cause: that user's saved UI state is corrupt, account inactive, or wrong privileges.
- Check: DMS FileMaker client → **Database Admin → Users** → find the user → Status = Active; check **Errors** and **Access** tabs.
- Fix: **Current UI** tab → **Reset To Default** (or **Reset UI Windows**) — see `Reset DMS User UI.png`. For staff privilege mismatches see `Active Directory/FileMaker_AD_External_Authentication_Troubleshooting_Runbook.txt`.

### Symptom: members cannot activate OrgNet accounts or reset passwords
- Likely cause: Global Settings switch left off after maintenance.
- Check: DMS Web → Systems → Global Settings → OrgNet section.
- Fix: turn **Account activations** / **Password resets and changes** back on.

### Symptom: finance or pay-stub data missing/erroring in DMS
- Likely cause: **Dynamics access is enabled** switched off (e.g. after a GP update) or Dynamics GP SQL Server unavailable.
- Check: Global Settings → Dynamics section; GP SQL Server reachable [FILL IN: host/port test].
- Fix: re-enable the switch; for GP itself see `dynamics-gp-2018-admin-guide.md`.

### Symptom: document attachments will not open/upload
- Likely cause: **Blobs can be accessed and uploaded** off, or attachment storage full.
- Check: Global Settings → Blobs; `df -h` on [FILL IN: blob storage host/volume].
- Fix: re-enable the switch / free space.

### Symptom: maintenance banner still showing after work is done
- Check: `MySQL_SiteMessages` record DateEnd.
- Fix: set DateStart/DateEnd to past dates.

---

## Monitoring & Alerting

**Current state: nothing monitors DMS Web or erpdb.** `SysAdmin Procedures/monitoring-alerting-guide.md` lists service uptime / HTTP reachability of public services as an unmonitored, high-impact gap, and DMS Web is not in its coverage matrix at all. Every past Apache crash was reported by users.

Proposed minimum [CONFIRM: proposed default — add rows to the monitoring guide's coverage matrix when implemented]:

| Check | Target | Interval | Alert when | Why |
|---|---|---|---|---|
| HTTP reachability | `https://dms.example.com/` | 1 min | non-2xx/3xx for 3 consecutive checks | Web tier down |
| FMS Apache reachability | `https://erpdb.example.com/` (or `/fmi/data/vLatest/productInfo`) from port.example.com | 1 min | no response for 3 checks | **The known crash** — catches it before users |
| Cert expiry | dms.example.com, erpdb.example.com | daily | < 30 days | Manual-reissue certs fail silently |
| Disk | erpdb `/`, port.example.com `/mnt/data` | 15 min | > 80% warn, > 90% page | FMS and Docker both fail badly on full disk |
| FMS backups | FMS schedule result | daily | failed/missed | Backup schedules can vanish on upgrade |

A self-healing watchdog on erpdb is reasonable because the failure and fix are well understood. Proposal (launchd job every 2 minutes, logs each restart) [CONFIRM before deploying — it masks crashes, so it must log and alert, not just restart]:

```
#!/bin/zsh
# /usr/local/sbin/fms-apache-watchdog.sh — restart FMS Apache if it is not running
if ! pgrep -f "httpd.*FILEMAKER" >/dev/null; then                         # FMS Apache absent
  logger -t fms-apache-watchdog "FMS Apache not running - attempting start"   # leave a trace in the unified log
  cd "/Library/FileMaker Server/HTTPServer/bin" && ./httpdctl start           # documented start command (runs as root via launchd)
fi
```

The watchdog will not fix a cert/key mismatch — it will just log repeated failures, which is itself the alert you want.

---

## Backup & Restore

| What | Where | Mechanism | Status |
|---|---|---|---|
| DMS FileMaker databases | erpdb | FMS scheduled backups [FILL IN: schedule, destination, retention, offsite via Retrospect?] | [FILL IN] |
| FMS backup schedule definitions | erpdb | Export before any FMS upgrade (Runbook 2 step 4) | [FILL IN: export location] |
| FMS Apache cert/key files | erpdb `conf/` + IT vault | Manual copy | [FILL IN: confirm vault copy exists] |
| DMS Web compose project + site | port.example.com `/mnt/data/orgdocker/dms.example.com` | [FILL IN: backup of `/mnt/data`? git repo for the mantis code?] | [FILL IN] |
| Site messages / settings in MySQL | [FILL IN: MySQL host] | per `Databases/mysql-administration-guide.md` | [FILL IN] |

DMS is Tier 1 in `Hardware & Backup/backup-restore-testing-procedure.md` — follow that procedure for restore testing. Restored copies contain member PII; handle accordingly.

**Library inconsistency to resolve:** `Databases/mysql-administration-guide.md`, `SysAdmin Procedures/monitoring-alerting-guide.md` and `Hardware & Backup/backup-restore-testing-procedure.md` describe erpdb.example.com as a MySQL host (`/var/lib/mysql`, ODBC). The DMS source notes show erpdb is a **macOS FileMaker Server** host. [CONFIRM: does erpdb also run MySQL, or should those rows be corrected? Where does Dynamics GP's SQL Server actually live?]

---

## Disaster Recovery

- RTO/RPO: [FILL IN — DMS is Tier 1; see `SysAdmin Procedures/business-continuity-plan.md` Scenario 5]. [CONFIRM: proposed default — RTO 4 business hours for DMS Web, RPO = last FMS backup (target ≤ 24 h).]
- An FMS Apache crash is **not** a DR event: it is fixed in minutes with Runbook 1.

**erpdb lost** — rebuild order:
1. macOS host (hardware/VM [FILL IN: physical Mac or VM?]) with the same hostname and IP; DNS unchanged.
2. Install FileMaker Server (licence details are in the vault — `[internal: FileMaker licence download page]` is the 2018 download record; do not copy keys into docs).
3. Restore the DMS files from the latest FMS backup; re-create backup schedules from the export.
4. Re-configure external authentication (`fm__dms__*` AD groups) and FileMaker SSL cert.
5. Place `erpdb.example.com*` cert files from the vault into `HTTPServer/conf/` and apply Runbook 1 step 4 (fresh install = mismatched pair guaranteed).
6. Health checks; DMS Web functional test.

**port.example.com / DMS Web container lost** — restore `/mnt/data/orgdocker/dms.example.com` to a Docker host (see `Linux & Servers/docker-migration-procedure.md`), `docker compose up -d`, repoint DNS/proxy for dms.example.com, re-issue or restore the dms.example.com cert. Database data is unaffected.

Escalation if the owner is unavailable: [FILL IN: second IT contact → DMS developer → FileMaker partner/Claris support].

---

## Security

- **Exposure:** dms.example.com is internet-facing for members. erpdb's FMS Apache must be reachable from port.example.com; whether it is reachable from the internet is [FILL IN]. The KB says the Admin Server "will not be accessible remotely" when Apache is down — confirm "remotely" means the office LAN/VPN, not the internet. [CONFIRM: proposed default — firewall erpdb:443 to port.example.com and the IT management network only; see `Networking Guide/firewall-change-procedure.md`.]
- **Data sensitivity:** member PII, local finance, labour-relations data; Dynamics GP pay data. Treat any compromise as a privacy incident (`Security Procedures/ransomware-response-playbook.md` flags erpdb as highest priority).
- **Authentication:** members/executives via OrgNet accounts; staff FileMaker access via AD external auth (`fm__dms__*`). DMS Web **Shadow Login** lets an admin act as a user — [CONFIRM: proposed default — restrict to named Systems staff and confirm it is logged].
- **Credential-incident levers:** Global Settings can stop web logins and OrgNet password resets instantly without taking the site down.
- **Private keys:** `erpdb.example.com_key.pem` and its copy `server.key` exist on erpdb, plus `server.fm_created.key`. Check they are not world-readable (`ls -l "/Library/FileMaker Server/HTTPServer/conf/"*.key "/Library/FileMaker Server/HTTPServer/conf/"*key*.pem`). Master copy in the IT vault.
- **Accounts:** `fmadmin` (erpdb), `orgadmin` (port), FMS admin, DMS Developer — all in the IT vault; rotate after staff departures (`SysAdmin Procedures/Onboarding_Offboarding_Checklists.txt`).
- **Patching:** FMS updates trigger the cert/key reset — every FMS patch is a planned Runbook 2 window, never an unattended update. [CONFIRM: disable FMS/macOS auto-update on erpdb.]

---

## Decisions & History

| Date | Decision / Change | Why / Source |
|---|---|---|
| 2014 | DMS Web at `https://dms.example.org`, login with FirstClass credentials | `DMS Manual/DMSWebForExec.pdf` |
| ~2021–2022 | FMS Apache `server.key`/`server.pem` replaced with `erpdb.example.com` cert/key; FMS originals kept as `server.fm_created.*` | Apache crashing every ~2 weeks and failing to relaunch on cert/key mismatch — `DMSWeb Apache crash solution.txt` |
| 2022 (dated 2022-08-04 file) | Recurrence after an FMS upgrade; confirmed fix is only needed after FMS install/upgrade | same |
| 2022-06-10 | Documented restart: `sudo ./httpdctl start` | [internal KB: Apache down on DMS web] (updated 2022-06-11) |
| 2024-09-26 | Temp/untemp via `mantisdeployment.php` on port.example.com; FMS backup-schedule export warning; Missing Admin connection limit | [internal KB: temp DMS web and reboot production] |
| 2026-10-04 | This runbook consolidates the above | — |

## References

- `Dynamics GP & DMS/DMSWeb Apache crash solution.txt` — original crash diagnosis and cert/key fix
- `[internal KB: Apache down on DMS web]` — restart procedure (kb.example.com)
- `[internal KB: temp DMS web and reboot production]` — maintenance procedure (kb.example.com)
- `Dynamics GP & DMS/DMS Settings for Dynamics update.png` — Global Settings page and Dynamics access toggle
- `Dynamics GP & DMS/Reset DMS User UI.png` — Database Admin → Users → Current UI → Reset To Default
- `Dynamics GP & DMS/DMS Manual/` — end-user manuals (DMSWebForExec, DMSWebForMembers, Events, PRC, Local Budget)
- `Dynamics GP & DMS/dynamics-gp-2018-admin-guide.md` — Dynamics GP administration
- `[internal KB: Apache virtual host setup]` — historical macOS vhost pattern (consultant dev vhosts / mantis)
- `Linux & Servers/reverse-proxy-and-tls-automation-guide.md` — Apache/TLS patterns, 502/503 troubleshooting
- `Linux & Servers/docker-migration-procedure.md` — moving the DMS Web compose project
- `Security & Hardening/certificate-pki-lifecycle-guide.md` — certificate inventory and renewal policy
- `Databases/mysql-administration-guide.md` — MySQL hosts (see inconsistency note under Backup)
- `SysAdmin Procedures/monitoring-alerting-guide.md` — monitoring gap analysis and coverage matrix
- `SysAdmin Procedures/business-continuity-plan.md` — Scenario 5, loss of DMS / Dynamics GP
- `Hardware & Backup/backup-restore-testing-procedure.md` — Tier 1 restore testing for DMS
- `Active Directory/FileMaker_AD_External_Authentication_Troubleshooting_Runbook.txt` — `fm__dms__*` group auth
- `Security Procedures/ransomware-response-playbook.md` — erpdb isolation rules
- Upstream: Claris FileMaker Server Help (Admin Console, `fmsadmin` CLI, SSL certificates, Data API); Apache HTTP Server `mod_ssl` documentation

## Change log

| Date | Author | Change |
|---|---|---|
| 2026-10-04 | IT lead (it@example.com) | Initial draft consolidating DMS Web / erpdb Apache notes. UNVERIFIED — resolve [FILL IN]/[CONFIRM] markers on erpdb and port.example.com. |

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

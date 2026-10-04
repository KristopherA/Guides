# FileMaker Server Administration (erpdb.example.com / DMS)

> Status: initial draft, 2026-10-04. Contains [FILL IN] markers — see INDEX.md.

<!-- Tier 3 (full): Core + Extended sections. FileMaker Server hosts Tier 1
     data (member, grievance and arbitration records), so every section applies. -->

## Overview

FileMaker Server (FMS — branded "Claris FileMaker Server" from version 19 on)
hosts the FileMaker database files behind **DMS**, the organization's in-house membership and
labour-relations system. DMS holds member records, locals and employer data,
communications, grievances and arbitration files, events, OH&S, finance views
and the Education/Trust Fund modules — the DMS user manuals under
`Dynamics GP & DMS/DMS Manual/` describe the user-facing side. Staff reach DMS in
three ways: the FileMaker Pro client ("DMS 1.0" / "v1"), **DMS Web** at
`dms.example.com` (a separate web application that reads and writes the hosted files
through FMS's web services), and [FILL IN: the mobile apps referenced on the DMS
Web Global Settings screen — FileMaker Go, or a separate app using the Data API?].

The server is **erpdb.example.com**, a macOS host (shell prompt `fmadmin@erpdb`, FMS
installed under `/Library/FileMaker Server/`). The backup and restore testing
procedure ranks "DMS / erpdb.example.com" as **Tier 1 — critical**: losing it stops
member services and legally time-bound processes such as grievance and
arbitration deadlines.

This guide covers the server: architecture, access, hosted files, accounts and
external authentication, certificates, backups, restore, maintenance, upgrades,
logs, monitoring, troubleshooting, security and DR. It does **not** cover DMS
application development (layouts, scripts) or the DMS Web container internals,
other than where they touch the server.

> **Discrepancy to resolve.** `Databases/mysql-administration-guide.md`,
> `Hardware & Backup/backup-restore-testing-procedure.md` and
> `Networking Guide/ipam-vlan-topology-reference.md` describe erpdb.example.com as a
> "DMS / Dynamics GP database" reached over ODBC, as if it were a MySQL or SQL
> Server host. The 2022 Apache notes and the KB articles show it is the
> **FileMaker Server** host. [CONFIRM: erpdb.example.com runs FileMaker Server only,
> and the MySQL tables DMS uses (e.g. the `MySQL_SiteMessages` layout) live on
> another host and are reached from FMS via ESS/ODBC. Once confirmed, correct the
> erpdb rows in the three documents above.]

## Quick Facts

| Field            | Value |
|------------------|-------|
| Owner            | IT lead (it@example.com); DMS development: consultant — [FILL IN: full name, role and contact for the DMS developer] |
| Environment      | prod. [FILL IN: is there any test/dev FileMaker Server? Without one, upgrades and restore tests happen against production] |
| Location         | erpdb.example.com — [FILL IN: IP address, VLAN, physical Mac model or VM, rack location] |
| OS               | macOS — [FILL IN: exact version; check with `sw_vers`] |
| FMS version      | [FILL IN: run `fmsadmin -v` or check Admin Console > Dashboard]. Licensed in 2018 as FileMaker Server 16 (see Licensing below); upgraded since — the 2022 notes mention "an FMS upgrade a couple weeks back" |
| Access           | Admin Console `https://erpdb.example.com/admin-console` [VERIFY: path and port for the installed version]; SSH/console as `fmadmin`; `fmsadmin` CLI. Admin Console credentials in 1Password / IT vault |
| Dependencies     | Active Directory (auth2/auth3/auth4.example.com) for external authentication; DNS; TLS certificate for erpdb.example.com; [CONFIRM: MySQL host for ESS tables]; Retrospect for off-host backup copies; NTP |
| Dependents       | DMS Pro clients, DMS Web (`dms.example.com`, Docker on port.example.com), [FILL IN: mobile apps], Dynamics GP integration ("Dynamics access is enabled" toggle in DMS Web), AppDB2 user sync ("Update From AppDB2") |
| Backup tier      | Tier 1 — critical (`Hardware & Backup/backup-restore-testing-procedure.md`) |
| Last reviewed    | NEVER — drafted 2026-10-04 |

## How It Works

### Components

    FileMaker Pro (DMS v1) ---- TCP 5003 (TLS) ----+
    FileMaker Go [FILL IN]  ---- TCP 5003 (TLS) ----+
                                                    |
    Browser -> dms.example.com (DMS Web, Docker on       |
      port.example.com) -> HTTPS 443 --------------------+--> erpdb.example.com (macOS)
                                                    |     FileMaker Server
    Admin browser -> HTTPS /admin-console ----------+       Database Server  (hosts .fmp12)
                                                    |       Web Publishing Engine (Data API / XML / PHP / WebDirect)
    ODBC/JDBC clients -> TCP 2399 [CONFIRM: in use?]+       Bundled Apache (HTTPServer) on 80/443
                                                            Admin Server + Admin API
                                                            -> AD (external auth) -> auth2/auth3/auth4.example.com
                                                            -> ESS/ODBC -> [CONFIRM: MySQL host]

- **Database Server** opens and serves the hosted `.fmp12` files to clients on
  TCP 5003.
- **Web Publishing Engine** serves the Data API, XML/PHP custom web publishing
  and WebDirect, fronted by FMS's **own bundled Apache** in
  `/Library/FileMaker Server/HTTPServer/`. DMS Web depends on this path: per the
  KB article, when that Apache goes down DMS Web still loads (it is a separate
  Docker container) but **logins and all database access fail**, and the Admin
  Console is unreachable remotely. [CONFIRM: which API DMS Web uses — Data API,
  XML, or the PHP API.]
- **Admin Server** runs the Admin Console and the Admin API.

### Ports

| Port | Protocol | Used by | Exposure in this environment |
|------|----------|---------|-----------------|
| 5003 | TCP | FileMaker Pro / Go clients; server-to-server | [FILL IN: internal VLANs / VPN only? Confirm it is not open to the internet] |
| 443  | TCP (HTTPS) | Data API, XML/PHP, WebDirect, Admin Console (19+) | From port.example.com (DMS Web) and admin workstations. [FILL IN: anything else] |
| 80   | TCP (HTTP) | Redirect to 443 only | [CONFIRM: proposed — redirect only, never serve data on 80] |
| 2399 | TCP | ODBC/JDBC (xDBC) into FileMaker files | [CONFIRM: proposed — closed unless a named consumer exists] |
| 16000 | TCP (HTTPS) | Admin Console in FMS 17/18; internal admin traffic in 19+ | [VERIFY: on 19/2023/2024 the console is at 443 `/admin-console` and 16000 should be reachable from admin hosts only] |
| 16001–16004, 5013, 8998 etc. | TCP | Internal FMS process ports | Localhost only. [VERIFY: list for the installed version from the Claris "ports used" KB] |

### Where things live (macOS)

    /Library/FileMaker Server/Data/Databases/        # live hosted files — never copy these while open
    /Library/FileMaker Server/Data/Backups/          # default scheduled backup destination [CONFIRM: actual destination]
    /Library/FileMaker Server/Data/Progressive/      # progressive backup working copies [VERIFY: default path]
    /Library/FileMaker Server/Logs/                  # Event.log, Access.log, Stats.log, fmdapi.log, wpe logs
    /Library/FileMaker Server/HTTPServer/conf/       # bundled Apache config, server.key / server.pem
    /Library/FileMaker Server/HTTPServer/logs/       # bundled Apache logs
    /Library/FileMaker Server/CStore/                # FMS certificate store (imported custom cert) [VERIFY]
    /Library/FileMaker Server/Database Server/bin/   # fmsadmin and FMDeveloperTool [VERIFY]

### Licensing

`[internal: FileMaker licence download page]` is **not documentation** — it is a February 2018
FileMaker, Inc. Electronic Software Download page for the organization's volume licence:
FileMaker Server 16 (25 concurrent connections), FileMaker Pro 16 and Pro 16
Advanced, maintenance contract effective 2017-01-15 to 2020-03-31. It contains
**licence keys in plaintext**. Do not copy them into this document.
[CONFIRM: proposed — move the PDF into 1Password / IT vault and delete it from
this library; licence keys in a shared folder are a licence-compliance and
misuse risk.] Current licence: [FILL IN: Claris licence type (annual / FLT /
concurrent), connection count, expiry date, renewal contact. Check Admin
Console > Server > License.]

## Administration Access

### Admin Console

Browse to `https://erpdb.example.com/admin-console` from an admin workstation
[VERIFY: on FMS 17/18 it was `https://erpdb.example.com:16000`]. Credentials are in
1Password / IT vault under [FILL IN: vault item name]. Main areas:
Dashboard, Databases (open/close/pause, verify, client list), Backups
(schedules, progressive), Configuration (certificates, notifications, external
authentication, web publishing), Logs.

### fmsadmin CLI

Run on erpdb.example.com as `fmadmin`. `fmsadmin` prompts for the Admin Console user
and password when `-u`/`-p` are omitted. **Never use `-p <password>` on the
command line** — it lands in shell history and the process list.

    fmsadmin -v                                    # FMS version
    fmsadmin list files -s                         # hosted files, status, client count
    fmsadmin list clients -s                       # connected clients with IDs, user, file, IP
    fmsadmin list schedules -s                     # backup / script schedules with IDs and last run
    fmsadmin get serverconfig                      # cache size, max files, connection limits, etc.
    fmsadmin get serverprefs                       # [VERIFY: available on 19.x+; key names vary by version]
    fmsadmin status file "<file>.fmp12"            # state of one file

### Admin API (scripting / export)

Use when you need machine-readable output, e.g. exporting schedules before an
upgrade. Version path `v2` applies to FMS 19+ [VERIFY on the installed version].

    curl -s -X POST -u <admin-user> -H "Content-Type: application/json" https://erpdb.example.com/fmi/admin/api/v2/user/auth   # prompts for password; returns a bearer token
    curl -s -H "Authorization: Bearer <token>" https://erpdb.example.com/fmi/admin/api/v2/server/status                      # server status
    curl -s -H "Authorization: Bearer <token>" https://erpdb.example.com/fmi/admin/api/v2/databases                          # hosted files
    curl -s -H "Authorization: Bearer <token>" https://erpdb.example.com/fmi/admin/api/v2/schedules > fms-schedules.json     # schedule export
    curl -s -X DELETE -H "Authorization: Bearer <token>" https://erpdb.example.com/fmi/admin/api/v2/user/auth/<token>        # log out — tokens otherwise stay valid until timeout

## Hosted Files Inventory

Populate from `fmsadmin list files -s` and Admin Console > Databases. This table
answers "what breaks if this file won't open".

| File (`.fmp12`) | Purpose | Approx. size | Encrypted at rest | Opened at startup | Used by | Notes |
|---|---|---|---|---|---|---|
| [FILL IN: "DMS 1.0" main file name] | DMS main file — contains `MySQL_SiteMessages`, the Database Admin layout (Users / Logs / Defaults) | [FILL IN] | [FILL IN] | [FILL IN] | Pro clients, DMS Web | Site downtime messages are edited here (see Maintenance) |
| [FILL IN] | [FILL IN: data/separation-model files, if any] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |

[FILL IN: inter-file dependencies — which files reference which. Closing or
restoring one file of a multi-file solution in isolation can leave the others
pointing at an inconsistent copy.]

## Accounts, Privilege Sets and AD External Authentication

### Layers of permission

DMS access is controlled at **three** separate layers. Know which one you are
looking at before changing anything.

1. **AD group membership** — groups named `fm__dms__<group>` (e.g.
   `fm__dms__Systems`, `fm__dms__PersonnelV2`, `fm__dms__DepartmentManager`,
   `fm__dms__LabourRelations`), all under `CN=Users,DC=example,DC=com`.
2. **FileMaker accounts and privilege sets** — in each hosted file, File >
   Manage > Security. External Server accounts map an AD group to a privilege
   set; FileMaker evaluates them in **account-access priority order**, and the
   first active matching entry wins.
3. **DMS application permissions ("HasTasks")** — per-user JSON of module/task
   flags maintained in the DMS **Database Admin** layout (e.g. `DMS_Membership`
   → `view_entities`, `DMS_Communications` → `create_communications`), plus UI
   state that can be reset with "Reset UI Windows" / "Reset To Default".
   See `Dynamics GP & DMS/Reset DMS User UI.png`.

The library-root screenshot `[internal screenshot: DMS permission screen]` shows
`(DMS_LabourRelations) edit_arbitration_committee` and
`(DMSWeb_NumberedFiles) edit_arbitration_committee`. Those are in the
`(<module>) <task>` shape of layer 3, not a FileMaker privilege-set dialog.
[CONFIRM: this is the DMS / DMS Web task-permission list for the Arbitration
Committee (ARC) — if so, move it to `Dynamics GP & DMS/` and close the item in
`NEEDS_MANUAL_REVIEW.md`.]

### External authentication (summary)

FMS performs its own directory lookup — the DMS developer does not write the
LDAP query. Connection parameters (LDAPS 636, base DN, `servicesadmin` bind
account, UPN login) are in `Active Directory/ldap-connection-reference.md`.
Domain controllers seen by the runbook: auth2.example.com (10.60.1.221),
auth3.example.com (192.168.2.222, remote office), auth4.example.com (10.60.1.145).
[VERIFY: on macOS, FMS external authentication relies on the host's own
directory binding (Directory Utility / `dsconfigad`) rather than an LDAP
configuration inside FMS; FMS 19.1+ also supports OAuth / Entra ID. Record which
mechanism erpdb.example.com uses: `dsconfigad -show`.]

**Full troubleshooting procedure:**
`Active Directory/FileMaker_AD_External_Authentication_Troubleshooting_Runbook.txt`.
Its conclusion (2026-09-29): AD membership, DNS, network, LDAPS trust, bind, UPN
lookup and `memberOf` all **PASS**. A user who belongs to several `fm__dms__`
groups (e.g. sysadmin: `PersonnelV2` and `Systems`) receives the privilege
set of whichever matching External Server entry is **highest in FileMaker's
account-access priority**. Once direct LDAP succeeds, stop changing AD and work
in Manage Security.

Rules for this server:

- Order External Server entries **most specific / most privileged first**
  (e.g. `fm__dms__Systems` above `fm__dms__PersonnelV2`). [CONFIRM: proposed
  ordering rule — agree with the DMS developer.]
- Document the existing order (priority, group, method, privilege set,
  active/inactive) **before** reordering, as the runbook instructs. Record it in
  the table below.
- Keep at least one **local Full Access account** per file for break-glass
  use when AD is unreachable; credentials in 1Password / IT vault only.
- Keep `[Guest]` disabled. [FILL IN: confirm on every hosted file.]

| Priority | Account / AD group | Auth | Privilege set | Extended privileges | Active |
|---|---|---|---|---|---|
| [FILL IN] | [FILL IN: e.g. fm__dms__Systems] | External Server | [FILL IN] | [FILL IN: fmapp, fmrest, fmxml, fmphp, fmwebdirect as applicable] | [FILL IN] |
| [FILL IN] | [FILL IN: e.g. fm__dms__PersonnelV2] | External Server | [FILL IN] | [FILL IN] | [FILL IN] |
| [FILL IN] | [FILL IN: DMS Web service account] | FileMaker | [FILL IN] | [FILL IN: fmrest or fmxml/fmphp only] | [FILL IN] |
| [FILL IN] | [FILL IN: break-glass Full Access] | FileMaker | [Full Access] | [FILL IN] | [FILL IN] |

### Disconnecting a single user

Admin Console > Databases > Clients, or:

    fmsadmin list clients -s                                    # find the client ID
    fmsadmin disconnect client <ID> -m "Disconnecting for maintenance" -t 60 -y   # 60 s warning, no confirmation prompt

The DMS Database Admin layout also has a "Disconnect v1 User" button for DMS
Pro sessions.

## SSL / TLS Certificate

### What is in place

- FMS uses an imported certificate for **erpdb.example.com** for client (5003) and
  web (443) connections. Issuer and expiry: [FILL IN: run the check below and
  record in the inventory in `Security & Hardening/certificate-pki-lifecycle-guide.md`
  — erpdb.example.com has no row there yet].
- Files named `erpdb.example.com.pem`, `erpdb.example.com_key.pem` and
  `erpdb.example.com_intermediate.pem` exist in
  `/Library/FileMaker Server/HTTPServer/conf/` (2022 notes).
- [FILL IN: is dms.example.com (DMS Web) on the `*.example.com` wildcard or its own cert?
  It is served by the Docker host, not by FMS.]

    echo | openssl s_client -connect erpdb.example.com:443 -servername erpdb.example.com 2>/dev/null | openssl x509 -noout -subject -issuer -dates   # what 443 serves now
    sudo ls -l "/Library/FileMaker Server/CStore/"                                                                                     # certificate files FMS holds [VERIFY path]

### Known trap — bundled Apache key/cert mismatch after upgrades

From `Dynamics GP & DMS/DMSWeb Apache crash solution.txt` and the KB article
[internal KB: Apache down on DMS web]: FMS's bundled Apache periodically crashed
and would not restart because `server.key` / `server.pem` in
`/Library/FileMaker Server/HTTPServer/conf/` did not match. The fix was to keep
FMS's generated pair as `server.fm_created.*` and copy the erpdb.example.com key and
certificate into place. **Every FMS reinstall or upgrade recreates those two
files**, so the fix must be re-applied after each upgrade (see Upgrades). That
fix lets Apache relaunch cleanly; it does not stop the underlying crash.
[CONFIRM: whether this is still needed on the current version once the
certificate is imported properly with `fmsadmin certificate import` — a
correctly imported certificate should be what FMS writes into the Apache config.]

### Renewal / import procedure

Follow the generic seven-step flow in
`Security & Hardening/certificate-pki-lifecycle-guide.md` (establish what is
served, generate key/CSR, submit, verify pairing and chain, install, verify from
a client, record). The FMS-specific install step:

    fmsadmin certificate create "/CN=erpdb.example.com"                                       # optional: let FMS generate key + CSR [VERIFY: subject syntax for your version]
    fmsadmin certificate import /path/erpdb.example.com.crt --keyfile /path/erpdb.example.com.key --intermediateCA /path/intermediate.crt   # prompts for key passphrase if encrypted — do not pass --keyfilepass on the command line
    fmsadmin restart server                                                              # certificate is only loaded on restart — closes all files, schedule a window
    echo | openssl s_client -connect erpdb.example.com:443 -servername erpdb.example.com 2>/dev/null | openssl x509 -noout -dates   # confirm new expiry is served

Then verify from real clients: FileMaker Pro shows a closed padlock with no
certificate warning when opening the DMS file; DMS Web logins work; the Admin
Console loads without a browser warning. Re-check the bundled Apache files above.
[CONFIRM: proposed lead time — begin renewal 45 days before expiry, alert at 30.]

## Backups

### Design

FMS must back up its own files. **Never let Retrospect or any file-level tool
copy `/Library/FileMaker Server/Data/Databases/` while files are open** — the
copy will be inconsistent or damaged. Retrospect should copy the FMS **backup
destination** instead.

| Setting | Value | Reason |
|---|---|---|
| Hourly schedule | [CONFIRM: proposed — every hour 07:00–19:00 weekdays, keep 24] | Limits data loss during the working day to ≤ 1 h |
| Daily schedule | [CONFIRM: proposed — 23:00 nightly, keep 14] | Point-in-time copies across two weeks for "deleted it last Tuesday" restores |
| Weekly / monthly | [CONFIRM: proposed — weekly Sunday keep 8; monthly keep 12] | Longer retention for grievance/arbitration files whose errors are found late |
| Verify backup integrity | [CONFIRM: proposed — ON for daily and weekly schedules] | Catches a damaged file at backup time instead of at restore time |
| Clone | [CONFIRM: proposed — ON for weekly] | An empty clone is a clean structural copy if data files are ever damaged |
| Progressive backup | [CONFIRM: proposed — ON, 5-minute interval] | Near-continuous copy of changes; protects against server/disk failure, not against user deletions it faithfully copies |
| Backup destination | [FILL IN: path — default `/Library/FileMaker Server/Data/Backups/`; on a separate volume is better] | A backup on the same disk as the live files dies with that disk |
| Off-host copy | [FILL IN: Retrospect source/schedule that copies the backup folder; time must be after the nightly FMS backup finishes] | 3-2-1: off-host and offsite |
| Encryption-at-rest key | [FILL IN: if files are EAR-encrypted, where the key is held — vault item] | An encrypted backup is unusable without the key |
| Email notifications | [FILL IN: Admin Console > Notifications — SMTP relay and recipient] | Failed backups must alert someone (see Monitoring) |

Current state: [FILL IN: record the existing schedules from
`fmsadmin list schedules -s` here before changing anything].

### Commands

    fmsadmin list schedules -s                                   # IDs, type, last run, next run, status
    fmsadmin run schedule <ID>                                   # run a backup schedule now
    fmsadmin backup "<file>.fmp12"                               # ad-hoc backup to the default backup folder
    fmsadmin backup "<file>.fmp12" -d "filemac:/<Volume>/<path>/" -k 3   # to a specific folder, keep 3 [VERIFY: filemac path syntax]
    fmsadmin disable schedule <ID>                               # pause a schedule (e.g. during restore)
    fmsadmin enable schedule <ID>                                # re-enable it — do not forget
    ls -lt "/Library/FileMaker Server/Data/Backups/" | head      # newest backup sets first
    find "/Library/FileMaker Server/Data/Backups" -maxdepth 1 -type d -mmin -90   # anything written in the last 90 min? (empty = hourly backup stale)

Progressive backup is enabled and its interval set in Admin Console > Backups >
Progressive Backup. [VERIFY: whether it can be set via `fmsadmin set
serverconfig` on the installed version.]

## Restore Procedure

An untested restore is a rumour. Record every test in the restore-test log in
`Hardware & Backup/backup-restore-testing-procedure.md` and update the DMS row
of its "last tested" table.

### Restore a whole file (production)

1. Decide the restore point with the business owner — everything after it is
   lost unless re-entered. Note the time.
2. Announce downtime (see Maintenance > Planned downtime).
3. Stop new connections and close the file:

        fmsadmin send -m "DMS is being restored. Please save and log out."      # broadcast to connected clients
        fmsadmin close "<file>.fmp12" -m "DMS closing for restore" -t 300 -y    # 5-minute grace, then close
        fmsadmin list files -s                                                   # confirm status Closed
        fmsadmin disable schedule <ID>                                           # stop backups overwriting your evidence

4. Keep the damaged/current file — move, do not delete:

        sudo mkdir -p "/Library/FileMaker Server/Data/PreRestore-$(date +%F)"                             # holding folder
        sudo mv "/Library/FileMaker Server/Data/Databases/<file>.fmp12" "/Library/FileMaker Server/Data/PreRestore-$(date +%F)/"   # preserve current state

5. Copy the chosen backup in and fix ownership:

        sudo cp -p "/Library/FileMaker Server/Data/Backups/<set>/<path>/<file>.fmp12" "/Library/FileMaker Server/Data/Databases/"   # restore copy
        sudo chown fmserver:fmsadmin "/Library/FileMaker Server/Data/Databases/<file>.fmp12"   # FMS cannot open files it does not own [VERIFY owner/group]

6. Open, verify and re-enable:

        fmsadmin open "<file>.fmp12"                     # prompts for the EAR key if encrypted
        fmsadmin list files -s                           # status Normal
        fmsadmin enable schedule <ID>                    # backups back on
        fmsadmin run schedule <ID>                       # take a fresh backup of the restored state

7. Test as a user: log in with an AD account in FileMaker Pro and via DMS Web,
   check a recent record exists, and check record counts on [FILL IN: two
   stable tables, e.g. members and grievances] against the pre-restore figures.

Restoring from a **progressive** backup follows the same steps, using the
newest complete copy in the progressive folder. [VERIFY: progressive folder
layout for the installed version.] For a single deleted record or small set,
prefer opening a backup copy **locally in FileMaker Pro** and importing the
records into the live file over replacing the whole file.

### Restore test (quarterly) [CONFIRM: proposed cadence for Tier 1 depth-1 test]

Without a test FMS, test without touching production:

1. Copy the newest daily backup to an admin Mac (not into the live Databases
   folder).
2. Open it locally in FileMaker Pro with the break-glass account (EAR key from
   the vault if encrypted).
3. Record: file opens without a consistency warning; record counts on the
   chosen tables; newest modification timestamp; time taken end to end.
4. Delete the local copy — it contains member data
   (`Hardware & Backup/backup-restore-testing-procedure.md`, "Test restores
   contain live data").

Last tested: [FILL IN]. Result: [FILL IN]. Next due: [FILL IN].

## Maintenance

### Planned downtime (from the KB "How to Temp DMS Web and reboot Production")

1. Email the Staff Mail List, Regional Offices and News with the window.
2. Post the DMS Web maintenance message: log into DMS as Developer, open the
   `MySQL_SiteMessages` layout in the DMS 1.0 main file, update the usual
   downtime record's Message, and set DateStart/DateEnd (inclusive). Set the
   dates in the past to remove it afterwards.
3. "Temp" DMS Web (put it in maintenance mode) on the Docker host — as
   `orgadmin`, not root or sudo:

        ssh orgadmin@port.example.com                                              # Docker host for dms.example.com
        cd /mnt/data/orgdocker/dms.example.com                                     # DMS Web deployment folder
        ./mantisdeployment.php temp mounts/sites/dmsweb                       # maintenance mode on
        ./mantisdeployment.php untemp mounts/sites/dmsweb                     # maintenance mode off (after the work)

4. Limit connections so FileMaker Pro clients cannot reconnect. The KB uses
   **The Missing Admin** tool on erpdb to set Client Connections to 0
   [FILL IN: what The Missing Admin is and where it is installed]. CLI
   equivalent:

        fmsadmin get serverconfig                                             # note the current connection limit first
        fmsadmin set serverconfig <ConnectionsKey>=0                          # [VERIFY: exact key name on the installed version]

5. Close files — Admin Console > Databases (Activity) > Close All, with a
   time and message; or:

        fmsadmin close -m "DMS maintenance — please log out now" -t 300 -y    # all files, 5-minute grace
        fmsadmin list clients -s                                              # stragglers?
        fmsadmin disconnect client <ID> -t 0 -y                               # force-disconnect if needed

   Closing can take a long time; clients may need force-disconnecting.
6. Do the work. Then reopen, restore the connection limit, untemp DMS Web,
   remove the site message.

        fmsadmin open                                                          # open all files in the Databases folder
        fmsadmin list files -s                                                 # all Normal

### Consistency check

Run after any crash, power loss (see `Hardware & Backup/power-outage-shutdown-runbook.md`),
or a backup reporting a verification failure:

    fmsadmin verify "<file>.fmp12"                   # closes the file, checks consistency, reopens — users are disconnected [VERIFY on installed version]
    grep -i "consistency\|damaged\|verify" "/Library/FileMaker Server/Logs/Event.log" | tail -20   # result

Proposed routine: [CONFIRM: monthly verify of each file in a maintenance window,
plus "Verify backup integrity" on daily schedules].

### Recover — last resort only

FileMaker's **Recover** rebuilds a damaged file and may silently drop data or
structure. Claris's own guidance is that a recovered file should **not** go back
into production if any good backup exists.

1. Try the most recent backup that passes a consistency check first.
2. Only if none exists, run Recover **on a copy**, never the original:
   FileMaker Pro > File > Recover, or on the server
   `"/Library/FileMaker Server/Database Server/bin/FMDeveloperTool" --recover <copy>.fmp12 -t <outdir>` [VERIFY: tool path and options on the installed version].
3. Read the recovery log, export the recovered data, and import it into a clean
   clone of the last good backup, with the DMS developer. Document what was lost.

### Pause instead of close

    fmsadmin pause "<file>.fmp12"                    # suspends writes so the file folder can be copied safely
    fmsadmin resume "<file>.fmp12"                   # resume — keep pauses short; users hang while paused

## Upgrades

Version notes:

- FMS 16 (licensed 2018) → 17/18 (Admin Console on 16000, new Admin API v1) →
  19 (Claris branding, Admin Console moved to `/admin-console` on 443,
  Admin API v2) → 2023 (v20) → 2024 (v21) → [VERIFY: current release and
  in-place upgrade support from the installed version; older majors required a
  full uninstall/reinstall].
- Check Claris's system requirements for the macOS version on erpdb **and**
  the FileMaker Pro version on every DMS client before upgrading. [FILL IN:
  FileMaker Pro version deployed to staff.]

Procedure:

1. Read the release notes; check DMS Web/API compatibility with the developer.
2. Export state that a full reinstall wipes (the KB warns schedules may not
   survive a major-version upgrade):

        fmsadmin list schedules -s > ~/fms-pre-upgrade-schedules-$(date +%F).txt   # human-readable list
        fmsadmin get serverconfig > ~/fms-pre-upgrade-serverconfig-$(date +%F).txt  # server settings
        # plus the Admin API schedules export above (JSON)

   Also screenshot: external authentication settings, notification settings,
   web publishing settings, certificate details.
3. Planned downtime steps 1–5; run a fresh backup of every file and copy it
   off the server.
4. [FILL IN: snapshot / Time Machine / Retrospect image of the host before
   upgrade, if any.]
5. Stop the server and run the installer:

        fmsadmin stop server -t 300 -y               # closes files and stops the Database Server

6. After install: re-import the certificate if prompted, re-apply the bundled
   Apache fix (below) if still needed, recreate any missing schedules,
   re-enable progressive backup, confirm external authentication.

        cd "/Library/FileMaker Server/HTTPServer/conf"
        sudo mv server.key server.fm_created.key         # keep FMS's generated pair for reference
        sudo mv server.pem server.fm_created.pem
        sudo cp erpdb.example.com_key.pem server.key          # organization key into place
        sudo cp erpdb.example.com.pem server.pem              # organization cert into place
        sudo "/Library/FileMaker Server/HTTPServer/bin/httpdctl" start   # start bundled Apache

7. Verify: `fmsadmin -v`, `fmsadmin list files -s`, Pro login with an AD
   account, DMS Web login, a manual `fmsadmin run schedule <ID>`. Then untemp.
8. Log the change in `SysAdmin Procedures/Change_Management_Log.txt` and the
   Change log below.

## Logs

| Log | Path | What it tells you |
|---|---|---|
| Event.log | `/Library/FileMaker Server/Logs/Event.log` | Server start/stop, file open/close, backups and verification, schedule results, errors — first stop for everything |
| Access.log | `/Library/FileMaker Server/Logs/Access.log` | Client logins, logouts, failed authentication with account name and IP — first stop for "can't log in" |
| Stats.log | `/Library/FileMaker Server/Logs/Stats.log` | Server load: clients, I/O, cache hit %, elapsed time per call. Only written when usage statistics are enabled |
| ClientStats.log / TopCallStats.log | same folder | Per-client and slowest-call statistics; enable only while investigating |
| fmdapi.log / wpe logs | same folder [VERIFY names] | Data API and Web Publishing Engine — DMS Web errors |
| Bundled Apache | `/Library/FileMaker Server/HTTPServer/logs/` | Apache errors, including the key/cert mismatch on startup |

    tail -f "/Library/FileMaker Server/Logs/Event.log"                          # live events
    grep -i "error\|warning" "/Library/FileMaker Server/Logs/Event.log" | tail -50   # recent problems
    grep -i "<user>@example.com" "/Library/FileMaker Server/Logs/Access.log" | tail -20   # a user's recent logins/failures
    fmsadmin enable serverstats                                                  # start writing Stats.log
    fmsadmin disable serverstats                                                 # stop it
    fmsadmin enable clientstats                                                  # per-client stats while diagnosing; disable afterwards

Log size and retention are set in Admin Console > Logs. [CONFIRM: proposed —
Event and Access logs 40 MB each, rolled; forwarded to Graylog.]

## Monitoring & Alerting

Nothing monitors FMS today — consistent with the gap analysis in
`SysAdmin Procedures/monitoring-alerting-guide.md`. Proposed checks, in
priority order [CONFIRM: all thresholds]:

| Check | How | Alert when | To |
|---|---|---|---|
| Backup freshness | `find` on the backup folder (above) from cron/launchd, or FMS email notification on schedule failure | No new backup in 90 min during business hours, or any failed schedule | [FILL IN] |
| Client port | `nc -vz -w 3 erpdb.example.com 5003` | Fails twice in a row | [FILL IN] |
| Web / DMS Web path | `curl -s https://erpdb.example.com/fmi/data/vLatest/productInfo` (no auth needed) | Non-200 — this is the "Apache has gone down" case | [FILL IN] |
| Certificate expiry | `echo \| openssl s_client -connect erpdb.example.com:443 2>/dev/null \| openssl x509 -noout -enddate` | < 30 days | [FILL IN] |
| Disk space | `df -h "/Library/FileMaker Server/Data"` | > 80 % used on data or backup volume | [FILL IN] |
| Files open | `fmsadmin list files -s` | Any expected file not Normal | [FILL IN] |
| Failed logins | Access.log → Graylog | Spike per account/IP | [FILL IN] |

FMS's built-in email notifications (Admin Console > Notifications) need an SMTP
relay: [FILL IN: mail.example.com relay settings and recipient address]. Forwarding
FMS logs to Graylog needs a shipper on macOS [FILL IN: chosen method].

Normal baseline: [FILL IN: typical connected clients during the day, Stats.log
cache hit % (expect > 95 %), backup duration].

## Troubleshooting

### Symptom: DMS Web loads but nobody can log in; Admin Console unreachable remotely

- Likely cause: FMS's bundled Apache has stopped ([internal KB: Apache down on DMS web]).
- Check: `curl -sk -o /dev/null -w '%{http_code}\n' https://erpdb.example.com/fmi/data/vLatest/productInfo`   # 000 or 5xx = Apache down
- Check: `tail -50 "/Library/FileMaker Server/HTTPServer/logs/error_log"`   # look for key/cert mismatch [VERIFY file name]
- Fix: `cd "/Library/FileMaker Server/HTTPServer/bin" && sudo ./httpdctl start`   # expect "========== start ==========" then the httpd -k start line
- If it refuses to start citing the key, re-apply the server.key/server.pem fix (Upgrades step 6).

### Symptom: FileMaker Pro clients can't connect / host not listed

- Check: `fmsadmin list files -s`   # file closed or not hosted?
- Check: `nc -vz -w 3 erpdb.example.com 5003`   # from the client's network; fail = firewall/VPN/Database Server down
- Check: `fmsadmin list clients -s` and `fmsadmin get serverconfig`   # connection limit left at 0 after maintenance? licence limit reached?
- Fix: `fmsadmin open "<file>.fmp12"`, restore the connection limit, or `fmsadmin start server` if the Database Server is stopped. For network issues see `Networking Guide/firewall-change-procedure.md`.

### Symptom: file won't open, or opens with "damaged" / consistency errors

- Check: `grep -i "<file>" "/Library/FileMaker Server/Logs/Event.log" | tail -30`   # reason for refusal
- Check: `ls -l "/Library/FileMaker Server/Data/Databases/"`   # ownership must be FMS's user/group; wrong owner after a manual copy is common
- Check: `df -h "/Library/FileMaker Server/Data"`   # full disk causes write failures and damage
- Fix: fix ownership (`sudo chown fmserver:fmsadmin <file>` [VERIFY]), free disk, then `fmsadmin verify "<file>.fmp12"`. If verify fails: restore from backup (Restore Procedure). Recover only as last resort.

### Symptom: certificate warning in FileMaker Pro or browser

- Check: `echo | openssl s_client -connect erpdb.example.com:443 -servername erpdb.example.com 2>/dev/null | openssl x509 -noout -subject -issuer -dates`   # expired? wrong name? self-signed default?
- Check: Admin Console > Configuration > SSL Certificate   # is the custom cert still imported after an upgrade?
- Fix: re-import with `fmsadmin certificate import ... --intermediateCA ...` then `fmsadmin restart server`. Missing intermediate is the usual cause of "untrusted" on some clients only.

### Symptom: AD user can't log in, or gets the wrong privilege set

- Check: `grep -i "<user>@example.com" "/Library/FileMaker Server/Logs/Access.log" | tail`   # authentication failure vs. success
- Check: direct LDAP with the runbook's `ldapsearch` UPN + `memberOf` test   # proves AD side
- Check: `dsconfigad -show`   # macOS host still bound to example.com? [VERIFY: applies if FMS uses OS directory binding]
- Fix: if LDAP passes, the cause is in File > Manage > Security — External Server account priority, an inactive entry, or a missing group. Follow `Active Directory/FileMaker_AD_External_Authentication_Troubleshooting_Runbook.txt` steps 9–11. Users must sign in as `user@example.com` [CONFIRM: login format FMS expects].

### Symptom: DMS works but user sees wrong tabs/modules or stale layout

- Likely cause: DMS application permissions (HasTasks) or saved UI state, not FMS.
- Fix: DMS Database Admin > Users > the user > Current UI > "Reset UI Windows" / "Reset To Default", or "Update From HasTasks". See `Dynamics GP & DMS/Reset DMS User UI.png`.

### Symptom: backup schedule failed

- Check: `fmsadmin list schedules -s`   # last result
- Check: `grep -i backup "/Library/FileMaker Server/Logs/Event.log" | tail -20`   # reason: disk full, destination missing, verify failure
- Fix: correct the destination/space, then `fmsadmin run schedule <ID>`. A verification failure means the **live** file needs a consistency check now.

## Security

- **Exposure**: 5003 and 443 should be reachable only from organization networks / VPN
  and port.example.com. [FILL IN: current pfSense/firewall rules for erpdb.example.com.]
  Admin Console access restricted to admin hosts [CONFIRM: proposed — use the
  Admin Console IP restriction setting].
- **Auth**: AD external authentication via `fm__dms__` groups; local
  break-glass Full Access account per file; `[Guest]` disabled; Admin Console
  account distinct from any FileMaker account.
- **Extended privileges**: only the DMS Web service account's privilege set
  holds the web extended privilege it uses (fmrest / fmxml / fmphp). Remove
  `fmwebdirect`, `fmxdbc` from any set that does not need them [FILL IN: audit].
- **Transport**: "Use SSL for database connections" ON; custom certificate,
  not FMS's default self-signed one.
- **Encryption at rest**: [FILL IN: are the DMS files EAR-encrypted? If yes, the
  key lives in 1Password / IT vault and is needed for every restore.]
- **Secrets**: Admin Console, `fmadmin` macOS account, break-glass, EAR key,
  DMS Web service account, licence keys — 1Password / IT vault only. Never in
  this library. `[internal: FileMaker licence download page]` currently breaks this rule (licence
  keys).
- **Data sensitivity**: member, grievance and payroll-adjacent data. erpdb is
  called out in `Security Procedures/ransomware-response-playbook.md` — do not
  isolate it alone during an incident; get a second person.
- **Patching**: macOS and FMS updates per `SysAdmin Procedures/Patch_Management_Workflow.txt`.
  [CONFIRM: proposed — apply FMS security updates within 30 days.]

## Disaster Recovery

- Tier 1. [CONFIRM: proposed RPO 1 hour (hourly backups; minutes with
  progressive), RTO 8 business hours — agree with organization leadership and record in
  `SysAdmin Procedures/business-continuity-plan.md`.] The RTO is a guess until
  a depth-3 rebuild has been timed.
- **Rebuild from nothing**:
  1. Mac hardware meeting Claris requirements [FILL IN: spare/standby Mac?],
     macOS installed, hostname `erpdb`, static IP, DNS records unchanged.
  2. Install the same FMS version as recorded in Quick Facts (installer and
     licence from the vault / Claris account).
  3. Re-create the Admin Console account (vault), import the erpdb.example.com
     certificate, apply the bundled Apache fix if needed.
  4. Re-establish AD binding / external authentication and test with the
     runbook's representative accounts.
  5. Restore `.fmp12` files from the newest verified backup (Retrospect
     off-host copy if the host is lost), fix ownership, open.
  6. Recreate schedules from the exported list/JSON; enable progressive backup.
  7. Re-create ODBC/ESS DSNs [FILL IN: DSN names and target MySQL host].
  8. Confirm DMS Web on port.example.com reaches the new server; untemp.
- **Escalation if the owner is unavailable**: [FILL IN: second admin], then
  the consultant (DMS developer) [FILL IN: contact], then [FILL IN: Claris support /
  partner contract].

## Decisions & History (ADR-lite)

| Date | Decision / Change | Why / Source |
|---|---|---|
| 2018-02-08 | FileMaker Server 16 + Pro 16 volume licence (25 concurrent connections), maintenance to 2020-03-31 | `[internal: FileMaker licence download page]` |
| 2022-06-10 | Bundled Apache restarted with `httpdctl start` after outage | [internal KB: Apache down on DMS web] |
| 2022 (approx.) | Replaced FMS-generated `server.key`/`server.pem` with erpdb.example.com pair to stop Apache restart failures; must be redone after each FMS upgrade | `Dynamics GP & DMS/DMSWeb Apache crash solution.txt` |
| 2024-09-26 | Downtime procedure (temp DMS Web, limit connections, close files) published | KB "How to Temp DMS Web and reboot Production" |
| 2026-09-29 | AD external-auth investigation: AD/LDAPS all pass; focus moved to FileMaker account-access priority | AD runbook |
| [FILL IN] | Upgrades to FMS 17/18/19/2023/2024 | [FILL IN] |

## References

Internal (library-root relative):

- `Active Directory/FileMaker_AD_External_Authentication_Troubleshooting_Runbook.txt` — AD/privilege-set troubleshooting (authoritative)
- `Active Directory/ldap-connection-reference.md` — LDAPS parameters, `servicesadmin` bind account
- `Active Directory/AD-Admin-Security-Guide.md` — AD security, service-account hygiene
- `Security & Hardening/certificate-pki-lifecycle-guide.md` — renewal flow; add an erpdb.example.com row to its inventory
- `Hardware & Backup/backup-restore-testing-procedure.md` — Tier 1 cadence, restore-test log
- `Hardware & Backup/power-outage-shutdown-runbook.md` — close FMS files cleanly before power-down
- `SysAdmin Procedures/monitoring-alerting-guide.md` — monitoring gaps, Graylog
- `Dynamics GP & DMS/DMSWeb Apache crash solution.txt`
- `[internal KB: Apache down on DMS web]`
- `[internal KB: temp DMS web and reboot production]`
- `Dynamics GP & DMS/Reset DMS User UI.png`, `Dynamics GP & DMS/DMS Settings for Dynamics update.png`
- `Dynamics GP & DMS/DMS Manual/` — end-user DMS documentation
- `[internal screenshot: DMS permission screen]` (library root) — see Accounts section
- `[internal: FileMaker licence download page]` — 2018 licence download page (contains licence keys — move to vault)
- `Databases/mysql-administration-guide.md` — contains the erpdb discrepancy noted in Overview
- Online KB: [internal KB: Apache down on DMS web], [internal KB: temp DMS web and reboot production]

Upstream (Claris) [VERIFY: use the edition matching the installed version]:

- Claris FileMaker Server Help (includes `fmsadmin` command reference): https://help.claris.com/en/server-help/
- Claris FileMaker Server Installation and Configuration Guide: https://help.claris.com/en/server-installation-configuration-guide/
- Claris FileMaker Admin API Guide: https://help.claris.com/en/admin-api-guide/
- Claris Support knowledge base (ports used by FileMaker Server, recovery guidance): https://support.claris.com/

## Change log

| Date | Author | Change |
|---|---|---|
| 2026-10-04 | IT lead (it@example.com) | Initial draft from library sources — UNVERIFIED |

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

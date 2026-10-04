# Microsoft Dynamics GP 2018 — Administrator's Guide
### Setup, Maintenance, Troubleshooting & Finance Department Support Reference

Reference guide for the GP system administrator. Covers architecture, install/setup, routine maintenance, backend SQL techniques for advanced troubleshooting, and a plain-language GUI troubleshooting section written for Finance Department staff. Techniques in sections 5–6 are drawn from real fixes used in this environment — several are recognizable as Microsoft's own published guidance (KB878449); others are direct-SQL patterns pulled from a third-party Canadian Payroll module (tables prefixed `CPY`). Treat every direct-SQL example as a *pattern to adapt*, not a script to run verbatim against production.

---

## 1. Architecture Overview

Dynamics GP 2018 is a Dexterity application (the `Dynamics.exe`/`GP.exe` runtime plus `.dic` dictionaries) sitting on a SQL Server backend. Key pieces an admin needs to know:

| Component | What it is |
|---|---|
| `DYNAMICS` database | The system database — company list (`SY01500`), users, security, activity tracking |
| Company databases (e.g. `TWO`, or a named DB) | One database per company, holds that company's transactional/master tables |
| `DYNGRP` | The SQL Server role every GP user's login is a member of — table/proc permissions are granted to this role, not to individual users |
| Dexterity dictionaries (`.dic`) | Compiled application code — core (`Dynamics.dic`) plus one per module/ISV product installed |
| `DEX_LOCK` / `DEX_SESSION` (in `tempdb`) | Dexterity's row-locking mechanism — rebuilt at SQL Server startup by a stored procedure marked as a startup proc |
| GP Utilities (`GPUtilities.exe` / launched from the installer) | Runs database creation/upgrade scripts, loads sample company (Fabrikam), sets up ODBC |
| `Dex.ini` | Per-workstation client config (default company, dictionary paths, logging flags) — lives in the GP program folder on each client |

Third-party/ISV modules (like the Canadian Payroll module referenced throughout this guide) add their own tables into each company database with their own prefix (`CPY` in this environment) and their own Dexterity dictionary — they follow the same DYNGRP permission model as core GP.

---

## 2. Installation & Environment Setup

GUI: GP installer → GP Utilities (runs automatically on first launch post-install).

- SQL Server backend must be a supported version/edition for GP2018 (check current Microsoft Dynamics GP compatibility documentation for the exact SQL Server build list before provisioning — this changes with GP service packs and SQL Server CUs).
- Run **GP Utilities** on the SQL Server (or a machine with a direct connection) first — it creates/upgrades `DYNAMICS` and prompts to load the Fabrikam sample company; do this before installing any client workstations.
- Each workstation needs the GP client installed and pointed at the SQL Server via ODBC/OLE DB — the install wizard configures this; verify with **Microsoft Dynamics GP Utilities** or the ODBC Data Source Administrator (`odbcad32`) if a workstation can't connect.
- `Dex.ini` on a problem workstation is the first thing to check for a client that won't launch or points at the wrong server — it's a plain-text file in the GP install folder.
- Registration keys (product + module keys) are entered via **Microsoft Dynamics GP > Maintenance > System > Registration** — needed after adding modules or renewing.

---

## 3. Company & User Setup

GUI: **Microsoft Dynamics GP > Tools > Setup > System**.

- **Company setup**: GP Utilities creates the company shell; company-level setup (address, fiscal periods, posting settings) is done in **Setup > Company > Company**.
- **User setup**: **Setup > System > User** creates the SQL Server login and GP user record together, and assigns company access.
- **Security**: GP2018 uses role-based security (**Setup > System > Security Roles** / **Security Tasks**) — a user gets one or more Roles, each Role is built from Tasks, each Task grants access to specific windows/reports/SmartLists. This is the GUI-side security model; the SQL-side counterpart is the `DYNGRP` role that all GP logins belong to, which is why "permission denied" errors are sometimes a GP Security Task problem and sometimes a raw SQL grant problem (see section 5.2 for the latter).
- **Account Framework / Segments**: set up once at the start of an implementation via **Setup > Company > Account Format** — essentially immutable afterward, so get this right before go-live.

---

## 4. Routine Maintenance

| Task | GUI | Frequency |
|---|---|---|
| Full SQL backup of DYNAMICS + all company DBs | SQL Server Management Studio / maintenance plan | Nightly minimum |
| Transaction log backups | SSMS / maintenance plan | Per your recovery-point requirement |
| Check Links | **Microsoft Dynamics GP Utilities > Maintenance > Check Links** | After any data-integrity concern, before/after major imports |
| Reconcile (module-level) | e.g. **Financial > Utilities > Reconcile**, **Sales > Utilities > Reconcile** | Monthly, or when balances look wrong |
| SQL index/statistics maintenance | SQL Agent job (`sp_updatestats`, index rebuild) | Weekly/scheduled |
| Clear stuck user sessions | **Microsoft Dynamics GP > Maintenance > Clear Data > Activity** (admin only, all users must be out) | As needed, after a crash left a phantom session |
| Rebuild `DEX_LOCK`/`DEX_SESSION` | SQL script (section 5.3) | Only after a `tempdb` rebuild or when Dexterity throws SQL requirement errors |
| Year-end / period-end close | Module-specific close routines (GL, Payroll, etc.) | Per fiscal calendar |

**Always** run Check Links and take a fresh backup before and after any bulk data change — whether that change is done through the GUI, the Macro Recorder, or direct SQL.

---

## 5. Backend SQL Techniques for Admins

These are real patterns used to resolve specific GP issues. **Every one of these bypasses Dexterity's business logic and runs directly against the database — take a verified backup first, run it in a Test company or restored copy before touching production, and do it during a maintenance window with all users out of GP.** If you have an active support contract with Microsoft or your GP partner, their guidance takes precedence over generic examples like these.

### 5.1 Capturing SQL Server logins before a server move (KB878449)

When migrating GP to a new SQL Server, you need every GP-related SQL login re-created with the *same* password and SID, or every user's GP login breaks. Microsoft's published script (`sp_help_revlogin` / the SQL 2005+ variant `seeMigrateSQLLogins`) reads the login table and prints out ready-to-run `CREATE LOGIN`/`sp_addlogin` statements with the password hash embedded, so nothing has to be manually reset.

```sql
-- Run on the OLD server, capture the PRINT output to a .sql file
USE master
-- ... creates sp_hexadecimal + sp_help_revlogin (or seeMigrateSQLLogins on SQL 2005+) ...
-- then, at the bottom:
EXEC sp_help_revlogin          -- or EXEC seeMigrateSQLLogins on SQL 2005/2008
-- Save the PRINTed output, run it against the NEW server to recreate logins identically
```

Save the query *results/output* (not the script itself) to a text file, and run that output on the new server after GP databases are restored there.

### 5.2 Re-granting DYNGRP permissions after a restore or schema change (KB878449)

The single most common "you do not have permission" error after restoring a company database onto a new server, or after an ISV module adds new tables, is that the new/restored objects were never granted to `DYNGRP`. The fix loops through every table, view, and stored procedure in the database and grants `DYNGRP` full CRUD/execute rights:

```sql
declare @cStatement varchar(255)
declare G_cursor CURSOR for
  select 'grant select,update,insert,delete on [' + convert(varchar(64),name) + '] to DYNGRP'
  from sysobjects where (type = 'U' or type = 'V') and uid = 1
-- loop and EXEC each generated statement, then repeat for type = 'P' with "grant execute ... to DYNGRP"
```

Run this against **each company database** (and `DYNAMICS`) any time you restore a database to a new server, or right after installing/upgrading a module that adds tables.

### 5.3 Rebuilding Dexterity's SQL requirements (`DEX_LOCK` / `DEX_SESSION`)

Dexterity (the runtime GP is built on) needs two tables in `tempdb` — `DEX_LOCK` and `DEX_SESSION` — to manage record locking. Because `tempdb` is rebuilt from scratch every time SQL Server restarts, these tables are recreated by a stored procedure (`smDEX_Build_Locks`) marked to auto-run at SQL Server startup. If that startup procedure was lost (e.g., after a `tempdb` rebuild, an in-place SQL Server upgrade, or a botched restore of `master`), GP clients fail to connect with a "SQL requirements not met" style error.

```sql
use tempdb
-- drop DEX_LOCK / DEX_SESSION if they exist
use master
-- create procedure smDEX_Build_Locks (creates DEX_LOCK, DEX_SESSION, indexes, grants to public)
sp_procoption 'smDEX_Build_Locks','startup','true'   -- <- this is the critical step people forget
smDEX_Build_Locks                                     -- run it once immediately too
```

If GP suddenly throws Dexterity SQL requirement errors after any SQL Server-level maintenance, this is the first thing to check and re-run.

### 5.4 Standing up a Test/Sandbox company by copying production

The standard way to get a safe Test company is: restore a copy of the production company database under a new name, add it as a company in GP Utilities/`SY01500`, then run a cleanup script that rewrites every `CompanyID`/`InterID`-style column in the copied database to match the *new* company's ID (otherwise cross-references and the company key mismatch and things silently misbehave):

```sql
-- Finds every column named COMPANYID / CMPANYID / INTERID / DB_NAME / DBNAME
-- across ALL tables in the current database, and updates it to the correct
-- value for THIS database as registered in DYNAMICS.dbo.SY01500
select case
  when upper(a.COLUMN_NAME) in ('COMPANYID','CMPANYID')
    then 'update '+a.TABLE_NAME+' set '+a.COLUMN_NAME+' = '+cast(b.CMPANYID as char(2))
  else 'update '+a.TABLE_NAME+' set '+a.COLUMN_NAME+' = '''+db_name()+''''
end
from INFORMATION_SCHEMA.COLUMNS a, DYNAMICS.dbo.SY01500 b
where upper(a.COLUMN_NAME) in ('COMPANYID','CMPANYID','INTERID','DB_NAME','DBNAME')
  and b.INTERID = db_name()
-- ...then EXEC each generated statement, logging which tables were touched
```

Run this **only** against the freshly copied/renamed database, never production, and re-grant DYNGRP (5.2) on it afterward since a raw database copy/restore carries the source server's permissions, not the target's.

### 5.5 Bulk rate & master-table maintenance (payroll/tax example pattern)

Some periodic changes — annual tax table updates, WCB/premium rate changes, pay rate adjustments across many employees — have no bulk-edit screen in the GUI, only one-record-at-a-time entry. The pattern used here: `SELECT` first to confirm exactly which rows match, then `UPDATE` using the *same* WHERE clause, then `SELECT` again to confirm the result:

```sql
-- 1. Confirm what you're about to change
SELECT PIncomeCode, PDescription, PRate FROM CPY10060 WHERE PRate = '43.93000'

-- 2. Make the change against the master table (defaults for new employees)
UPDATE CPY10060 SET PRate = '46.12000' WHERE PRate = '43.93000'

-- 3. Repeat against the employee-assigned table so existing employees pick it up too
UPDATE CPY10140 SET PRate = '46.12000' WHERE PRate = '43.93000'

-- 4. Confirm
SELECT PIncomeCode, PDescription, PRate FROM CPY10140 WHERE PRate = '46.12000'
```

The same select-verify → update → select-verify pattern was used for basic personal tax exemption amounts (`CPY10105`), WCB percent/maximum (`CPY10070`), and employer address fields (`CPY10010`) — always filter as tightly as the data allows (exact old value, or old value **and** employee/income code) so the WHERE clause can't accidentally touch rows you didn't intend.

### 5.6 Surgical duplicate-row cleanup with `DEX_ROW_ID`

Every Dexterity table has a `DEX_ROW_ID` — an internal, unique-per-row identity value. When a data problem is "this employee has a duplicate paycode assignment causing a double calculation," filtering on employee ID + income code alone can match more than one row. Adding `DEX_ROW_ID` to the WHERE clause targets exactly one physical row:

```sql
SELECT * FROM CPY10140 WHERE PEmployeeID = 'EMP001' AND PIncomeCode = '903070' AND DEX_ROW_ID = '28110'
DELETE FROM CPY10140 WHERE PEmployeeID = 'EMP001' AND PIncomeCode = '903070' AND DEX_ROW_ID = '28110'
```

Always `SELECT` with the exact same WHERE clause immediately before a `DELETE` to confirm it returns exactly one row.

### 5.7 Tracing a transaction end-to-end for reconciliation

For "why doesn't this cash receipt match what's in the GL" style questions, join across the relevant module tables rather than checking each screen separately. The pattern used here traces a Payables payment (`PM30200`/`PM30300`/`PM30600`) through Cash Management (`CM20200`) to its GL distribution (`GL00100`/`GL00105`), filtered to one batch and one account segment:

```sql
SELECT CM20200.CMTrxNum, CM20200.TRXAMNT, PM30200.BACHNUMB, PM30300.APTODCNM, GL00100.ACTDESCR
FROM CM20200
  INNER JOIN PM30300 ON CM20200.SRCDOCNUM = PM30300.VCHRNMBR
  INNER JOIN PM30200 ON PM30300.APTVCHNM = PM30200.VCHRNMBR
  INNER JOIN PM30600 ON PM30200.VCHRNMBR = PM30600.VCHRNMBR
  INNER JOIN GL00100 ON PM30600.DSTINDX = GL00100.ACTINDX
WHERE GL00100.ACTNUMBR_3 = '002' AND PM30200.BACHNUMB = '002-05-30-08'
ORDER BY CM20200.CMTrxNum, PM30300.APTODCNM
```

This kind of join is read-only and safe to run any time — it's the reporting-side counterpart to the update patterns above, useful for building a one-off reconciliation report when SmartList Builder isn't already set up for the question being asked.

---

## 6. Macro Recorder — the Safer Bulk-Edit Alternative

Before reaching for direct SQL updates (section 5.5), consider GP's built-in **Macro Recorder** — it drives the actual GP windows, so every change still goes through Dexterity's normal validation, triggers, and any ISV business logic, unlike a raw SQL `UPDATE`.

GUI: **Microsoft Dynamics GP > Tools > Macro > Record** (and **Play**).

How it's used for bulk edits in practice:

1. Open the window you need (e.g., an employee setup window), turn on **Record**, and manually make the change for one record — the recorder writes out a plain-text `.mac` file describing every field and button interaction.
2. Stop recording, open the `.mac` file in a text editor. A single edit looks like this:

```
CheckActiveWin dictionary 'Canadian Payroll'  form 'P_CPY_SETP_Employee' window 'P_CPY_SETP_Employee'
NewActiveWin dictionary 'Canadian Payroll'  form 'P_CPY_SETP_Employee' window 'P_CPY_SETP_Employee'
  TypeTo field 'P_Employee_ID' , 'EMP002'
  MoveTo field 'P_WCB_Code'
  TypeTo field 'P_WCB_Code' , 'WCB001'
  MoveTo field 'P_Employer_Number'
NewActiveWin dictionary 'Canadian Payroll'  form DiaLog window DiaLog
  ClickHit field OK
NewActiveWin dictionary 'Canadian Payroll'  form 'P_CPY_SETP_Employee' window 'P_CPY_SETP_Employee'
  MoveTo field 'Save Button'
  ClickHit field 'Save Button'
```

3. Copy that block repeatedly, changing only the `Employee_ID` (and the value being set) for each record — a few hundred employees becomes a few hundred nearly-identical blocks in one text file.
4. **Tools > Macro > Play** the finished file against GP with all the target records ready — it runs unattended, keystroke-for-keystroke, exactly as if a person had done it.

When to prefer this over direct SQL: any time the field has GP-side validation, triggers a workflow, or updates related tables the way the UI does but a raw `UPDATE` wouldn't (the WCB code change above is a good example — the macro comment `# WCB flag automatically turned on.` shows GP setting a related flag as a side effect of the UI edit, which a direct SQL `UPDATE` to just the WCB code column would have missed entirely).

When direct SQL is still the right tool: true master-table/rate corrections with no meaningful per-record business logic (section 5.5), or cleanup of already-corrupted data the UI won't let you touch (section 5.6).

---

## 7. Common User-Level Problems — GUI Fixes for Finance Staff

Plain-language, no code — hand this section directly to Finance Department staff. Each item says what to try first, and when to stop and call IT/the GP administrator instead.

**Can't log in / password rejected**
Confirm Caps Lock and that you're selecting the right company at the login screen. If your password was recently reset company-wide or you've been locked out after failed attempts, contact the GP administrator — passwords are reset at the SQL Server level, not something you can self-serve.

**"This record is currently being edited by another user" / a window won't let you in**
Someone else has that exact record open, or a previous session didn't close cleanly. Wait a few minutes and try again; if it persists after everyone who might have it open has confirmed they've closed GP, contact the administrator — they can clear the stuck session (section 4) without you losing work, but only an admin should do this while confirming no one is actually still using it.

**Screen looks frozen, grayed out, or a button won't respond**
Try clicking elsewhere on the window first — GP sometimes has a message or a Lookup window hidden behind the main one. If truly frozen, do not force-close and reopen a new session on top of it — close it fully first (see "safely closing GP" below), then reopen. If it won't close, contact IT before force-ending the process, since an ungraceful exit is what causes the "another user editing" lock issue for the next person.

**A batch won't post / says it's out of balance**
Check the batch's total against the sum of the individual transactions in it — a single mis-keyed amount is the usual cause. Use the batch's Edit List/Report before posting to see every line. If the numbers genuinely balance and it still won't post, stop and contact the GP administrator rather than re-entering the batch — reposting can sometimes create a duplicate.

**A report or SmartList is missing rows you expect to see**
Check the date range and any filters/restrictions at the top of the SmartList or report options window first — this is the cause the large majority of the time. If the filters look right and data is still missing, escalate to IT.

**Printing goes to the wrong printer, or nothing prints**
Check **Report Destination** — in most report/posting windows there's a "Printer/Screen/File" destination option, and it may be set to something other than what you expect. Reset it to Printer and pick the correct one from the dropdown.

**A menu item, window, or report you used to have access to is gone**
This is almost always a Security Role change, not a bug — someone adjusted what your role can see. Contact the GP administrator with the exact window/report name; they can check and adjust your assigned Security Tasks (section 3) if the access was removed in error.

**Lookup window doesn't show a record you know exists**
Check the Lookup window's own filter/restrict-by field near the top — it's easy to have it scoped to "Active only" or a specific range from a previous search. Clear the filter and search again before assuming the record is missing.

**GP feels sluggish**
Close windows/browsers you're not actively using inside GP rather than leaving dozens open — each open window is a small resource drain and, more importantly, can hold record locks that slow other users down too. Log all the way out at end of day rather than leaving GP open and locked overnight.

**"You do not have permission" on something you should be able to do**
Two different causes look identical to a user: a Security Task issue (self-service fix by the admin, see section 3) or a backend SQL permission gap after a restore/upgrade (section 5.2, admin-only). Either way this is an IT ticket, not something to troubleshoot yourself — note the exact window and action you were attempting when you report it.

**Safely closing GP at end of day**
Use File > Exit (or the window's own close button back out to the main menu) rather than closing GP from the Windows taskbar/Task Manager. This lets GP release any record locks cleanly and is the single biggest thing end users can do to prevent the "locked by another user" issue for colleagues the next morning.

---

## 8. Escalation Guide

| Situation | Finance staff can try | Escalate to GP admin when |
|---|---|---|
| Login/password issue | Verify Caps Lock, correct company | Password reset needed, account locked |
| Stuck/locked record | Wait, confirm no one else has it open, reopen GP cleanly | Still locked after confirming no active user has it |
| Batch out of balance | Review Edit List against source documents | Balances but still won't post |
| Missing report/SmartList data | Check date range/filters | Filters correct, data still missing |
| Printing wrong destination | Check Report Destination setting | Destination correct, still fails |
| Missing menu/window access | — | Always an admin (Security Role) ticket |
| "No permission" error | — | Always an admin ticket — note exact window/action |
| GP frozen/unresponsive | Try clicking elsewhere, wait briefly | Won't close cleanly — get IT before force-ending |
| Any request to "fix data directly in SQL" | — | Always the GP admin, with a fresh backup taken first |

---

## 9. Canadian Payroll — Year-End, Tax Updates & Payroll-Specific Troubleshooting

Issues specific to running a Canadian Payroll module inside GP, beyond the general fixes in sections 1–8.

### 9.1 Year-end close & T4/RL-1 sequencing

Close the payroll year **only** after the last pay run of the calendar year is fully processed and posted, and after every correction/adjustment for that year has been entered — the close locks in the wage and tax data that T4/RL-1 slip generation reads from.

- Sequence: confirm all pay runs for the year are posted → run the module Reconcile utility → take a full backup → run the year-end close → generate a T4/RL-1 preview → fix any box-mapping issues found in the preview *before* filing → if a mistake is only found after slips have already gone out, that requires an amended T4/T4A filing with CRA, not just a GP reprint.
- GP will generally let you run the year-end close early even if it shouldn't be run yet — the tool not blocking you isn't the same as it being safe. Closing before the true final pay run of the year, or before a stray manual/void cheque from earlier in the year is confirmed posted, is the single most common year-end support call.
- Take the backup immediately before the close specifically — restoring from a backup is the only way back if the close ran too early.

### 9.2 Verifying an annual tax table update actually applied

CRA (and Revenu Québec where applicable) publish new CPP/EI/QPP/QPPIP rates, tax brackets, and basic personal amounts each year; your Canadian Payroll vendor ships a tax update that has to be applied before the first pay run of the new year. Don't just trust that the update installer finished cleanly — verify:

- Basic personal amounts changed for each jurisdiction in use (this is exactly what the `CPY10105` pattern in section 5.5 checks and, if needed, corrects).
- CPP/EI maximum insurable earnings and rates reflect the new year in the relevant setup window.
- Run one test pay per jurisdiction in the Test company (section 5.4) and manually check the calculated tax against the current year's published payroll deductions figures before running the first real pay of the year.

If an update was missed or applied late, the select-verify → update → select-verify pattern from 5.5 is the same technique used to hand-correct the affected table after the fact.

### 9.3 Direct deposit / EFT file troubleshooting

- The most common cause of a bounced deposit is a transposed routing/transit or account number in Employee Maintenance — verify it against an actual voided cheque or direct deposit form, don't just re-key from memory.
- Pre-note new employees (a $0 test transaction) before sending their first real deposit, if your bank/process supports it — it catches a bad account number before real money moves.
- If an entire EFT batch is rejected, check the file's header fields (company ID, creation date, sequence number) first — banks commonly reject the whole batch over one malformed header value even when every individual employee record inside it is fine.
- If one employee's deposit fails but the rest of the batch clears, that employee typically needs to be paid manually (cheque) for the current cycle while their banking info is corrected and re-verified — don't just resubmit the same bad record into the next EFT run.

### 9.4 Paycode & deduction sequencing / calculation order

Paycodes and deductions calculate in a defined order — gross paycodes first, then anything "based on" gross or based on another specific paycode/deduction. If a deduction is set up "based on" the wrong paycode, or something is sequenced before the paycode it should depend on, the paycheck comes out wrong with **no error message** — it just silently calculates against the wrong number.

- This is the same class of problem the duplicate-paycode cleanup in section 5.6 was fixing: a duplicate assignment doesn't just double-count that one paycode, it can shift the calculation order for everything sequenced after it.
- When a paycheck's net doesn't match manual math, check the paycode's "Based On" setup first, then check for a duplicate assignment (5.6), before assuming the tax tables or rates themselves are wrong.
- After any bulk paycode change — a rate update, a new paycode rollout — run at least one test pay per affected employee group in the Test company and manually verify gross-to-net before running it against real employees.

---

## 10. Managing SQL Server & Restoring GP from Backup

The SQL Server backend needs its own baseline configuration and backup discipline independent of GP itself — GP just happens to be an unusually backup-sensitive application, since a bad restore leaves you with permission errors (5.2), Dexterity errors (5.3), or a company list that doesn't match reality.

### 10.1 SQL Server configuration baseline for GP

GUI: SQL Server Management Studio (SSMS) → right-click server/database → Properties.

- **Collation**: GP requires a specific, case-insensitive server collation (set at SQL Server install time). A database restored or attached from a server with a different collation causes join/comparison failures inside GP — check collation before attaching an unfamiliar backup, not after.
- **Recovery model**: set `DYNAMICS` and every company database to **Full** recovery, not Simple — Full is what makes transaction-log backups (and point-in-time restore) possible. Simple recovery means your only recovery points are full/differential backups, with everything since the last one unrecoverable.
- **tempdb**: pre-size the data and log files rather than relying on autogrowth, and put them on your fastest storage — `DEX_LOCK`/`DEX_SESSION` (section 5.3) live here and every GP session touches them constantly.
- **Autogrowth**: set fixed-size growth increments (e.g., 512MB) instead of percentage-based growth on `DYNAMICS` and company databases — percentage growth on a large database causes long, session-blocking growth events during heavy posting.
- **Max server memory**: cap it in Server Properties > Memory so SQL Server doesn't starve the OS — especially important if SQL Server shares the box with anything else.

```sql
-- quick health checks
SELECT name, recovery_model_desc, collation_name FROM sys.databases   -- recovery model + collation per DB
EXEC sp_who2                                                            -- active connections, useful before a restore
DBCC SQLPERF(LOGSPACE)                                                   -- log file usage per DB
```

### 10.2 Backup strategy

GUI: SSMS → right-click database → **Tasks > Back Up**, or a SQL Server Agent maintenance plan for scheduling.

```sql
BACKUP DATABASE DYNAMICS TO DISK = 'D:\SQLBackups\DYNAMICS_full.bak' WITH INIT, COMPRESSION
BACKUP DATABASE TWO TO DISK = 'D:\SQLBackups\TWO_full.bak' WITH INIT, COMPRESSION          -- repeat per company DB
BACKUP LOG TWO TO DISK = 'D:\SQLBackups\TWO_log.trn' WITH INIT                              -- if using Full recovery
BACKUP DATABASE master TO DISK = 'D:\SQLBackups\master.bak'                                  -- needed to recover logins in a full DR
BACKUP DATABASE msdb TO DISK = 'D:\SQLBackups\msdb.bak'                                        -- needed to recover SQL Agent jobs
RESTORE VERIFYONLY FROM DISK = 'D:\SQLBackups\TWO_full.bak'                                      -- confirm the backup file isn't corrupt
```

Recommended cadence: full backup nightly for `DYNAMICS` and every company database, transaction log backups every 15–60 minutes if you need point-in-time recovery, and periodic *actual* test-restores to a scratch instance — a backup you've never restored is unverified. Back up `master` and `msdb` too; they're what let you recover server-level logins and SQL Agent jobs (including the `smDEX_Build_Locks` startup procedure, section 5.3) in a full disaster-recovery scenario. Always take a fresh backup immediately before running any of the direct-SQL techniques in section 5 or the year-end close in 9.1.

### 10.3 Restoring Dynamics GP from backup

Three different scenarios, same core discipline:

- **Full disaster recovery** (lost server/instance): restore `master` and `msdb` first (may require rebuilding the instance first, then restoring these over it), then `DYNAMICS`, then every company database.
- **Single company restore** (one database corrupted, others fine): restore just that one company database — leave `DYNAMICS` and other companies alone.
- **Point-in-time restore** (need to undo a bad import or bad posting from earlier today): full backup + log backups WITH `STOPAT`, restoring into a copy first if you're not certain of the exact time.

```sql
-- Force everyone out before restoring over a live database
ALTER DATABASE TWO SET SINGLE_USER WITH ROLLBACK IMMEDIATE

-- Standard restore
RESTORE DATABASE TWO FROM DISK = 'D:\SQLBackups\TWO_full.bak' WITH REPLACE, RECOVERY

-- Point-in-time restore (undo everything after a specific bad transaction)
RESTORE DATABASE TWO FROM DISK = 'D:\SQLBackups\TWO_full.bak' WITH REPLACE, NORECOVERY
RESTORE LOG TWO FROM DISK = 'D:\SQLBackups\TWO_log.trn' WITH STOPAT = '2026-07-01 09:14:00', RECOVERY

ALTER DATABASE TWO SET MULTI_USER
```

**After any restore, before letting users back into GP:**

1. **Fix orphaned logins.** A restored database's users don't automatically match the SIDs of logins on the server they landed on — this is the classic "login failed for user" error immediately after a restore, even though the login clearly exists.
   ```sql
   ALTER USER [someuser] WITH LOGIN = [someuser]     -- re-links the DB user to the server login by name (modern syntax)
   -- or, to find every orphan at once:
   EXEC sp_change_users_login 'Report'                -- lists orphaned users in the current database
   ```
2. **Re-run the DYNGRP grant script** (section 5.2) against the restored database — a restore carries the source server's permission mappings, not the target's.
3. **Rebuild `DEX_LOCK`/`DEX_SESSION`** (section 5.3) if the restore involved rebuilding the SQL instance itself, not just the user databases.
4. **Run Check Links, then the module Reconcile** (section 4) on the restored company.
5. **Verify `DYNAMICS.dbo.SY01500`** still lists the company against the correct database name — if a company database was restored under a different physical name than it had before, `SY01500` needs to point at the new name.
6. **Spot-check before declaring it done**: log in as a test user, open a recent transaction and a recent report, and confirm the numbers match what was expected at the restore point.

Common restore failures, in order of how often they actually happen: forgetting step 1 (orphaned logins) and getting a flood of login-failed errors; forgetting step 2 (DYNGRP) and getting a flood of permission-denied errors; restoring `DYNAMICS` and a company database from backups taken at different times, so the company list and the actual data disagree with each other.

---

## 11. Correcting Posted Transaction Mistakes

Section 7 is about catching mistakes *before* posting; section 10.3 is the nuclear, whole-database rollback option. This section is the middle ground almost everyone actually needs: something already posted was wrong, and GP has a module-native Void/Correct/Reverse tool built for exactly that, so you don't need a restore for a single bad transaction.

General rule: GP does not let you edit a posted transaction directly. Every fix below works by creating a reversing entry (a void, a correction, or an unapply/reapply) — the original transaction stays in history, and a new offsetting entry brings the books back to correct. That's by design (audit trail) and it means "undoing" always leaves a visible trace, which is normal and expected, not a sign something went wrong.

### 11.1 General Ledger

- A posted GL journal entry can't be edited, but GP will build a reversing entry for you: look the transaction up in **Transactions > Financial > General** (use the transaction lookup, not a new entry), and use the **Correct** option on a posted transaction — GP generates the offsetting reversal automatically rather than making you key the reversal by hand.
- If the correction needs to land in a specific fiscal period (common at month/quarter boundaries), check which period the reversal actually posts to before assuming it landed where you meant it to.

### 11.2 Payables Management

- **Void a posted payment** (check or EFT): **Transactions > Purchasing > Void Historical Transactions – Payables**, select the payment, Void. This reverses the GL distribution and reopens the associated voucher so it can be paid correctly.
- **Void a posted invoice/credit memo** the same way if it hasn't been paid yet; if it has already been paid, you generally need to void the payment first, then the invoice.
- Gotcha: voiding a payment that posted into a now-closed period posts the void into the *current* open period instead — the expense/cash hit and its reversal end up in different periods, which is normal but worth flagging to Finance so it doesn't look like an unexplained variance later.

### 11.3 Receivables Management

- **Misapplied cash receipt** (the receipt itself was correct, it just got applied to the wrong invoice): don't void the whole receipt — use **Transactions > Sales > Apply Sales Documents** to unapply it, then reapply it to the correct invoice. This is the single most common Receivables fix and doesn't touch the underlying receipt record at all.
- **Void a posted Receivables transaction** entirely (wrong customer, duplicate entry, etc.): **Transactions > Sales > Void Historical Transactions – Receivables**.

### 11.4 Payroll (including Canadian Payroll)

- **Void a posted payroll check**: the module's Void Payroll Checks window reverses the GL posting and the associated tax/deduction accumulators for that employee.
- Two things a payroll void does **not** do, both worth knowing before promising a quick fix: it does not retroactively correct a T4/RL-1 slip that's already been issued (that needs an amended filing with CRA — ties to section 9.1), and it does not reverse a direct-deposit/EFT file that's already been transmitted to the bank (that has to be resolved with the bank directly — ties to section 9.3). The GP void only fixes GP's own books; anything that already left GP (a filed slip, a transmitted EFT file) needs its own separate correction outside the system.

### 11.5 Bank / Cash Management

- **Void a posted bank transaction** (a deposit or a non-AP/AR cheque entered directly in Cash Management): use the module's Void Bank Transactions / Bank Transaction Entry void function.
- If the transaction was already matched to a bank statement in Bank Reconciliation before the mistake was caught, voiding it can throw off a reconciliation that's already been completed — check whether that statement is still open before voiding, and if it's closed, loop in whoever owns bank rec before proceeding.

### 11.6 When the module-level undo isn't enough

Some mistakes genuinely can't be fixed with a Void/Correct window: a batch posted into the wrong company entirely, an import that created thousands of bad records, a bulk SQL/macro edit (sections 5–6) that went wrong at scale. For those:

- Weigh a full point-in-time restore (10.3) against how much *other* legitimate work happened in the same window — a restore rolls back everything in that database to that moment, not just the mistake.
- A middle path that avoids losing other users' work: restore the affected company into a **scratch/Test company copy** at the needed point in time (section 5.4), pull just the correct historical values or documents out of that copy, and manually re-key or re-import just what's needed back into live production. Slower than a straight restore, but it only touches what actually needs fixing.

**Quick decision guide:** caught before posting → fix the batch directly (section 7). Caught after posting, and it's one identifiable transaction → the module Void/Correct/Reverse tool above (11.1–11.5). Caught after posting, and it affects many transactions or can't be cleanly isolated → point-in-time restore or scratch-copy extraction (10.3, 11.6).

---

## Quick Reference Index

| Need | Where |
|---|---|
| Create/upgrade DYNAMICS or load Fabrikam | GP Utilities |
| Add a company | GP Utilities → then Company setup in GP |
| Add a user / set access | Setup > System > User |
| Assign what a user can see | Setup > System > Security Roles / Security Tasks |
| Fix data integrity after an import or crash | Maintenance > Check Links, then module Reconcile |
| Fix "you do not have permission" after a restore | DYNGRP grant script (5.2) against the affected database |
| Fix "SQL requirements not met" | Rebuild `smDEX_Build_Locks` startup proc (5.3) |
| Migrate SQL Server without breaking GP logins | `sp_help_revlogin`/`seeMigrateSQLLogins` capture (5.1) |
| Stand up a Test company | Restore copy + CompanyID rewrite script (5.4) |
| Bulk rate/table change with no GUI bulk-edit screen | Select-verify → Update → select-verify SQL pattern (5.5), or Macro Recorder (6) if the field has business logic |
| Clean up a confirmed duplicate row | Target by `DEX_ROW_ID` (5.6), select-verify before delete |
| Trace a transaction across modules for reconciliation | Read-only join across module tables (5.7) |
| Year-end close order/sequencing | Final pay posted → Reconcile → backup → close (9.1) |
| Verify the annual tax update actually applied | Check basic personal amounts + CPP/EI rates, test pay per jurisdiction (9.2) |
| EFT/direct deposit batch or employee rejected | Verify bank/transit/account numbers, check file header, pre-note (9.3) |
| Paycheck calculates wrong with no error shown | Check paycode "Based On" order, then duplicate assignments (5.6, 9.4) |
| Set up SQL Server correctly for GP | Collation, Full recovery model, tempdb sizing, fixed autogrowth (10.1) |
| Back up DYNAMICS + company DBs | Full nightly + log backups if Full recovery, verify with `RESTORE VERIFYONLY` (10.2) |
| Restore GP after a crash/corruption | Restore DB → fix orphaned logins → DYNGRP grant (5.2) → Check Links (10.3) |
| Undo a bad import/posting from earlier today | Point-in-time restore with `RESTORE LOG ... WITH STOPAT` (10.3) |
| "Login failed" right after a restore | Orphaned login — `ALTER USER ... WITH LOGIN` (10.3) |
| Undo a posted GL entry | Look it up in General transaction entry, use Correct to auto-reverse (11.1) |
| Undo a posted AP payment or invoice | Void Historical Transactions – Payables (11.2) |
| Cash receipt applied to the wrong invoice | Unapply/reapply in Apply Sales Documents, don't void the receipt (11.3) |
| Undo a posted payroll check | Void Payroll Checks — does NOT fix a filed T4 or a sent EFT file (11.4) |
| Undo a posted bank deposit/cheque | Void Bank Transactions — check if already bank-reconciled first (11.5) |
| Mistake too big/messy for a module void | Scratch-company extraction or point-in-time restore (11.6, 10.3) |

---

*Administrator reference — every direct-SQL example in section 5 modifies production data outside of Dexterity's validation layer. Back up first, test against a copy, and run during a maintenance window with users out of GP. When in doubt, the Macro Recorder (section 6) or your GP partner/Microsoft support is the safer path.*

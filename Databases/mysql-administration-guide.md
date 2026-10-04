# MySQL Administration

> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

## Overview

MySQL is one of the organization's core platforms. It backs the DMS / Dynamics GP integration,
the AppDB1 and AppDB2 application databases, and a primary/replica pair used for
reporting and resilience. This guide is the single reference for installing,
securing, replicating, backing up and troubleshooting MySQL in this environment. It supersedes
the 2018-era wiki set (see "Superseded documents") and is written against a
current **MySQL 8.x on Ubuntu 24.04 LTS** baseline.

The old wikis were written for MySQL 5.7 on Ubuntu 16.04/18.04. A great deal of
their procedure is still correct. Some of it is wrong in ways that fail loudly
(renamed commands) and some in ways that fail quietly (deprecated syntax that is
accepted but ignored). Where a command or directive changed, this guide says so in
a **Changed since the 2018 docs** callout rather than silently rewriting, so that
anyone working on a host that has not yet been upgraded can still follow along.

> **Security note, read before using the old files.** Several of the superseded
> documents contain plaintext passwords — root passwords, the replication account
> password, and a backup account password. Those credentials must be treated as
> compromised: they have sat in a shared documentation folder for years. Rotate
> anything still in use, and do not copy any credential out of those files into
> this one. Secrets belong in 1Password, never in this library.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Environment      | prod (plus [FILL IN: any staging/lab instances])                       |
| Location         | See host inventory below. Hosts run on [FILL IN: Proxmox VMs, physical, or a mix per host] |
| Access           | SSH as `orgadmin`, then `mysql` locally. Remote MySQL on 3306 (TLS required). [FILL IN: whether SSH/3306 are restricted to a management VLAN or VPN] |
| Dependencies     | TLS certificates (wildcard cert + CA in `/etc/mysql`), DNS, NTP, Retrospect client for filesystem backup, network path between primary and replica |
| Dependents       | Dynamics GP / DMS via ODBC, AppDB1, AppDB2, [FILL IN: other applications and reports] |
| Last reviewed    | 2026-09-11                                                             |

## Host and Database Inventory

Fill this in per host. Until it is filled in, nobody can answer "what breaks if
this server is down", which is the question that matters during an incident.

| Host | Role | MySQL version | OS | Databases | Consumers | Replication | Backup method | Notes |
|---|---|---|---|---|---|---|---|---|
| db01.example.com | Replication primary (source) | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Source for db02.example.com | [FILL IN] | [FILL IN] |
| db02.example.com | Replication replica | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | Replica of db01.example.com | [FILL IN] | [FILL IN] |
| erpdb.example.com | DMS / Dynamics GP database | [FILL IN] | [FILL IN] | [FILL IN] | Dynamics GP, DMS via ODBC | [FILL IN] | [FILL IN] | [FILL IN] |
| appdb1.example.com | Application database | [FILL IN] | [FILL IN] | `appdb1` | [FILL IN] | [FILL IN] | mysqldump via cron | Original 2018 build host |
| appdb2.example.com | Application database | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | System76 Meerkat, desktop OS with VNC |
| prod01.example.com | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] |

[FILL IN: any MySQL instance running in Docker — see the Docker section — and any
instance not listed above.]

## How It Works

Each host runs a standalone `mysqld` on port 3306 with TLS mandatory
(`require_secure_transport = ON`), using a wildcard certificate and CA bundle
stored in `/etc/mysql`. Clients that cannot present or accept TLS are refused
outright — this is deliberate and has bitten new integrations before, so it is the
first thing to check when a new client "can't connect for no reason".

db01.example.com and db02.example.com form an asynchronous primary/replica pair. The primary
writes binary logs; the replica's IO thread copies those logs into local relay
logs; the replica's SQL thread applies them. Asynchronous means the replica is
allowed to be behind, and that an outage of the primary can lose the tail of the
log. The replica is not a backup — it faithfully replicates a `DROP TABLE` too.

    application / Dynamics GP
      -> ODBC connector (MySQL Connector/ODBC 8.x)
      -> mysqld :3306 (TLS enforced)
      -> InnoDB datadir /var/lib/mysql
      -> binary log  ---(replication)--->  db02.example.com relay log -> replica datadir
      -> nightly mysqldump -> /home/orgadmin/...MySQLBackUps -> Retrospect -> offsite

Config lives in `/etc/mysql/` — the packaged layout uses `/etc/mysql/my.cnf` which
includes `/etc/mysql/conf.d/` and `/etc/mysql/mysql.conf.d/`. Data lives in
`/var/lib/mysql`. The error log is `/var/log/mysql/error.log`.

> **Changed since the 2018 docs — where to put config.** The old wikis show edits
> made directly in `/etc/mysql/my.cnf` (and in one case an ambiguous
> `sudo nano mysql.cnf`). On Ubuntu 24.04 with MySQL 8, put site-specific settings
> in a dedicated file such as `/etc/mysql/mysql.conf.d/99-org.cnf`. Package
> upgrades can overwrite the distributed files; a separate include file survives
> them and makes it obvious what the organization changed and what came from the package.

---

## Setup / Installation

### Base Linux

The 2018 base-build wiki documents System76 Jackal hardware, `ifconfig`,
`/etc/network/interfaces` and the System76 driver PPA. Most of that no longer
applies. Current baseline:

    sudo apt update && sudo apt -y dist-upgrade      # patch before building anything
    sudo apt install -y openssh-server               # remote access
    sudo dpkg-reconfigure tzdata                     # timezone
    timedatectl                                      # confirm NTP sync is active
    sudo hostnamectl set-hostname db01.example.com        # FQDN as hostname
    hostname -f                                      # verify

> **Changed since the 2018 docs — networking.** `/etc/network/interfaces` and
> `ifconfig` are gone. Ubuntu 18.04 introduced netplan and the "Ubuntu 18 wiki"
> already records this; 24.04 uses netplan too, with config under
> `/etc/netplan/*.yaml` and `sudo netplan try` before `sudo netplan apply`. Use
> `ip a` and `ip r` instead of `ifconfig` and `route`. YAML does not accept tabs.

> **Changed since the 2018 docs — hardware drivers.** The System76 PPA
> (`ppa:system76-dev/stable`) was needed for the original Jackal server hardware.
> [VERIFY: was valid for MySQL 5.7 / Ubuntu 18, confirm on current version —
> whether any MySQL host is still on System76 hardware needing that PPA, or
> whether they are all Proxmox guests now.]

Host hardening (SSH, firewall, accounts) is not repeated here. See
`Security & Hardening/fresh-box-hardening-cheatsheet.txt` and
`Security & Hardening/sshd hardening conf.txt`.

[VERIFY: was valid for MySQL 5.7 / Ubuntu 18, confirm on current version — the
original appdb1 build used an eCryptfs-encrypted home directory for `orgadmin`,
which is where backups were written. eCryptfs home encryption is deprecated and
was a recurring source of "the backup ran but the files are not there" confusion
when a cron job ran while the home directory was unmounted. Confirm whether any
host still does this; if so, move backups off the encrypted home.]

### Install MySQL

    sudo apt install -y mysql-server                 # MySQL 8.x from the Ubuntu archive
    systemctl status mysql --no-pager                # confirm running
    mysql --version                                  # record the exact version

> **Changed since the 2018 docs — the MySQL APT repository.** The Ubuntu 18 wiki
> installs Oracle's `mysql-apt-config` .deb to get a newer MySQL. On 24.04 the
> distribution package is already MySQL 8.x, so the extra repository is usually
> unnecessary. Add it only if a specific 8.x point release is required, and be
> aware of the warning from the old doc that still holds: once the MySQL APT repo
> is enabled you can no longer install MySQL packages from the Ubuntu archive.

> **Changed since the 2018 docs — the root password.** The old procedure runs
> `mysqladmin -uroot -p password yourNewPassword` and hunts for a temporary
> password in `/var/log/mysqld.log`. On Ubuntu's MySQL 8 package there is no
> generated temporary password; the `root` account is created with
> `auth_socket` authentication, so `sudo mysql` logs you in as root with no
> password at all. Setting a password is a deliberate change:
>
>     sudo mysql                                                     # auth_socket login, no password
>     ALTER USER 'root'@'localhost' IDENTIFIED WITH caching_sha2_password BY '<from 1Password>';
>
> Consider leaving root on `auth_socket` and creating named admin accounts
> instead — it is a better posture and it is what the package default assumes.
> Note that `mysqladmin -u root -p version` from the old docs still works as a
> quick liveness test.

### Secure the installation

    sudo mysql_secure_installation                   # removes test db, anonymous users, remote root

This step is unchanged and still worth running. MySQL 8 adds a validate_password
component prompt; accept it and record the chosen policy level here:
[FILL IN: password validation policy level in use].

Service control:

    sudo systemctl status mysql                      # state
    sudo systemctl restart mysql                     # full restart
    sudo systemctl stop mysql

> **Changed since the 2018 docs — service control.** `service mysql start|stop`
> and `/etc/init.d/mysqld` from the old wikis still work through compatibility
> shims but should not be used in new procedures. Use `systemctl`. Note also the
> unit is `mysql`, not `mysqld`, on Ubuntu.

---

## Configuration

### TLS

Take the server certificate, private key and CA/root certificate and place them in
`/etc/mysql` owned `root:mysql` with mode `0640` (key) / `0644` (cert). The old
docs are inconsistent about this — one shows `0640 root:mysql`, another shows
world-readable `644 root:root`, and the AppDB2 wiki shows `755 mysql:mysql`. The
correct answer is the restrictive one: **the private key must not be world
readable, and MySQL will refuse to start or silently skip TLS if permissions are
too open.**

    sudo chown root:mysql /etc/mysql/wildcardkey.pem
    sudo chmod 640 /etc/mysql/wildcardkey.pem        # key: group-readable by mysql only
    sudo chmod 644 /etc/mysql/wildcardcert.pem       # cert and CA may be world-readable

Settings, in `/etc/mysql/mysql.conf.d/99-org.cnf`:

    [mysqld]
    require_secure_transport = ON                    # refuse all non-TLS client connections
    ssl-ca   = /etc/mysql/thawteroot.pem             # CA / root bundle
    ssl-cert = /etc/mysql/wildcardcert.pem           # server certificate
    ssl-key  = /etc/mysql/wildcardkey.pem            # private key

Verify:

    SHOW VARIABLES LIKE '%ssl%';                     # paths populated, no blanks
    SHOW SESSION STATUS LIKE 'Ssl_version';          # what your own session negotiated
    SHOW GLOBAL VARIABLES LIKE 'tls_version';        # which TLS versions are offered
    SHOW STATUS LIKE 'Ssl_cipher';                   # non-empty means the session is encrypted

> **Changed since the 2018 docs — TLS variables and defaults.** `have_ssl` and
> `have_openssl`, shown in every old wiki's expected output, are deprecated in
> MySQL 8.0 and removed in 8.4. Do not use them as your check; use
> `SHOW STATUS LIKE 'Ssl_cipher'` on a live session instead. MySQL 8 also
> auto-generates self-signed certificates at first start if none are configured,
> which means "TLS is on" is no longer evidence that *your* certificate is in use —
> always confirm the configured paths. Finally, MySQL 8.0.28+ defaults
> `tls_version` to TLSv1.2 and TLSv1.3 only; TLSv1 and TLSv1.1 are gone, which
> can break very old ODBC clients.

> **Note on the old file contents.** `_ARCHIVE/superseded-2018-mysql/MySQL wikis/6 - MySQL Replication Set up.txt` (archived 2026-09-11) contains a
> transcription error worth knowing about: it lists `ssl-cert=` twice, once
> pointing at the key file, with no `ssl-key=` line at all. Do not copy that block.
> Several files also disagree about whether the cert is `.pem` or `.crt`. Use the
> actual filenames on the host.

Certificate rotation is a separate, newer procedure and is not restated here.
See **`Databases/mysql certificate rotation.txt`** — the important part is that
MySQL 8 supports `ALTER INSTANCE RELOAD TLS;` for a zero-downtime swap, so a cert
rotation no longer requires a restart. That capability did not exist in the
workflow the 2018 docs assumed.

### Generating a key and CSR

Still valid, from `_ARCHIVE/superseded-2018-mysql/SSL and MYSQL.txt` (archived 2026-09-11):

    openssl req -new -newkey rsa:2048 -nodes -keyout server.key -out server.csr   # key + CSR
    openssl req -text -noout -in server.csr                                        # inspect before submitting

[VERIFY: was valid for MySQL 5.7 / Ubuntu 18, confirm on current version — the old
docs reference a Thawte root (`thawteroot.pem`). Confirm the current CA and
whether the wildcard certificate is still the one MySQL uses.] RSA 2048 remains
acceptable; [CONFIRM: proposed default — use 3072-bit RSA or P-256 ECDSA for new
certificates.] Note that the CSR method shown in the old file provides no way to
add Subject Alternative Names, and modern clients validate SAN, not Common Name —
so a CSR built exactly as the old doc shows will produce a certificate that
current clients reject.

---

## ODBC Connectors (Dynamics GP / DMS)

Dynamics GP and DMS reach MySQL through MySQL Connector/ODBC. This is the part of
the stack most likely to be affected by a MySQL upgrade.

    sudo apt install -y unixodbc odbcinst              # ODBC driver manager

> **Changed since the 2018 docs — ODBC packages.** The old wikis install
> `libodbc1` and `odbcinst1debian2`. Those names are obsolete on 24.04; use
> `unixodbc` and `odbcinst`. The old wikis also install Connector/ODBC 5.3 and
> 8.0.12 from hand-downloaded tarballs into
> `/usr/lib/x86_64-linux-gnu/odbc/libmyodbc5w.so`. Use a current 8.x connector —
> the 8.x connector is required to talk to a MySQL 8 server using the default
> `caching_sha2_password` authentication plugin. A 5.x connector will fail to
> authenticate against a default MySQL 8 account, and the error message does not
> make the cause obvious.

Install the connector (tarball method, unchanged in shape):

    tar -xvf mysql-connector-odbc-<version>-linux-glibc<ver>-x86-64bit.tar.gz
    sudo cp <dir>/lib/libmyodbc8* /usr/lib/x86_64-linux-gnu/odbc/           # driver libraries
    sudo <dir>/bin/myodbc-installer -d -a -n "MySQL" \
      -t "DRIVER=/usr/lib/x86_64-linux-gnu/odbc/libmyodbc8w.so;"           # register the driver

> Note: `2 - ... ODBC` / `_ARCHIVE/superseded-2018-mysql/MySQL wikis/5 - Configure MySQL ODBC.txt` (archived 2026-09-11) contains two typos that
> will waste your afternoon — `tar -xcf` (should be `-xvf`) and a driver path
> missing a slash (`/usr/lib/x86_64-linux-gnuodbc/`). Corrected above.

Create and check a DSN:

    sudo myodbc-installer -s -a -c2 -n "<dsnname>" \
      -t "DRIVER=MySQL;SERVER=<host>;DATABASE=<db>;UID=<user>;"             # create system DSN
    cat /etc/odbcinst.ini                                                   # registered drivers
    cat /etc/odbc.ini                                                       # configured DSNs
    isql -v <dsnname> <user> <password>                                     # test the DSN end to end

**Do not put passwords in `/etc/odbc.ini`.** The old wiki's worked example writes
`PWD=123456` into a world-readable file. That was an upstream blog's example, not
a local decision, but it was copied into our docs and should not be repeated.
[FILL IN: how DSN credentials are actually supplied in this environment for the GP/DMS
connection — prompted, stored in the application, or in a restricted file.]

Because `require_secure_transport = ON`, the DSN must also be told to use TLS:

    ...;SSLCA=/path/to/ca.pem;SSLMODE=REQUIRED;                             # TLS options on the DSN

[FILL IN: the exact DSN names, target hosts and databases used by Dynamics GP and
DMS, and where the Windows-side DSNs are configured.]

Related: `Dynamics GP & DMS/` and
`Databases/MySQL Set up/Official Docs/connector-odbc-en.pdf`.

---

## Replication

### Concepts

Replication in this environment is asynchronous: the primary commits without waiting for the
replica. The replica can lag, and an ungraceful loss of the primary can lose the
tail of the binary log. Design around that.

Binary log format matters. The 2018 setup wiki configured `binlog_format =
STATEMENT` and warned about non-deterministic SQL producing different results on
each server. The organization now runs **ROW**-based replication (confirmed by the 2026
replication patching procedure), which removes that class of problem.

> **Changed since the 2018 docs — binlog format default.** MySQL 8.0 defaults to
> `binlog_format = ROW`; 5.7 defaulted to `MIXED` and the organization explicitly set
> `STATEMENT`. If you are reading the old wiki's config block, do not re-apply
> `binlog_format = STATEMENT`. Note also that `binlog_format` is deprecated as of
> MySQL 8.0.34 — row-based is becoming the only supported option.

### Primary (source) configuration

In `/etc/mysql/mysql.conf.d/99-org.cnf` on db01.example.com:

    [mysqld]
    server_id     = 1                                # must be unique across the topology
    log_bin       = /var/log/mysql/mysql-bin         # enable binary logging
    binlog_format = ROW                              # row-based (current standard)
    # plus the TLS block from the Configuration section

[FILL IN: the actual `server_id` values in use, and the real `log_bin` prefix —
the 2018 docs used `appdb1-bin` on a host later renamed. Do not assume.]

Restart MySQL, then confirm binary logging is on:

    SHOW BINARY LOGS;                                # lists log files and sizes
    SHOW VARIABLES LIKE 'datadir';                   # usually /var/lib/mysql
    SHOW BINARY LOG STATUS;                          # current file and position (MySQL 8.4+)
    SHOW MASTER STATUS;                              # same thing, MySQL 8.0 and earlier

> **Changed since the 2018 docs — replication command names.** MySQL 8.0.22
> deprecated the master/slave vocabulary and 8.4 removed the old commands
> entirely. Both forms are shown throughout this guide because the organization has hosts on
> both sides of that line. The mapping:
>
> | Old (5.7, 8.0)                   | Current (8.0.22+, required in 8.4+)        |
> |----------------------------------|--------------------------------------------|
> | `CHANGE MASTER TO`               | `CHANGE REPLICATION SOURCE TO`             |
> | `MASTER_HOST=` / `MASTER_USER=`  | `SOURCE_HOST=` / `SOURCE_USER=`            |
> | `MASTER_LOG_FILE=` / `_POS=`     | `SOURCE_LOG_FILE=` / `SOURCE_LOG_POS=`     |
> | `MASTER_SSL=1`                   | `SOURCE_SSL=1`                             |
> | `START SLAVE` / `STOP SLAVE`     | `START REPLICA` / `STOP REPLICA`           |
> | `SHOW SLAVE STATUS`              | `SHOW REPLICA STATUS`                      |
> | `SHOW MASTER STATUS`             | `SHOW BINARY LOG STATUS`                   |
> | `SQL_SLAVE_SKIP_COUNTER`         | `SQL_REPLICA_SKIP_COUNTER`                 |
> | `RESET SLAVE`                    | `RESET REPLICA`                            |
> | Field `Slave_IO_Running`         | `Replica_IO_Running`                       |
> | Field `Slave_SQL_Running`        | `Replica_SQL_Running`                      |
> | Field `Seconds_Behind_Master`    | `Seconds_Behind_Source`                    |
>
> Scripts and monitoring that grep for `Slave_IO_Running` will silently return
> nothing after an upgrade to 8.4. That is a quiet failure worth hunting for
> before the upgrade, not after.

### Replica configuration

In `/etc/mysql/mysql.conf.d/99-org.cnf` on db02.example.com:

    [mysqld]
    server_id = 2                                    # unique, not necessarily sequential
    # plus the TLS block

### Replication account

Create on the primary, restricted to the replica's address:

    CREATE USER 'replicant'@'<replica-ip>' IDENTIFIED BY '<from 1Password>' REQUIRE SSL;
    GRANT REPLICATION SLAVE ON *.* TO 'replicant'@'<replica-ip>';
    FLUSH PRIVILEGES;

> **Changed since the 2018 docs — password hashing.** The old wiki uses
> `SELECT password('...')` and `IDENTIFIED BY PASSWORD '<hash>'`. **The
> `PASSWORD()` function and the `IDENTIFIED BY PASSWORD` syntax were removed in
> MySQL 8.0** — they will error, not warn. Use `IDENTIFIED BY '<plaintext>'` and
> let the server hash it, or `IDENTIFIED WITH caching_sha2_password BY '...'`.
> Also: `caching_sha2_password` is the MySQL 8 default and requires either TLS or
> RSA key exchange for the initial handshake. Since the organization enforces TLS anyway this
> is not an obstacle, but a replica configured without `SOURCE_SSL=1` will fail to
> authenticate in a way that looks like a wrong password.

> **Changed since the 2018 docs — password length.** The old note "PW max length
> 32 characters" reflected the old hashing scheme. It no longer applies; use a
> long generated password from 1Password.

**The replication account password must be rotated.** It is recorded in plaintext
in at least two superseded files. [FILL IN: date rotated.]

### Point the replica at the primary

Get the coordinates from the primary:

    SHOW BINARY LOG STATUS;                          # note File and Position

Then on the replica — current syntax:

    STOP REPLICA;
    CHANGE REPLICATION SOURCE TO
      SOURCE_HOST='<primary-ip>',                    # IP, not name: MySQL does not resolve DNS here
      SOURCE_USER='replicant',
      SOURCE_PASSWORD='<from 1Password>',
      SOURCE_LOG_FILE='<file from above>',
      SOURCE_LOG_POS=<position from above>,
      SOURCE_SSL=1;                                  # required, transport is enforced
    START REPLICA;
    SHOW REPLICA STATUS\G

Legacy syntax for a host still on 5.7 or early 8.0:

    CHANGE MASTER TO master_host='<ip>', master_user='replicant', master_password='<pw>',
      master_log_file='<file>', master_log_pos=<pos>, MASTER_SSL=1;
    START SLAVE;

[CONFIRM: proposed default — move to GTID-based replication
(`gtid_mode=ON`, `enforce_gtid_consistency=ON`, `SOURCE_AUTO_POSITION=1`). GTIDs
remove the entire class of "what file and position were we at" errors and make
failover and re-pointing dramatically safer. This is a change with real
prerequisites and should be planned, not done in passing.] [FILL IN: whether GTID
is currently enabled on db01/db02.]

### Verifying replication

Healthy state — all of:

    SHOW REPLICA STATUS\G

    Replica_IO_Running:    Yes        # reading the source's binary log
    Replica_SQL_Running:   Yes        # applying relay log to the local database
    Seconds_Behind_Source: 0
    Last_IO_Error:         (empty)
    Last_SQL_Error:        (empty)

With row-based replication, `Seconds_Behind_Source: 0` alone is **not** proof the
replica is caught up — a large transaction can still be mid-apply. Also check that
`Relay_Log_Space` has stopped growing and that `Replica_SQL_Running_State` shows
an idle state such as "Replica has read all relay log" or "Waiting for source to
send event". This point is made forcefully in
`Databases/mysql replication patching.txt` and it is the single most important
operational detail in this whole section.

    watch -n 5 'mysql -e "SHOW REPLICA STATUS\G" | egrep "Replica_IO_Running:|Replica_SQL_Running:|Seconds_Behind_Source:|Relay_Log_Space:|Replica_SQL_Running_State:"'

Functional test: make a change on the primary, read it back on the replica.
**Never write to the replica.** A write on the replica breaks replication, often
subtly, and the fix is usually a rebuild. Consider enforcing this:

    SET GLOBAL read_only = ON;                       # blocks writes from normal accounts
    SET GLOBAL super_read_only = ON;                 # blocks writes from SUPER accounts too

### Reading binary and relay logs

    mysqlbinlog /var/log/mysql/mysql-bin.000001 | less                       # decode a binary log
    mysqlbinlog --base64-output=decode-rows --verbose <binlog>               # readable row events
    mysqlbinlog -d <dbname> <binlog> > <dbname>-events.txt                   # one database only

With row-based logging, `--base64-output=decode-rows --verbose` is essential —
without it the row events are opaque base64 and the log looks empty of content.
Under the old STATEMENT format you could read the raw SQL directly, which is why
the old wiki does not mention these flags.

Relay logs live in the datadir on the replica and rotate and delete themselves
once applied. A growing pile of relay logs means the SQL thread is not keeping up
or has stopped — check `Replica_SQL_Running` first.

### Patching a replicated pair

Do not improvise this. Follow the current procedures:

- **`Databases/mysql replication patching.txt`** — the RBR-aware runbook. Replica
  first, verify fully idle, optionally set the source read-only, then patch the
  source. This is the authoritative one.
- **`Databases/mysql_patching_guide_replica.txt`** — the condensed checklist form,
  dated 2026-04-15, targeting Ubuntu 24.04. Note it uses the legacy
  `STOP SLAVE` / `SHOW SLAVE STATUS` vocabulary and a
  `FLUSH TABLES WITH READ LOCK` on the source where the longer procedure prefers
  `super_read_only`. The longer document's approach is safer.

The failure this ordering prevents: patching the source first, rebooting it, and
discovering the replica was 40 minutes behind with unapplied row events.

---

## Backups

Three layers, each covering the others' gaps:

1. **Nightly logical dump** (`mysqldump`) — the restorable baseline.
2. **Binary log copies** — everything since the last dump, for point-in-time recovery.
3. **Retrospect** — takes the dump files and the host filesystem offsite.

If only the first layer exists, the worst case is losing up to 24 hours of data.

### Credentials without plaintext

`mysql_config_editor` stores credentials in `~/.mylogin.cnf`, obfuscated and
readable only by the owner, so cron jobs and scripts need no inline password.

    mysql_config_editor set --login-path=backups --host=localhost --user=backups --password
    mysql_config_editor print --all                  # verify; password shows as *****

Wrap the password in double quotes when prompted so shell metacharacters such as
`#` are not interpreted.

> Important caveat the old docs do not state: `.mylogin.cnf` is **obfuscated, not
> encrypted**. Anyone who can read the file can recover the password. It protects
> against shoulder-surfing and casual `ps` inspection, not against a compromised
> account. Keep file permissions tight and treat the host as holding a credential.

[VERIFY: was valid for MySQL 5.7 / Ubuntu 18, confirm on current version — the old
docs configure a `remote` login-path against `appdb1.example.com:13306` and
`db01.example.com:13306`. Port 13306 is non-standard; confirm whether that port
forward or alternate listener still exists.]

### Nightly dump

    /usr/bin/mysqldump --login-path=backups --opt --single-transaction --routines \
      --default-character-set=utf8mb4 --comments --source-data=2 --dump-date \
      --databases <dbname> > /home/orgadmin/<db>MySQLBackUps/<db>dump_$(date +%Y-%m-%d).sql

Flag by flag:

- `--login-path=backups` — credentials from `.mylogin.cnf`, nothing on the command line.
- `--single-transaction` — consistent snapshot of InnoDB without locking writers.
- `--routines` — include stored procedures and functions, which `--opt` does not.
- `--source-data=2` — write the binary log file and position into the dump as a
  comment, so point-in-time recovery knows where to resume.
- `--dump-date` — timestamp in the file.

> **Changed since the 2018 docs — three flags.**
> `--master-data=2` is deprecated in MySQL 8.0.26 and replaced by
> `--source-data=2`; same meaning.
> `--default-character-set=utf8` should be `utf8mb4`. In MySQL 8, `utf8` is an
> alias for the 3-byte `utf8mb3` and cannot represent emoji or some CJK
> characters — dumping with it can silently mangle data that the server stores
> correctly.
> The backticked `` `date ...` `` form in the old crontab entries works but
> `$(date ...)` is clearer; in **crontab** either way you must still escape `%`
> as `\%`, because cron treats an unescaped `%` as a newline. That old warning is
> still entirely correct and still catches people.

### Cron

As root, `crontab -e`:

    SHELL=/bin/sh
    PATH=/usr/sbin:/usr/bin:/sbin:/bin

    0 3 * * * /bin/sh /home/orgadmin/MySQLNightlyBackup.sh     # nightly logical dump at 03:00

Giving cron an explicit `SHELL` and `PATH` at the top of the crontab is still
necessary and is a common cause of "it works when I run it by hand".

The script itself contains the `mysqldump` line above. [FILL IN: current script
path, which databases each host dumps, retention of old dump files, and whether
dumps are compressed.] Nothing in the old docs prunes old dumps, so
[FILL IN: confirm the dump directory is not slowly filling the disk] — see the
disk capacity section of `SysAdmin Procedures/monitoring-alerting-guide.md`.

**Nothing alerts when this job fails.** A dump that has not run for three weeks
looks exactly like one that ran last night, until you need it.

### Binary log backup (point-in-time recovery)

Full backups give you last night. Binary logs give you the minutes before the
incident. Stream them continuously with `mysqlbinlog --read-from-remote-server`:

    mysqlbinlog -R --raw --host=db01.example.com --login-path=backups \
      --result-file=/home/orgadmin/AppDB1MySQLBackUps/ --to-last-log mysql-bin.000009

Notes from hard experience recorded in the old files: `--raw` plus a
`--result-file=` **directory** (with the trailing slash) is what actually works;
the same command without `--raw` and with a file path did not. `--stop-never` runs
it as a continuous streaming process rather than a one-shot copy.

Critically — and the old docs make this point well — **binary logs must live on
different storage from the datadir.** If they are on the disk that died, they are
gone with it, and the point-in-time capability was imaginary.

[FILL IN: where binary logs are actually copied to today, and whether the copy is
running.]

Retention of binary logs on the server:

    SHOW VARIABLES LIKE 'binlog_expire_logs_seconds';     # MySQL 8 retention control
    PURGE BINARY LOGS BEFORE '2026-09-01 00:00:00';       # manual purge

> **Changed since the 2018 docs — binlog retention variable.** `expire_logs_days`
> is deprecated; MySQL 8 uses `binlog_expire_logs_seconds` (default 2592000, i.e.
> 30 days). Never delete binary log files from the filesystem by hand — MySQL
> maintains a `.index` file and will be confused by files that vanish. And never
> purge logs a replica has not yet read.

### Recovery

Restore the full dump, then replay binary logs forward.

    mysql --login-path=backups < /path/to/dump_2026-09-07.sql        # restore the baseline

Read the `CHANGE MASTER TO` / `CHANGE REPLICATION SOURCE TO` comment near the top
of the dump — that is the log file and position where the dump was taken, and
where replay must begin.

Replay to a point in time:

    mysqlbinlog --stop-datetime="2026-09-08 09:59:59" mysql-bin.000009 | mysql -u root -p
    mysqlbinlog --start-datetime="2026-09-08 10:01:00" mysql-bin.000009 | mysql -u root -p

The two commands above are the classic "undo the bad DELETE at 10:00" pattern:
replay everything up to one second before it, then everything from one second
after. Positions are more precise than timestamps when several statements share a
second:

    mysqlbinlog --start-datetime="2026-09-08 09:55:00" --stop-datetime="2026-09-08 10:05:00" \
      mysql-bin.000009 > /tmp/mysql_restore.sql          # inspect first, find the exact positions
    mysqlbinlog --stop-position=368312  mysql-bin.000009 | mysql -u root -p
    mysqlbinlog --start-position=368315 mysql-bin.000009 | mysql -u root -p

Always dump the log to a file and read it before piping anything into `mysql`.

**Last tested restore: [FILL IN: date]. An untested restore is a rumour.**

### Backing up a replica

If you back up db02.example.com rather than the primary — a legitimate strategy to keep
dump load off the primary — you must also back up the replication state
(`master.info` / relay-log.info, or their table equivalents in MySQL 8) or the
restored replica cannot resume replication.

> **Changed since the 2018 docs — replication metadata location.** MySQL 8 stores
> replication metadata in InnoDB tables (`mysql.slave_master_info`,
> `mysql.slave_relay_log_info`) by default, not in the `master.info` and
> `relay-log.info` files the old documentation describes. A dump that includes the
> `mysql` schema captures them; a dump of application databases only does not.

### Retrospect

Retrospect takes the dump files and host filesystem offsite.

    sudo apt install -y lib32stdc++6                         # 32-bit dependency for the client
    wget <retrospect client tarball URL>
    tar -xvf Linux_Client_x86_<version>.tar
    sudo ./Install.sh                                        # answer yes to install and auto-start

Then set the client password so the Retrospect server can see the host.

> [VERIFY: was valid for MySQL 5.7 / Ubuntu 18, confirm on current version — the
> pinned client version in the old docs is 15.1.2.101 from 2018, fetched from an
> S3 URL that may no longer exist, and `lib32stdc++6` availability on Ubuntu 24.04
> needs checking (it requires i386 multiarch to be enabled). Get the current
> client from the Retrospect console or vendor site.]

[FILL IN: Retrospect server hostname, which MySQL hosts have the client installed,
backup script/set names, schedule, and where the offsite copy goes.] See
`Hardware & Backup/` and `Retrospect Restore/`.

### MyISAM maintenance

Only relevant if any MyISAM tables remain. Everything created this decade is
InnoDB.

    mysqlcheck --login-path=backups --all-databases --check         # server-side check
    # or: CHECK TABLE / REPAIR TABLE / OPTIMIZE TABLE / ANALYZE TABLE

Prefer the SQL statements or `mysqlcheck` over `myisamchk`: the server does the
work and there is no risk of the tool and the server touching the same files at
once. `myisamchk` must only be run with mysqld stopped.

[VERIFY: was valid for MySQL 5.7 / Ubuntu 18, confirm on current version — whether
any database still contains MyISAM tables. Check with:]

    SELECT table_schema, table_name, engine FROM information_schema.tables
      WHERE engine <> 'InnoDB' AND table_schema NOT IN
      ('mysql','information_schema','performance_schema','sys');

---

## MySQL in Docker

The organization runs Docker, and `_ARCHIVE/superseded-stubs/MySQL in Docker.txt` (archived 2026-09-11) currently holds a set of
reference links about running MySQL and MySQL InnoDB Cluster in containers. That
material is captured here so the link dump can be retired.

**Status: reference only.** [FILL IN: whether any MySQL instance actually runs
in Docker today, and if so on which host and for what purpose.] Nothing in the
library indicates a containerised MySQL is in production.

### If you do run MySQL in a container

The rules that matter, and the ones people get wrong:

- **Persist the data.** A container's writable layer is disposable. The datadir
  must be a named volume or bind mount, or the database is gone when the container
  is recreated on the next image update.

      docker volume create mysql-data                 # named volume for /var/lib/mysql

- **Persist and own the config.** Mount a `.cnf` into
  `/etc/mysql/conf.d/` rather than editing inside a running container.
- **Pin the image tag** to a specific 8.x version, never `latest`. An unattended
  `docker compose pull` that jumps a major version can trigger an irreversible
  datadir upgrade.
- **Never put the root password in the compose file** in a repo or in this
  library. Use `MYSQL_ROOT_PASSWORD_FILE` with a Docker secret, or an env file
  excluded from version control.
- **TLS applies equally.** `require_secure_transport = ON` is a standard, not
  a host-specific setting. Mount the certificates in and configure the paths.
- **Backups do not come free.** Run `mysqldump` from inside the container on a
  schedule and write the output to a mounted host path that Retrospect covers:

      docker exec <container> mysqldump --single-transaction --routines \
        --source-data=2 --databases <db> > /backups/<db>_$(date +%F).sql

- **Upgrades are not a restart.** Moving a container from 8.0 to 8.4 runs the data
  dictionary upgrade against the persisted volume and is not reversible. Take a
  dump first.
- **InnoDB Cluster in Docker** (Group Replication + MySQL Router + MySQL Shell) is
  a substantially more complex system than the async primary/replica pair the organization runs
  today. It solves automatic failover. It introduces quorum, split-brain and
  network-partition behaviour that must be understood before it is deployed. Do
  not adopt it because it appeared in a tutorial. If the organization wants automatic failover,
  that deserves its own design document and its own guide.

### Reference links (carried forward)

- https://www.datacamp.com/tutorial/set-up-and-configure-mysql-in-docker — single-instance MySQL in Docker
- https://diptochakrabarty.medium.com/setting-mysql-cluster-using-docker-f0e405d03762 — MySQL cluster with Docker
- https://dev.mysql.com/blog-archive/docker-compose-setup-for-innodb-cluster/ — Oracle's own docker-compose InnoDB Cluster setup
- https://medium.com/@ahmedamedy/mysql-clustering-with-docker-611dc28b8db7 — MySQL clustering with Docker

---

## Operations (Day-2)

### Start / Stop / Restart

    sudo systemctl restart mysql                     # full restart, brief outage
    sudo systemctl status mysql --no-pager           # state and recent log lines

### Health check

    mysqladmin --login-path=backups version          # quick liveness
    mysqladmin --login-path=backups status           # uptime, threads, queries
    SHOW PROCESSLIST;                                # what is running right now
    SHOW ENGINE INNODB STATUS\G                      # locks, deadlocks, buffer pool

### Logs

    tail -f /var/log/mysql/error.log                 # the file that actually matters
    journalctl -u mysql --since today                # systemd view, startup failures

[FILL IN: whether MySQL hosts ship logs to graylog01.example.com. As of this draft
nothing in the library says they do — see `SysAdmin Procedures/monitoring-alerting-guide.md`.]

### Patching

Standalone hosts: normal OS patching, then confirm MySQL came back.

    sudo apt update && sudo apt -y dist-upgrade
    sudo reboot
    systemctl status mysql --no-pager
    mysqladmin --login-path=backups version

Replicated pair: **use `Databases/mysql replication patching.txt`.** Replica first.

Major version upgrades (8.0 -> 8.4) are not routine patching. Take a dump,
read the upstream release notes for removed options, and check for the removed
replication commands and removed TLS variables listed in the callouts above.

### Certificate rotation

See **`Databases/mysql certificate rotation.txt`**. Summary of why it is separate:
MySQL 8 supports `ALTER INSTANCE RELOAD TLS;`, which picks up replaced certificate
files without a restart and without dropping existing connections. Confirm key
permissions before reloading — a permissions error makes the reload fail in a way
that is easy to miss. If the CA itself is changing, deploy the new CA to clients
first and trust both CAs during the transition, or every client will fail
verification at once.

---

## Troubleshooting

### Symptom: mysqld will not start, journalctl shows AppArmor DENIED

- Likely cause: the AppArmor profile for `/usr/sbin/mysqld` is blocking a path
  MySQL needs, often after an upgrade or a datadir move.
- Check:

    journalctl -xe | grep -i apparmor                    # look for DENIED lines and the "name=" path

- Fix: add the denied paths to the local override, one per line, **each line
  starting with two spaces**:

    sudo nano /etc/apparmor.d/local/usr.sbin.mysqld

      /proc/*/status r,
      /sys/devices/system/node/ r,
      /sys/devices/system/node/*/meminfo/ r,

    sudo systemctl reload apparmor                       # apply
    sudo systemctl start mysql

  Take the path from the `name=` field of the denial. A trailing `r,` grants read.
  This procedure comes straight from the 2018 troubleshooting wiki and is still
  correct. If you move the datadir or the certificate location, expect to do this.

### Symptom: a client cannot connect, "SSL connection error" or immediate refusal

- Likely cause: `require_secure_transport = ON` and the client is not using TLS,
  or the client's TLS version is below the server's minimum.
- Check: `SHOW VARIABLES LIKE 'require_secure_transport';` and
  `SHOW GLOBAL VARIABLES LIKE 'tls_version';`
- Check from a client: `openssl s_client -connect <host>:3306 -starttls mysql`
- Fix: configure TLS on the client. Do **not** turn off
  `require_secure_transport` to make a client work — that weakens every
  connection to solve one client's problem.

### Symptom: `Replica_IO_Running: Connecting`

- Likely cause: the replication account cannot authenticate or reach the source —
  wrong password, account not granted from the replica's address, port blocked, or
  (on MySQL 8) the connection is not using TLS and `caching_sha2_password` refuses.
- Check: `Last_IO_Error` in `SHOW REPLICA STATUS\G`. It usually names the cause
  directly, e.g. "error connecting to master 'replicant@...:3306'".
- Check on the source:

    SHOW VARIABLES LIKE '%port%';                        # confirm 3306
    SELECT user, host FROM mysql.user WHERE user='replicant';   # granted from the right address?

- Check the network path from the replica: `nc -vz <primary-ip> 3306`
- Fix: correct the grant's host, reset the password, open the firewall, or add
  `SOURCE_SSL=1`.

### Symptom: `Replica_SQL_Running: No` with a `Last_SQL_Error`

- Likely cause: a statement or row event could not be applied — commonly a
  duplicate key because the object already existed on the replica, or drift caused
  by a write made directly on the replica.
- Check: read `Last_SQL_Error` in full. It names the table and the failing event.
- Fix, only when you understand why it failed and are certain the event is safely
  skippable:

    STOP REPLICA;
    SET GLOBAL SQL_REPLICA_SKIP_COUNTER = 1;             # MySQL 8.0.26+ (was SQL_SLAVE_SKIP_COUNTER)
    START REPLICA;

  **Do not skip blindly.** Each skipped event is data the replica does not have.
  A duplicate-key error after a restart may be harmless; the same error caused by
  drift means the replica is already wrong and skipping makes it worse. With
  row-based replication, investigate before skipping. If GTIDs are enabled, the
  skip counter does not apply — use `gtid_next` to inject an empty transaction
  instead.

### Symptom: relay logs piling up on the replica

- Likely cause: the SQL thread is stopped or cannot keep up. The IO thread writes
  relay logs, the SQL thread consumes them.
- Check: `Replica_SQL_Running`, `Relay_Log_Space`, and
  `Replica_SQL_Running_State`. A state of "Applying batch of row changes" on a
  large transaction is normal and will clear; a stopped SQL thread will not.
- Fix: clear the SQL error and restart the SQL thread. If it is genuinely a
  throughput problem, that is a capacity conversation, not a quick fix.

### Symptom: `Seconds_Behind_Source` reads 0 but the replica is not actually current

- Likely cause: row-based replication mid-transaction. The field is measured
  against the last event's timestamp and can read 0 while a large transaction is
  still applying.
- Check: `Relay_Log_Space` stable, `Replica_SQL_Running_State` idle, and
  `Exec_Source_Log_Pos` close to `Read_Source_Log_Pos`.
- Fix: wait. Do not start maintenance on the source based on the 0.

### Symptom: disk full on a MySQL host

- Likely cause: binary logs never purged, dump files never pruned, or genuine
  data growth. MySQL stops accepting writes and can leave tables in a bad state.
- Check:

    df -h /var/lib/mysql /var/log/mysql               # datadir and log filesystems
    du -sh /var/log/mysql/mysql-bin.*                 # binary log footprint
    du -sh /home/orgadmin/*MySQLBackUps              # accumulated dump files

- Fix: purge binary logs **through MySQL** (`PURGE BINARY LOGS BEFORE ...`), never
  with `rm`, and only once the replica has read them. Prune old dumps. Then set
  `binlog_expire_logs_seconds` and add a dump retention rule so it does not recur.
- Reference: https://dev.mysql.com/doc/refman/8.0/en/full-disk.html

### Symptom: a statement ends without executing in an interactive session

- Likely cause: a stored procedure or trigger body containing `;`.
- Fix: `DELIMITER //` to change the statement terminator, then `DELIMITER ;` after.

---

## Security

- Exposure: 3306 on each host. [FILL IN: which hosts accept remote connections and
  from which networks.] TLS is mandatory (`require_secure_transport = ON`).
- Auth: MySQL accounts, per host. [FILL IN: whether any MySQL instance
  authenticates against AD/LDAP.] MySQL 8 defaults to `caching_sha2_password`.
- Certificates: wildcard certificate and CA bundle in `/etc/mysql`, key mode 0640
  `root:mysql`. Rotation: `Databases/mysql certificate rotation.txt`. Expiry is not
  currently monitored — see `SysAdmin Procedures/monitoring-alerting-guide.md`.
- Secrets: 1Password, never in this library. The superseded files violate this
  rule; treat every credential in them as burned and rotate it.
- Accounts to audit: [FILL IN: review `SELECT user, host FROM mysql.user;` on each
  host and remove anything unrecognised, especially accounts with `%` as host.]
- Hardening: `mysql_secure_installation` at build time; `Security & Hardening/fresh-box-hardening-cheatsheet.txt`
  for the host.

## Disaster Recovery

- RTO / RPO: [FILL IN: agreed targets per database.] Current technical capability
  without further work: RPO of up to 24 hours from nightly dumps alone, improving
  to minutes if and only if binary log copying is verified to be running.
- Rebuild order for a lost primary: provision host -> install and configure MySQL
  with TLS -> restore the most recent dump -> replay binary logs to the failure
  point -> re-establish replication -> re-point application connections and ODBC DSNs.
- The replica is **not** a backup. It replicates destructive statements faithfully.
- Escalation: [FILL IN: contact order if the owner is unavailable.]

## Superseded Documents

This guide replaces the following. They should be archived, not deleted — they
contain host-specific history that may still be needed — but they must not be
followed as current procedure, and they contain plaintext credentials that must be
rotated.

| Path | Why superseded |
|---|---|
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/1 - Base Linux set up.txt` (archived 2026-09-11) | Ubuntu 16.04, `ifconfig`, `/etc/network/interfaces`, System76 driver PPA |
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/2 - MySQL install and configuration.txt` (archived 2026-09-11) | MySQL 5.7 install; root password procedure no longer applies |
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/3 - Secure MySQL.txt` (archived 2026-09-11) | TLS config carried forward; `have_ssl` checks deprecated |
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/4 - Troubleshooting MySQL.txt` (archived 2026-09-11) | AppArmor procedure carried forward verbatim |
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/5 - Configure MySQL ODBC.txt` (archived 2026-09-11) | Obsolete packages and connector versions; contains two command typos |
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/6 - MySQL Replication Set up.txt` (archived 2026-09-11) | Pre-8.0 replication syntax; `STATEMENT` binlog format; `PASSWORD()` hashing; contains a broken ssl-key config block |
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/7 - Troubleshooting MySQL Replication.txt` (archived 2026-09-11) | Old field and command names; ends mid-sentence |
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/8 - Configure MySQL Backups.txt` (archived 2026-09-11) | `--master-data=2`, `utf8` charset |
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/9 - Configure Linux Retrospect.txt` (archived 2026-09-11) | Pinned 2018 client version and dead download URL |
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/10 - Backup strategy and methodology.rtf` (archived 2026-09-11) | MySQL 5.7 backup excerpt; content carried forward |
| `_ARCHIVE/superseded-2018-mysql/MySQL wikis/11 - MySQL References.txt` (archived 2026-09-11) | Link list carried forward; **contains plaintext replication password** |
| `_ARCHIVE/superseded-2018-mysql/Current Wiki.txt` (archived 2026-09-11) | Duplicates wikis 1-2 plus appdb1 specifics; Ubuntu 16.04 |
| `_ARCHIVE/superseded-2018-mysql/SSL and MYSQL.txt` (archived 2026-09-11) | CSR procedure carried forward; no SAN support; MySQL 5.0 caveats |
| `_ARCHIVE/superseded-2018-mysql/Binlog backup.txt` (archived 2026-09-11) | Duplicate of wiki 10 in plain text; content carried forward |
| `_ARCHIVE/superseded-2018-mysql/backup script and cron.txt` (archived 2026-09-11) | Scratch notes; **contains a plaintext backup account password** |
| `_ARCHIVE/superseded-2018-mysql/Ubuntu 18 wiki.txt` (archived 2026-09-11) | Ubuntu 18 netplan build for db01.example.com; superseded by 24.04 baseline |
| `_ARCHIVE/superseded-2018-mysql/AppDB1SetupGuide.txt` (archived 2026-09-11) | Composite of the above for appdb1; **contains plaintext root, MySQL root and replication passwords** |
| `_ARCHIVE/superseded-2018-mysql/AppDB2 wiki.txt` (archived 2026-09-11) | AppDB2 build notes; MySQL portion carried forward, desktop/VNC portion is host-specific and should move to its own host doc |
| `_ARCHIVE/superseded-stubs/MySQL in Docker.txt` (archived 2026-09-11) | 331-byte link dump; links carried forward into the Docker section above |

Not superseded, still current — defer to these:

- `Databases/mysql certificate rotation.txt`
- `Databases/mysql replication patching.txt`
- `Databases/mysql_patching_guide_replica.txt`

Not superseded, retained as-is:

- `Databases/MySQL Set up/Official Docs/` — vendor PDFs (note: the backup and ODBC
  excerpts are the 5.7 editions)
- `Databases/MySQL Set up/Installers/` — archived connector and Retrospect tarballs
- `_ARCHIVE/superseded-2018-mysql/Mysql Beginner Course UDemy.txt` (archived 2026-09-11), `_ARCHIVE/superseded-2018-mysql/history.txt` (archived 2026-09-11) — training
  and shell-history material, not procedure

## Decisions & History (ADR-lite)

| Date       | Decision / Change                                              | Why / Ticket |
|------------|-----------------------------------------------------------------|--------------|
| 2018-11    | Replication established db01.example.com -> db02.example.com, STATEMENT format | Original build |
| [FILL IN]  | Moved to ROW-based replication                                  | [FILL IN — confirmed in use by 2026 patching docs, date unknown] |
| 2026-04-15 | Replica patching procedure written for Ubuntu 24.04            | `mysql_patching_guide_replica.txt` |
| 2026-09-11 | This guide consolidates and supersedes the 2018 wiki set        | Library consolidation |

## References

Internal:

- `Databases/mysql certificate rotation.txt`
- `Databases/mysql replication patching.txt`
- `Databases/mysql_patching_guide_replica.txt`
- `SysAdmin Procedures/monitoring-alerting-guide.md`
- `SysAdmin Procedures/Backup_DR_Runbook.txt`
- `SysAdmin Procedures/Patch_Management_Workflow.txt`
- `Security & Hardening/fresh-box-hardening-cheatsheet.txt`
- `Hardware & Backup/`, `Retrospect Restore/` — Retrospect
- `Dynamics GP & DMS/` — the main ODBC consumer

Upstream (current versions; the old docs linked 5.7 and Ubuntu 16.04 editions):

- MySQL 8.0 Reference Manual: https://dev.mysql.com/doc/refman/8.0/en/
- Replication: https://dev.mysql.com/doc/refman/8.0/en/replication.html
- Point-in-time recovery: https://dev.mysql.com/doc/refman/8.0/en/point-in-time-recovery.html
- mysqlbinlog: https://dev.mysql.com/doc/refman/8.0/en/mysqlbinlog.html
- mysql_config_editor: https://dev.mysql.com/doc/refman/8.0/en/mysql-config-editor.html
- Connector/ODBC: https://dev.mysql.com/doc/connector-odbc/en/
- Full disk handling: https://dev.mysql.com/doc/refman/8.0/en/full-disk.html
- Encrypted connections: https://dev.mysql.com/doc/refman/8.0/en/encrypted-connections.html

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

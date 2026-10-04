> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Reverse Proxy and TLS Certificate Automation (example.com)

## Overview

Most web services are not exposed directly. Apache sits in front, terminates TLS, and proxies to an application listening on a local port or another internal host. This guide covers the Apache virtual host pattern used in this environment, how to add a new proxied service end to end, how Let's Encrypt certificates are issued and renewed automatically, the difference between the wildcard certificate and per-host certificates and when each is correct, how a silently failed renewal turns into an outage and how to catch it first, the HTTP security header baseline, and troubleshooting for the three failures that actually happen: 502s, certificate chain errors, and renewal failures.

Apache is the front end on several hosts — db01.example.com, db02.example.com, sign.example.com, appdb2.example.com, appdb1.example.com, erpdb.example.com, prod01.example.com, files.example.com, forums.example.com among them. Each is configured independently, which means this guide describes a pattern rather than a single system, and the per-host specifics have to be confirmed on each host.

## Quick Facts

| Field            | Value                                                                                          |
|------------------|------------------------------------------------------------------------------------------------|
| Environment      | prod                                                                                            |
| Location         | Apache on the individual service hosts: db01.example.com, db02.example.com, sign.example.com, appdb2.example.com, appdb1.example.com, erpdb.example.com, prod01.example.com, files.example.com, forums.example.com — [FILL IN: confirm the complete list of hosts running Apache as a front end, and which are Linux vs macOS] |
| Access           | SSH to each host (see `Security & Hardening/ssh keys.rtf` and `Security & Hardening/sshd hardening conf.txt`); inbound HTTPS via pfSense port forward from the public IP |
| Dependencies     | DNS (both views — internal for clients, external for ACME validation and public access), pfSense NAT and firewall rules for ports 80/443, Let's Encrypt / ACME reachability, the backend application on each host, `systemd` timers or cron for renewal |
| Dependents       | Every user-facing web service in this environment. A failed renewal or a broken vhost takes down the service, not just its TLS. |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                                                        |

## How It Works

### The request path

```
Client → public DNS (example.com) → pfSense WAN (public IP)
       → NAT port forward 443 → Apache on the service host (TLS terminated here)
       → mod_proxy → backend application (localhost:<port> or an internal host)
```

For internal clients the path is shorter — internal DNS resolves the host to its internal address and the client reaches Apache directly, skipping the NAT. This is the split-horizon behaviour described in `Networking Guide/dns-dhcp-administration-guide.md`, and it matters here: a certificate problem may be visible externally and invisible internally, or the reverse, depending on which name the client used.

TLS is terminated at Apache. The connection from Apache to the backend is normally plain HTTP over loopback. Where the backend is on a different host, that hop crosses the network and should be encrypted or confined to a trusted VLAN — [FILL IN: are there any cross-host proxy hops, and are they encrypted?]

### The Apache vhost pattern

The existing pattern is documented in `Linux & Servers/Apache Virtual Host Setup.pdf`. That document describes the macOS Apache layout: virtual hosts are enabled by uncommenting the include in `/etc/apache2/httpd.conf`

```
# Virtual hosts
Include /private/etc/apache2/extra/httpd-vhosts.conf
```

and the vhosts themselves are defined in `/private/etc/apache2/extra/httpd-vhosts.conf`, all in one file, each with `ServerName`, `DocumentRoot`, a `ServerAdmin`, per-vhost `ErrorLog` and `CustomLog` paths named after the hostname, and a matching `<Directory>` block granting access. Apache is reloaded with `sudo apachectl restart`.

That PDF's examples are **port 80 only, document-root serving, no TLS and no proxying** — it predates the current setup and is the historical pattern, not the target pattern. Two things about it are still worth carrying forward and are used below: the per-vhost log naming convention (`<hostname>-error_log` / `<hostname>-access_log`, which makes "which vhost is failing" answerable from `ls`), and the `ServerAdmin` field.

On Linux hosts the layout differs and is the more common case in this environment now:

| | macOS Apache | Debian/Ubuntu Apache |
|---|---|---|
| Main config | `/etc/apache2/httpd.conf` | `/etc/apache2/apache2.conf` |
| Vhosts | one file, `extra/httpd-vhosts.conf` | one file per site in `/etc/apache2/sites-available/`, symlinked from `sites-enabled/` |
| Enable a site | edit the shared file | `a2ensite <name>.conf` |
| Enable a module | edit `httpd.conf` | `a2enmod <module>` |
| Logs | `/private/var/log/apache2/` | `/var/log/apache2/` |
| Reload | `sudo apachectl restart` | `sudo systemctl reload apache2` |

[FILL IN: which hosts are macOS Apache and which are Debian/Ubuntu? The commands below assume Debian/Ubuntu; translate per the table for macOS hosts.]

### The target vhost pattern for a new proxied service

Two vhosts per service: a port 80 vhost that exists only to redirect and to answer ACME challenges, and a port 443 vhost that terminates TLS and proxies.

```apache
# /etc/apache2/sites-available/app.example.com.conf

<VirtualHost *:80>
    ServerName app.example.com
    ServerAdmin it@example.com

    # Leave this reachable: Let's Encrypt HTTP-01 validation needs it.
    # Serve the challenge path from disk, redirect everything else.
    Alias /.well-known/acme-challenge/ /var/www/html/.well-known/acme-challenge/
    <Directory "/var/www/html/.well-known/acme-challenge/">
        Require all granted
    </Directory>

    RedirectMatch permanent "^/(?!\.well-known/acme-challenge/)(.*)" "https://app.example.com/$1"

    ErrorLog  ${APACHE_LOG_DIR}/app.example.com-error_log
    CustomLog ${APACHE_LOG_DIR}/app.example.com-access_log combined
</VirtualHost>

<VirtualHost *:443>
    ServerName app.example.com
    ServerAdmin it@example.com

    SSLEngine on
    SSLCertificateFile    /etc/letsencrypt/live/app.example.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/app.example.com/privkey.pem

    # Proxy to the backend. ProxyPreserveHost keeps the original Host header so
    # the app generates correct absolute URLs instead of localhost ones.
    ProxyPreserveHost On
    ProxyPass        / http://127.0.0.1:3000/
    ProxyPassReverse / http://127.0.0.1:3000/

    # Tell the backend the original scheme — apps behind a proxy otherwise
    # think they are on plain HTTP and emit http:// redirects that loop.
    RequestHeader set X-Forwarded-Proto "https"

    # Security headers — see the baseline section below
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"

    ErrorLog  ${APACHE_LOG_DIR}/app.example.com-error_log
    CustomLog ${APACHE_LOG_DIR}/app.example.com-access_log combined
</VirtualHost>
```

Two details in that config are the cause of a large share of proxy problems and are worth stating plainly:

- **`ProxyPreserveHost On`** — without it, Apache sends `Host: 127.0.0.1:3000` to the backend. Applications that build absolute URLs from the Host header then emit links pointing at localhost, which fail for every user.
- **`X-Forwarded-Proto https`** — without it, the backend believes the request arrived over HTTP and issues a redirect to `http://`, which Apache's port-80 vhost redirects back to HTTPS, which the backend redirects to HTTP again. The symptom is an infinite redirect loop, and it looks like a certificate problem to the reporting user.

Required modules:

```
sudo a2enmod ssl proxy proxy_http headers rewrite    # enable everything the vhost above uses
sudo systemctl restart apache2                       # module changes need a restart, not a reload
```

### Where things live

| Thing | Location |
|---|---|
| Vhost configs | `/etc/apache2/sites-available/`, enabled via `sites-enabled/` symlinks (Debian/Ubuntu) |
| Certificates and keys | `/etc/letsencrypt/live/<name>/` — `fullchain.pem`, `privkey.pem`, `cert.pem`, `chain.pem` |
| Certbot config and account | `/etc/letsencrypt/` — back this whole directory up |
| Per-renewal config | `/etc/letsencrypt/renewal/<name>.conf` — records the authenticator, the domains, and the deploy hook |
| Renewal automation | `certbot.timer` (systemd) or `/etc/cron.d/certbot` — see below |
| Apache logs | `/var/log/apache2/<hostname>-error_log` and `-access_log` |
| Wildcard certificate | [FILL IN: where does the wildcard.example.com certificate live, which CA issued it, and what is its expiry? The MySQL configs in `Databases/MySQL Set up/` reference `/etc/mysql/wildcardcert.crt` and `/etc/mysql/wildcardkey.pem`, with a `thawteroot.pem` alongside — suggesting a commercial wildcard, not Let's Encrypt] |

---

## Operations (Day-2)

### Adding a new proxied service, end to end

The order matters. Doing DNS last means the certificate request fails; doing the certificate before the firewall means validation fails.

**1. Decide the hostname and which certificate it will use.** Per-host Let's Encrypt or the wildcard — see the wildcard section below. This decision changes steps 5 and 6.

**2. Create DNS records.** Both views, if the service is reachable from both.

- Internal: `dnsmgmt.msc` or PowerShell on a DC — add the A record and PTR.
- External: the public zone at [FILL IN: external DNS provider], if the service is internet-facing.

Verify before continuing. ACME validation will fail against a name the world cannot resolve:

```
nslookup app.example.com                    # internal view resolves
dig @1.1.1.1 app.example.com +short         # external view resolves — required for HTTP-01
```

**3. Check the CAA record.** If example.com publishes a CAA record that does not include Let's Encrypt, issuance will be refused. This fails at issuance time with a CAA error, and it is not an obvious error message.

```
dig example.com CAA +short                      # empty output means no restriction; otherwise letsencrypt.org must be listed
```

**4. Open the firewall path.** On pfSense, add the NAT port forward for 443 (and 80, if HTTP-01 validation will be used) to the new host, plus the matching WAN firewall rule. Record it in the NAT table in `Networking Guide/ipam-vlan-topology-reference.md`. Verify from outside the network, not from inside:

```
curl -sv -o /dev/null http://app.example.com/ 2>&1 | head -20   # run from off-network; expect a connection, not a timeout
```

**5. Install the backend and confirm it works before involving Apache.** Diagnosing a proxy is much harder when you do not know whether the backend is healthy.

```
curl -sf http://127.0.0.1:3000/ -o /dev/null && echo BACKEND-OK    # backend answers on its own port
ss -lntp | grep 3000                                               # confirm what is listening and as which process
```

Note whether the backend binds `127.0.0.1` or `0.0.0.0`. It should bind loopback only — if it binds all interfaces, it is reachable directly, bypassing Apache, TLS, and every security header below.

**6. Obtain the certificate.**

For a per-host Let's Encrypt certificate with the Apache plugin (needs the port-80 vhost from step 7 in place, or use `--webroot`):

```
sudo certbot certonly --apache -d app.example.com          # issue only; do not let certbot rewrite the vhost
```

`certonly` is deliberate. Letting certbot modify vhosts produces configuration nobody wrote and nobody reviews. Issue the cert, then reference it from a vhost you control.

Alternative, webroot — works without certbot touching Apache config at all:

```
sudo certbot certonly --webroot -w /var/www/html -d app.example.com
```

For the wildcard, no issuance is needed — point the vhost at the existing files. See the wildcard section.

**7. Write the vhost** using the pattern above, then:

```
sudo apache2ctl configtest                 # syntax check — ALWAYS before reload; "Syntax OK" expected
sudo a2ensite app.example.com.conf          # enable the site
sudo systemctl reload apache2              # graceful reload, existing connections finish
```

Never skip `configtest`. A syntax error on reload leaves Apache down, taking every other vhost on that host with it.

**8. Verify end to end.**

```
curl -sI https://app.example.com/ | head -1                          # expect HTTP/1.1 200 or an expected redirect
curl -sI http://app.example.com/ | grep -i location                  # expect a 301 to https://
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
```

That last command is the one to learn. `-servername` sets SNI, which matters when one IP serves several vhosts; without it you get whichever certificate Apache considers the default, which is frequently the wrong one and produces a confusing "the cert is wrong" conclusion.

Then check the headers and the proxy behaviour:

```
curl -sI https://app.example.com/ | grep -iE 'strict-transport|x-content-type|x-frame|referrer'   # headers present
curl -s https://app.example.com/ | head -40                                                        # the app actually rendered
```

**9. Document.** Add the host to the vhost inventory below, add the NAT entry to the IPAM reference, and add the certificate to the certificate inventory.

### Certificate inventory

| Hostname | Cert type | Issuer | Cert path | Renewal method | Renewal verified | Expiry | Services depending on it |
|---|---|---|---|---|---|---|---|
| [FILL IN] | [FILL IN: LE per-host / wildcard] | [FILL IN] | [FILL IN] | [FILL IN: certbot timer / manual] | [FILL IN: date last confirmed working] | [FILL IN] | [FILL IN] |
| wildcard.example.com | Wildcard | [FILL IN: CA — a `thawteroot.pem` appears alongside it in the MySQL configs] | [FILL IN] | [FILL IN: manual, if commercial] | [FILL IN] | [FILL IN] | MySQL on db01/db02/appdb2/appdb1 per `Databases/`, plus [FILL IN: which Apache vhosts] |
| [FILL IN] | | | | | | | |

### Vhost inventory

| Host | Vhost ServerName | Backend | Backend port | Cert used | Notes |
|---|---|---|---|---|---|
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |
| [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | [FILL IN] | |

### Let's Encrypt issuance and automated renewal

Let's Encrypt certificates are valid for 90 days. Certbot attempts renewal when a certificate is within 30 days of expiry, which gives a 30-day window in which renewal can fail repeatedly without anyone noticing — and then the service breaks on a specific morning with no change having been made. That window is the entire reason the verification below exists.

**Verify the renewal timer is actually running.** Installing certbot is not the same as renewal being scheduled, and a timer that was disabled during some past troubleshooting stays disabled.

systemd (the usual case on Ubuntu/Debian):

```
systemctl list-timers | grep certbot            # is it listed at all, and when does it next run
systemctl is-enabled certbot.timer              # expect: enabled
systemctl is-active certbot.timer               # expect: active
systemctl status certbot.timer                  # last trigger time and next trigger
journalctl -u certbot.service --since "60 days ago" | tail -50   # did past runs succeed
```

cron (older installs, and macOS hosts):

```
cat /etc/cron.d/certbot                         # the packaged cron entry, if present
sudo crontab -l                                 # root's crontab
ls -la /etc/cron.daily/ | grep -i cert          # some packages install here instead
```

If neither a timer nor a cron entry exists, renewal is not automated on that host regardless of what certbot itself reports. That is a finding.

**Test renewal without consuming rate limits.**

```
sudo certbot renew --dry-run                    # full renewal against the staging endpoint; changes nothing
```

A dry run exercises the real validation path — DNS, the firewall, the webroot, the hooks. If it fails, real renewal will fail too. Run it after any change to DNS, the firewall, or the vhosts, and run it on a schedule of its own if nothing else monitors this.

**Check what certbot believes it is managing:**

```
sudo certbot certificates                       # every cert, its domains, expiry, and days remaining
cat /etc/letsencrypt/renewal/app.example.com.conf   # authenticator, webroot path, and hooks for this cert
```

**Make sure the reload hook exists.** A renewed certificate on disk that Apache has not reloaded is still the old certificate in memory. This is the quiet failure: renewal succeeds, monitoring that checks the file passes, and the served certificate expires anyway.

```
sudo certbot renew --deploy-hook "systemctl reload apache2"    # set on the next renewal
```

Better, set it once for every certificate on the host:

```
sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-apache.sh >/dev/null <<'EOF'
#!/bin/sh
systemctl reload apache2
EOF
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-apache.sh   # runs after any successful renewal
```

Scripts in `renewal-hooks/deploy/` run after every successful renewal on that host, so a certificate added later is covered automatically. Verify it is honoured with `certbot renew --dry-run` — the dry run reports hook execution.

**Rate limits.** Let's Encrypt limits certificates per registered domain per week and duplicate certificates per week. A renewal loop that fails and is retried aggressively can exhaust the limit and block legitimate issuance for days. Use `--dry-run` while debugging, never repeated real requests.

### Wildcard versus per-host certificates

The organization has both. They solve different problems and the choice is not arbitrary.

| | `*.example.com` wildcard | Per-host Let's Encrypt |
|---|---|---|
| Covers | any single-label subdomain of example.com | exactly the names listed |
| Does **not** cover | the apex `example.com` itself, or multi-level names like `a.b.example.com` | anything not listed |
| Issuance effort | once, then distributed by hand | automated per host |
| Renewal | [FILL IN: manual if commercial; DNS-01 only if from Let's Encrypt] | automated, HTTP-01 or DNS-01 |
| Blast radius of key compromise | every service using it | one service |
| Works for a host with no inbound port 80 | yes | only via DNS-01 |
| Appears in Certificate Transparency logs as | one entry, hostnames not disclosed | one entry per hostname, publicly enumerable |

Practical guidance:

- **Use per-host Let's Encrypt by default** for anything internet-facing that can complete HTTP-01 validation. It renews itself, the blast radius is one service, and there is no manual step to forget.
- **Use the wildcard** where automation is impractical: services not reachable on port 80 from the internet, non-HTTP services that need TLS (the MySQL instances on db01, db02, appdb2 and appdb1 use it — see `Databases/MySQL Set up/`), appliances where certbot cannot run, and internal-only names with no public DNS.
- **Do not mix them on one vhost.** Pick one and reference it consistently.
- The wildcard's renewal is a **calendar event, not an automated process** if it is a commercial certificate. When it renews, it has to be redistributed to every host and service using it, and every one of those services has to be restarted or reloaded. That list must exist before the renewal, not be reconstructed during it — which is what the certificate inventory table above is for. [FILL IN: complete the list of every host and service using the wildcard, and the wildcard's current expiry date]

**Wildcard from Let's Encrypt** requires DNS-01 validation — HTTP-01 cannot validate a wildcard. That means certbot needs credentials to create a TXT record in the example.com public zone, via a DNS plugin for the provider. [FILL IN: does the external DNS provider have a certbot plugin, and would DNS-01 automation be viable here? It would remove the manual redistribution step entirely.]

### Routine operations

```
sudo apache2ctl configtest                        # validate config; run before every reload
sudo systemctl reload apache2                     # graceful — in-flight requests complete
sudo systemctl restart apache2                    # hard restart — needed after enabling a module
sudo apache2ctl -S                                # every vhost Apache has loaded and which file defined it
sudo apache2ctl -M                                # loaded modules
sudo certbot certificates                         # cert inventory and days remaining
tail -f /var/log/apache2/app.example.com-error_log # live errors for one vhost
```

`apache2ctl -S` is the fastest way to answer "why is this name serving the wrong site" — it shows the resolution order and the default vhost per address.

---

## Troubleshooting

### Symptom: 502 Bad Gateway

Apache reached the backend and did not get a usable response. The fault is almost always behind Apache, not in it.

- **Likely cause**: backend process is down, crashed, listening on a different port than the vhost proxies to, bound to the wrong interface, or too slow.
- **Check, in order**:

```
curl -sf http://127.0.0.1:3000/ -o /dev/null && echo BACKEND-OK    # does the backend answer at all
ss -lntp | grep 3000                                                # is anything listening, and on which address
systemctl status <backend-service>                                  # is the service running
tail -50 /var/log/apache2/app.example.com-error_log                  # Apache's own account of the failure
journalctl -u <backend-service> --since "1 hour ago"                # the backend's account
```

- **Reading the Apache error**: `AH00957: HTTP: attempt to connect to 127.0.0.1:3000 (*) failed` means nothing is listening — backend down or wrong port. `AH01114: HTTP: failed to make connection to backend` with a timeout means the backend accepted and then stalled — the backend is alive but hung, often on a database. `AH01102: error reading status line from remote server` means the backend closed the connection mid-response, frequently a backend crash on that specific request.
- **Check the backend's bind address specifically.** A backend bound to `0.0.0.0` works; one bound to `::1` when Apache proxies to `127.0.0.1` does not, and the error looks identical to the process being down. Prefer `ProxyPass ... http://127.0.0.1:3000/` with the backend explicitly on IPv4 loopback, or use `localhost` consistently on both sides.
- **If the backend is on another host**, the failure may be network or firewall rather than application. Test from the Apache host, not from your workstation:

```
curl -sv http://<backend-host>:3000/ -o /dev/null      # from the proxy host itself
```

- **Fix**: restart the backend, correct the port in the vhost, or correct the bind address. There is prior art for this failure mode in this environment: `Dynamics GP & DMS/Apache has gone down on dms.example.com – Systems Knowledge Base.pdf` and `Dynamics GP & DMS/DMSWeb Apache crash solution.txt`.

### Symptom: 503 Service Unavailable from Apache

Different from a 502 and diagnosed differently. If `mod_proxy` has marked a balancer member as failed, it returns 503 without attempting the backend at all — so the backend can be healthy and the error persists.

- **Check**: `tail /var/log/apache2/error_log` for `AH01144: No protocol handler was valid` (a missing module — usually `proxy_http` not enabled) or balancer member errors.
- **Fix**: `sudo a2enmod proxy_http && sudo systemctl restart apache2`, or reset the balancer member.

### Symptom: certificate chain error — "unable to verify the first certificate", "incomplete chain"

The leaf certificate is being served without the intermediate. Browsers often paper over this using a cached intermediate, so it works for you and fails for a user, a mobile client, or `curl`. That asymmetry is the tell.

- **Likely cause**: `SSLCertificateFile` points at `cert.pem` (leaf only) instead of `fullchain.pem` (leaf plus intermediates).
- **Check what is actually being served**:

```
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null 2>/dev/null | grep -A2 'Certificate chain'
```

A correct chain shows the leaf at depth 0 and at least one intermediate at depth 1. A chain with only depth 0 is the fault.

```
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null 2>&1 | grep -i 'verify'
```

Expect `Verify return code: 0 (ok)`. Anything else names the problem.

- **Fix**: point `SSLCertificateFile` at `/etc/letsencrypt/live/<name>/fullchain.pem`, then `apache2ctl configtest` and reload.

### Symptom: wrong certificate served for a hostname

- **Likely cause**: no vhost matches the requested `ServerName`, so Apache serves the default (first-loaded) vhost for that address, with its certificate. Or the client is not sending SNI.
- **Check**:

```
sudo apache2ctl -S                                              # which vhost is default for :443, and which file defines each
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -ext subjectAltName
```

Compare the SAN list against the name being requested. Note that for a wildcard, `*.example.com` matches `app.example.com` but **not** `example.com` and **not** `a.b.example.com`.

- **Fix**: correct the `ServerName`/`ServerAlias`, or add the missing vhost.

### Symptom: certificate expired

- **Immediate check**:

```
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null 2>/dev/null | openssl x509 -noout -dates
sudo certbot certificates                                        # what certbot thinks the expiry is
```

If certbot reports a valid future expiry but the server serves an expired one, **the certificate renewed and Apache was never reloaded**. Fix immediately with `sudo systemctl reload apache2`, then fix the cause by installing the deploy hook described above. This specific failure — valid file on disk, stale certificate in memory — is the most common way a fully automated renewal setup still produces an outage.

- **If certbot also reports it expired**, renewal has been failing. Go to the next section.

### Symptom: renewal failing silently

This is the failure mode worth designing against, because nothing visible happens until the service breaks.

- **Diagnose**:

```
sudo certbot renew --dry-run                                     # reproduces the real failure safely
journalctl -u certbot.service --since "60 days ago"              # history of attempts
sudo cat /var/log/letsencrypt/letsencrypt.log | tail -100        # detailed error from the last attempt
```

- **Common causes, in rough order of frequency**:

| Cause | How it shows | Fix |
|---|---|---|
| Port 80 no longer reachable from the internet | HTTP-01 challenge timeout | Restore the NAT/firewall rule for 80, or switch that cert to DNS-01 |
| The port-80 vhost redirects the ACME path to HTTPS | Challenge returns a redirect instead of the token | Exclude `/.well-known/acme-challenge/` from the redirect, as in the pattern above |
| Webroot path changed | "could not write challenge file" or 404 on the token | Correct `webroot_path` in `/etc/letsencrypt/renewal/<name>.conf` |
| Public DNS record removed or changed | Validation resolves to the wrong host | Restore the record; check both views |
| CAA record added or changed | Explicit CAA error at issuance | Add the Let's Encrypt identifier to the CAA record |
| The timer was disabled and never re-enabled | No attempts in the journal at all | `systemctl enable --now certbot.timer` |
| Rate limit hit after repeated failures | "too many certificates already issued" | Stop retrying, fix the underlying cause, wait out the window |
| Host is behind a proxy/WAF that intercepts `/.well-known/` | Challenge returns the wrong content | Exempt the ACME path at the proxy |

- **Detecting it before expiry** — this is the part that actually prevents the outage. Pick at least one:

```
# Expiry check against the SERVED certificate, not the file on disk.
# Exits nonzero if the served cert expires within 21 days. Suitable for cron + alert.
echo | openssl s_client -connect app.example.com:443 -servername app.example.com 2>/dev/null \
  | openssl x509 -noout -checkend 1814400 || echo "ALERT: app.example.com cert expires within 21 days"
```

```
# All certs on this host, days remaining, from certbot's own view:
sudo certbot certificates | grep -E 'Certificate Name|Expiry Date'
```

Checking the **served** certificate rather than the file is the important distinction — it catches the renewed-but-not-reloaded case that a file check misses. Run it against every hostname in the certificate inventory, from a host that is not the one serving it, and alert on failure. [FILL IN: is there an existing monitoring system that can run this check and alert — Graylog with a scripted input, an uptime checker, or a cron job on a monitoring host?]

Also worth doing: ensure Let's Encrypt's own expiry-warning emails go somewhere a human reads. They are sent to the ACME account email. [FILL IN: what email address is registered on the certbot ACME account on each host? `sudo certbot show_account` reports it.]

### Symptom: infinite redirect loop

- **Likely cause**: the backend does not know the request arrived over HTTPS, so it redirects to `http://`, which the port-80 vhost redirects back to `https://`.
- **Check**:

```
curl -sIL https://app.example.com/ | grep -iE '^(HTTP|location)'   # watch the redirect chain; a http↔https cycle confirms it
```

- **Fix**: set `RequestHeader set X-Forwarded-Proto "https"` in the 443 vhost, and configure the application to trust it. Some applications also need an explicit base-URL setting.

### Symptom: Apache will not start after a config change

- **Check**: `sudo apache2ctl configtest` names the file and line. `journalctl -xeu apache2` gives the startup failure.
- **Common causes**: a certificate path pointing at a file that does not exist (typo, or a cert that was deleted); a module used in the vhost that is not enabled; a duplicate `ServerName` across two vhosts; a port already in use by another service.
- **Fix**: correct and retest with `configtest` before reloading. If Apache is currently down and every vhost on the host is affected, disabling the offending site (`a2dissite`) restores the rest immediately while you work on it.

### Symptom: works internally, fails externally (or the reverse)

- **Likely cause**: split-horizon DNS. The two views resolve the name to different addresses, and only one path has a working vhost, certificate, or NAT rule.
- **Check**: resolve from both views and compare — `nslookup app.example.com` internally against `dig @1.1.1.1 app.example.com +short` externally. See `Networking Guide/dns-dhcp-administration-guide.md`.
- **Fix**: whichever view is wrong. Note that an internal-only name cannot complete HTTP-01 validation, because Let's Encrypt validates from the internet.

---

## Security

### HTTP security header baseline

Set these on every HTTPS vhost. `always` matters — without it the header is omitted on error responses, which is where it is often most needed.

```apache
# Force HTTPS for a year, including subdomains. Only add once HTTPS is known
# to work everywhere on the domain — a browser that has seen HSTS will refuse
# to fall back to HTTP, and the header cannot be un-sent from a client's cache.
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"

# Stop browsers from guessing content types (defeats a class of upload attack).
Header always set X-Content-Type-Options "nosniff"

# Block framing by other origins (clickjacking).
Header always set X-Frame-Options "SAMEORIGIN"

# Limit what referrer information leaks to other sites.
Header always set Referrer-Policy "strict-origin-when-cross-origin"

# Deny browser features the application does not use.
Header always set Permissions-Policy "geolocation=(), microphone=(), camera=()"

# Content-Security-Policy is the highest-value header and the one most likely
# to break an application. Start in report-only, watch the reports, then enforce.
# Header always set Content-Security-Policy-Report-Only "default-src 'self'"

# Do not advertise the exact Apache version and OS.
ServerTokens Prod
ServerSignature Off
```

Notes on the ones with sharp edges:

- **HSTS with `includeSubDomains`** applies to every subdomain of the name it is served on. Setting it on `example.com` itself commits every `*.example.com` service to HTTPS in every browser that has seen it, for the max-age duration. Do not set it at the apex until every subdomain does HTTPS correctly. Do not add `preload` unless the commitment is understood — removal from the preload list takes months.
- **CSP** will break applications that use inline scripts or third-party resources. Report-only first, always.
- **`ServerTokens Prod`** goes in the main config, not per-vhost.

Verify the headers are actually being sent:

```
curl -sI https://app.example.com/ | grep -iE 'strict-transport|x-content-type|x-frame|referrer|permissions|^server'
```

### TLS configuration baseline

```apache
SSLProtocol             -all +TLSv1.2 +TLSv1.3       # disable everything older
SSLHonorCipherOrder     off                           # let the client pick from the allowed set (modern practice)
SSLSessionTickets       off
SSLUseStapling          on                            # OCSP stapling — faster and more private for clients
SSLStaplingCache        "shmcb:logs/ssl_stapling(32768)"
```

`SSLProtocol` and the stapling cache are server-wide and belong in `/etc/apache2/mods-available/ssl.conf` rather than repeated per vhost.

Check what is negotiated:

```
openssl s_client -connect app.example.com:443 -servername app.example.com </dev/null 2>/dev/null | grep -E 'Protocol|Cipher'
```

### General

- **Exposure**: only ports 80 and 443 should be forwarded to these hosts from the internet. Backend ports must not be exposed — confirm with `ss -lntp` that backends bind loopback, and confirm in the pfSense NAT table that no forward points at a backend port.
- **Auth**: authentication is normally the application's responsibility, not Apache's. Where Apache does the authenticating, record how. [FILL IN: are any vhosts protected by Apache-level auth, and against what backend?]
- **Certificates**: see the certificate inventory. Private keys in `/etc/letsencrypt/` must be `root`-owned and mode 600 on the key files; `/etc/letsencrypt/archive` and `/live` should not be world-readable.
- **Secrets**: DNS provider API tokens used for DNS-01 validation are credentials with the power to alter the example.com zone. Store in 1Password, deploy with restrictive permissions, and never in a vhost file.
- **fail2ban** runs on the edge and on mail.example.com; Snort runs on pfSense. A blocked source IP presents as a timeout, not an error — check these before concluding a service is down. Unban procedures are in `Security & Hardening/`.

## Monitoring & Alerting

- **Certificate expiry** — the served-certificate check above, run against every hostname in the inventory, alerting at 21 days. This is the single highest-value check in this document.
- **Renewal success** — alert if `certbot.service` has not run successfully within [FILL IN: threshold, suggest 14 days] on any host.
- **Apache availability** — an HTTP check per vhost expecting a 200. [FILL IN: does an uptime check exist, and where do its alerts go?]
- **Error rate** — [FILL IN: are Apache error logs shipped to graylog01.example.com? A 502 rate spike is worth alerting on]
- **Baseline**: [FILL IN: normal request rate and error rate per vhost]

## Disaster Recovery

- **What to back up**: the whole of `/etc/letsencrypt/` (account keys, certificates, and renewal configs — without the account key, certbot cannot renew), `/etc/apache2/sites-available/`, and the enabled-site symlinks. [FILL IN: are these included in the existing backup job? See `Hardware & Backup/` and `SysAdmin Procedures/Backup_DR_Runbook.txt`]
- **Wildcard certificate and key**: must be recoverable independently of any one host, since re-issuing a commercial wildcard is not a same-day operation. [FILL IN: where is the authoritative copy of the wildcard cert and key held?]
- **Rebuild order** for one host: restore `/etc/letsencrypt/` → install Apache and enable modules → restore vhosts → `configtest` → start → verify with the end-to-end checks above.
- **If `/etc/letsencrypt/` is lost entirely**: certificates can be re-issued from scratch provided DNS and port 80 are working, but rate limits apply — do not attempt to re-issue everything at once.
- **RTO/RPO**: [FILL IN: agreed targets]

## Decisions & History (ADR-lite)

| Date       | Decision / Change                                                      | Why / Ticket |
|------------|------------------------------------------------------------------------|--------------|
| 2026-09-11 | This guide created                                                      | No documented standard for vhost pattern or renewal verification |
| [FILL IN]  | [FILL IN: when the wildcard was obtained, from which CA, and why a wildcard rather than per-host] | [FILL IN] |
| [FILL IN]  | [FILL IN: when Let's Encrypt was adopted and on which hosts]            | [FILL IN] |

## References

- `Linux & Servers/Apache Virtual Host Setup.pdf` — the original vhost pattern (macOS Apache, port 80, document-root serving); source of the per-vhost log naming convention used above
- `Networking Guide/dns-dhcp-administration-guide.md` — split-horizon DNS, CAA records, record change and verification procedure
- `Networking Guide/ipam-vlan-topology-reference.md` — NAT and port-forward inventory that these services depend on
- `Databases/MySQL Set up/` — wildcard certificate deployment for MySQL TLS (`/etc/mysql/wildcardcert.crt`, `wildcardkey.pem`, `thawteroot.pem`) on db01, db02, appdb2, appdb1
- `Databases/mysql certificate rotation.txt` — existing certificate rotation procedure for the database side
- `Security & Hardening/LDAPs certificate replacement procedure.txt` — related certificate replacement work
- `Dynamics GP & DMS/Apache has gone down on dms.example.com – Systems Knowledge Base.pdf` and `Dynamics GP & DMS/DMSWeb Apache crash solution.txt` — prior Apache backend failures in this environment
- `Linux & Servers/phpbb update.txt` — forums.example.com application maintenance
- Upstream: Apache `mod_proxy` and `mod_ssl` documentation; Let's Encrypt / certbot documentation

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

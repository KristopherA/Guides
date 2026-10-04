# Wireless, RADIUS and 802.1X Administration (example.com)

> Status: initial draft, 2026-10-04. Contains [FILL IN] markers — see INDEX.md.

## Overview

This guide covers staff wireless authentication end to end: the access point a laptop associates to, the controller that configures that AP, the RADIUS server the AP asks "is this person allowed on?", and the Active Directory account that ultimately answers the question. It also covers the guest network, the server certificate that the whole system depends on, how client devices are told to trust that certificate, and how to test, monitor and rebuild all of it.

It replaces the Aerohive-era KB article (marked "Deprecated — Needs to be rewritten for the TP-Link APs") and pulls together the Omada RADIUS note and the NPS certificate screenshot (all listed in References). It does **not** repeat client-side macOS Wi-Fi troubleshooting or wireless security incident response — those live in `Networking Guide/Wifi Guide/` and are linked where they apply.

**The single most important fact in this guide:** staff Wi-Fi uses PEAP, and PEAP depends on one server certificate (`radius.example.com`). When that certificate expires or is replaced carelessly, *every* staff wireless client fails at the same moment. Read "Server certificate" before touching anything certificate-related.

Audience: IT Ops staff with Domain Admin (or delegated NPS admin) rights on the RADIUS server, and admin access to the Omada and UniFi controllers.

## Quick Facts

| Field            | Value |
|------------------|-------|
| Owner            | IT lead (it@example.com), IT Ops |
| Environment      | prod |
| RADIUS server    | **Windows Network Policy Server (NPS)** — established from the KB article and the console screenshot (NPS console, policies `ORG_Wireless` / `UbiVPN`). Not FreeRADIUS. Current target: `auth4.example.com` (10.60.1.145, also a domain controller). Previous target: `auth.example.com` — [FILL IN: is auth.example.com decommissioned, and is NPS still installed/listening on it?] |
| Secondary RADIUS | [CONFIRM: the IPAM reference lists auth2.example.com (10.60.1.221) as an "802.1x backend". Confirm whether NPS is installed on auth2 with a synchronized config and whether the controllers list it as a secondary server. If not, there is no RADIUS redundancy] |
| Wireless controllers | TP-Link Omada at `omada.example.com` (main office site; APs are TP-Link EAPs). Ubiquiti UniFi controller also in use per IPAM reference — [FILL IN: UniFi controller host/URL and which site(s) or SSIDs it serves, e.g. the partner site] |
| Staff SSID       | "ExampleCorp-WiFi" in the Omada WLAN list (NPS connection request policy is named `ORG_Wireless`) — [CONFIRM: exact broadcast SSID string, including space vs underscore] |
| Auth method      | WPA2-Enterprise, PEAP with EAP-MSCHAPv2 against AD user accounts — [CONFIRM: whether the WLAN is WPA2-only or WPA2/WPA3 mixed] |
| Authorised users | AD group "Office Group" per KB article — [CONFIRM: exact AD group name/DN used in the NPS network policy] |
| EAP server cert  | Subject `radius.example.com`, issued by Thawte RSA CA 2018 (DigiCert), from digicert.com in .pfx form. Screenshot (2024-02-12) shows expiry 2024-12-21; the server desktop has a "radius.example.com 2025 certs" folder. [FILL IN: current certificate's expiry date and thumbprint] |
| Guest SSID       | [FILL IN: guest SSID name(s), controller, PSK or captive portal]. Subnets: PO Guest 192.168.20.0/24, Partner Guest 10.0.1.0/24 (from `[internal KB: network subnet list]`) |
| Secrets          | RADIUS shared secret(s), controller admin credentials, guest PSK and the PFX password live in 1Password / IT vault. **Never in this document.** |
| Access           | NPS: RDP to auth4.example.com from an admin workstation, `nps.msc`. Omada: https://omada.example.com, select the main office in the top-left site menu. UniFi: [FILL IN] |
| Dependencies     | AD (auth2/auth4), DNS, NTP (cert validity and Kerberos), pfSense rules allowing AP management subnet → NPS UDP 1812/1813, the public CA (DigiCert) for renewals, PoE switching for the APs |
| Dependents       | All staff wireless; the `UbiVPN` connection request policy on the same NPS (a VPN authenticates through this server too — [FILL IN: which VPN uses UbiVPN and whether it is still live]); wired 802.1x if/when enabled on EdgeSwitch ports |
| Last reviewed    | 2026-10-04 (new draft — UNVERIFIED) |

## How It Works

### The authentication path

```
Laptop / phone (supplicant)
   │  802.11 association, then EAPOL (EAP over LAN) — no IP address yet
   ▼
TP-Link EAP access point  (the "authenticator" / RADIUS client / NAS)
   │  RADIUS Access-Request, UDP 1812, from the AP's own management IP,
   │  signed with the shared secret
   ▼
NPS on auth4.example.com (10.60.1.145)  (the "authentication server")
   │  1. Connection Request Policy "ORG_Wireless" matches (NAS-Port-Type = Wireless)
   │  2. PEAP TLS tunnel built using the radius.example.com certificate
   │  3. Inside the tunnel: EAP-MSCHAPv2 username/password
   │  4. Network Policy checks group membership ("Office Group")
   ▼
Active Directory (the NPS host is itself a DC; user lookup + password check via Netlogon)
   │
   ▼
Access-Accept (+ keying material) → AP → 4-way handshake → client gets an IP on the staff VLAN via DHCP
```

The controller (Omada or UniFi) is **not** in the authentication path. It only pushes configuration — SSID, security mode, RADIUS profile (server IP, port, shared secret) — down to the APs. Each AP talks to NPS directly from its own management IP. Two consequences follow:

- NPS must have a RADIUS client entry for every AP (or for the AP management subnet), not for the controller.
- If the controller goes down, staff Wi-Fi keeps working. If NPS goes down, it does not, no matter how healthy the controller looks.

### Where things live

| Component | Where | Notes |
|-----------|-------|-------|
| NPS configuration (clients, policies) | auth4.example.com, `nps.msc`; stored locally on that server (`%SystemRoot%\System32\ias\ias.xml`) | Not replicated by AD. A second NPS server must be kept in sync by export/import |
| EAP server certificate | auth4.example.com, `Cert:\LocalMachine\My` (Local Computer → Personal) | Selected by name+expiry inside the PEAP settings of the `ORG_Wireless` CRP |
| Authentication results | auth4.example.com Security event log, events 6272/6273 (and others below) | Requires the "Network Policy Server" audit subcategory enabled |
| NPS accounting log | `%SystemRoot%\System32\LogFiles\IN*.log` by default — [CONFIRM: accounting configured to text file, SQL, or not at all] | Text-format accounting, useful for session history |
| SSID / RADIUS profile | Omada: Settings → Wireless Networks → WLAN, and the RADIUS Profile under Settings → Profiles (the older note says Network Profile → RADIUS Profile; the menu name differs by controller version). UniFi: Settings → WiFi, and Settings → Profiles → RADIUS | |

### NPS policy structure in this environment (from the 2024-02-12 screenshot)

| Policy type | Name | Order | What it does |
|-------------|------|-------|--------------|
| Connection Request Policy | `ORG_Wireless` | 1 | Wireless requests; **overrides network policy auth settings** with PEAP/EAP-MSCHAPv2 — so the server certificate is selected here. Full settings in Setup step 5 |
| Connection Request Policy | `UbiVPN` | 2 | VPN authentication — out of scope here but shares the server; do not delete |
| Connection Request Policy | Use Windows authentication for all users | 1000000 | Default catch-all |
| Network Policy | [FILL IN: name of the network policy that grants wireless access] | [FILL IN] | Grants access to members of "Office Group" — [CONFIRM] |

### PEAP-MSCHAPv2 vs EAP-TLS

| | PEAP-MSCHAPv2 (current) | EAP-TLS (possible future) |
|---|---|---|
| User proves identity with | AD username and password | A client certificate on the device |
| What clients need | Trust in the server cert's root and the server name `radius.example.com` | A client cert (AD CS autoenrolment on Windows, MDM/SCEP on Mac) **and** server trust |
| Weak point | If a client does not validate the server certificate, an evil-twin AP can capture an MSCHAPv2 exchange, which is crackable offline. Also: Windows 11 Credential Guard blocks MSCHAPv2 single sign-on | Certificate lifecycle on every device; needs an internal CA and a device-management channel |
| Password change effect | Saved credentials go stale after an AD password change → repeated failures and lockouts | None |
| Effort in this environment | Already running | [CONFIRM: proposal — evaluate EAP-TLS once an MDM is in place; see `Windows & Mac Workstations/mdm-and-apple-business-manager-guide.md`. Not worth attempting before then] |

WPA3-Enterprise is the same EAP exchange with stronger management-frame protection (PMF required). It does not change anything on NPS. WPA3-Enterprise "192-bit mode" requires EAP-TLS and is not applicable while PEAP is in use.

---

## Setup / Installation

These steps rebuild the RADIUS side from nothing on a Windows Server. They also serve as the procedure for standing up a secondary NPS server.

### 1. Install and register NPS

```powershell
Install-WindowsFeature NPAS -IncludeManagementTools          # installs NPS role + nps.msc console
netsh nps show config                                         # sanity check the service answers
netsh ras add registeredserver                                # registers NPS in AD (adds computer to "RAS and IAS Servers") so it can read dial-in properties
Get-ADGroupMember "RAS and IAS Servers" | Select-Object Name  # confirm the NPS host(s) are members
Get-Service IAS                                               # NPS service; should be Running / Automatic
```

### 2. Enable auditing so authentication results are logged

```powershell
auditpol /get /subcategory:"Network Policy Server"                                   # current state
auditpol /set /subcategory:"Network Policy Server" /success:enable /failure:enable   # required for events 6272/6273
```

[CONFIRM: if this is controlled by a Default Domain Controllers GPO, set it there instead, or the GPO will overwrite the local setting at next refresh.]

### 3. Import the EAP server certificate

Obtain the certificate as a .pfx (certificate + private key) — see "Server certificate" below for renewal.

```powershell
$pfxPass = Read-Host -AsSecureString "PFX password (from 1Password)"                              # never type it on the command line
Import-PfxCertificate -FilePath C:\certs\radius.example.com.pfx -CertStoreLocation Cert:\LocalMachine\My -Password $pfxPass   # Local Computer\Personal
Get-ChildItem Cert:\LocalMachine\My | Where-Object Subject -like "*radius.example.com*" | Format-List Subject,Issuer,NotAfter,Thumbprint,HasPrivateKey   # HasPrivateKey must be True
```

GUI equivalents: IIS Manager → Server Certificates → Import (the KB's method) or `certlm.msc` → Personal → Import.

**DC caution:** auth4 is a domain controller. A DC with no certificate in the NTDS service store will pick any Server Authentication certificate in Local Computer\Personal for LDAPS — which could be the RADIUS certificate. Confirm LDAPS is pinned to its own certificate in the NTDS store (see `Security & Hardening/LDAPs certificate replacement procedure.txt` and `Security & Hardening/certificate-pki-lifecycle-guide.md`) before adding or removing certificates in Personal on auth2/auth4.

### 4. Create the RADIUS clients (one per AP)

The KB's established practice is one RADIUS client entry per AP, for ease of troubleshooting. GUI: NPS → RADIUS Clients and Servers → RADIUS Clients → New.

```powershell
Get-NpsRadiusClient | Format-Table Name,Address,Enabled                                            # current list
$sec = [Runtime.InteropServices.Marshal]::PtrToStringAuto([Runtime.InteropServices.Marshal]::SecureStringToBSTR((Read-Host -AsSecureString "Shared secret from 1Password")))   # read secret without echo
New-NpsRadiusClient -Name "omada-ap-<location>" -Address "<AP management IP>" -SharedSecret $sec    # one per AP
Remove-Variable sec                                                                                # do not leave it in the session
```

| Setting | Value | Reason |
|---------|-------|--------|
| Friendly name | `omada-ap-<location>` / `unifi-ap-<location>` | Appears as ClientName in every 6272/6273 event, which makes failures traceable to a physical AP |
| Address | AP management IP — [FILL IN: AP management VLAN/subnet from the IPAM reference] | The AP, not the controller, sends RADIUS |
| Vendor | RADIUS Standard | No vendor-specific attributes needed for PEAP |
| Shared secret | Long random value, stored in 1Password / IT vault | [CONFIRM: proposal — one secret per controller site rather than one fleet-wide value, per `Security & Hardening/secrets-management-guide.md`. The Omada RADIUS profile applies one secret to every AP in the site, so per-AP secrets are not practical there] |
| "Access-Request messages must contain the Message-Authenticator attribute" (Advanced tab) | Enabled | Mitigates the 2024 BlastRADIUS (CVE-2024-3596) forgery attack. [CONFIRM: test with one AP first — TP-Link and Ubiquiti APs on current firmware send the attribute with EAP requests, but verify before enabling on all clients] |

If there are many APs, an AP-management subnet can be entered as a single client (address in CIDR form, e.g. `<subnet>/24`). [CONFIRM: proposal — move to a single subnet entry per site only once every AP has a reserved management IP; until then keep per-AP entries as the KB recommends.]

### 5. Connection Request Policy `ORG_Wireless`

| Setting | Value | Reason |
|---------|-------|--------|
| Processing order | 1 | Must be evaluated before the catch-all and before `UbiVPN` |
| Condition | NAS Port Type = Wireless - Other OR Wireless - IEEE 802.11 | Scopes the policy to wireless so VPN and future wired 802.1x get their own policies |
| Authentication | Authenticate requests on this server | No proxying |
| Override network policy authentication settings | Enabled | Existing design: EAP settings, including the certificate, live here. When replacing the cert, change it **here** |
| EAP type | Microsoft: Protected EAP (PEAP) → Secured password (EAP-MSCHAP v2) | Username/password against AD |
| Certificate issued to | radius.example.com (current one, check expiry date in the dialog) | The certificate clients validate |
| Enable Fast Reconnect | Enabled | Faster roaming between APs; cached TLS session |
| Disconnect clients without cryptobinding | Disabled — [CONFIRM: proposal — enable once all client OSes are confirmed to support PEAP cryptobinding; it defeats some tunnel-relay attacks] | Older clients may fail |
| Less secure methods: MS-CHAP (v1) | Currently ticked — [CONFIRM: proposal — untick. PEAP does not use it, and MS-CHAPv1 is obsolete] | Reduce attack surface |

### 6. Network Policy (grants access)

| Setting | Value | Reason |
|---------|-------|--------|
| Condition: Windows Groups | EXAMPLE\Office Group — [CONFIRM exact name] | Only staff accounts get on the staff SSID |
| Condition: NAS Port Type | Wireless - IEEE 802.11, Wireless - Other | Keeps this policy from granting VPN or wired access by accident |
| Access permission | Grant access | |
| Ignore user account dial-in properties | [CONFIRM: proposal — enable, so a user's legacy "Deny access" on the Dial-in tab cannot silently block Wi-Fi (reason code 65)] | Fewer surprise rejects |
| Authentication methods | PEAP / EAP-MSCHAPv2 (overridden by the CRP anyway) | Kept consistent so the policy still works if override is ever disabled |
| RADIUS attributes for dynamic VLAN | Not used — [CONFIRM]. If ever needed: Tunnel-Type = VLAN, Tunnel-Medium-Type = 802, Tunnel-Pvt-Group-ID = VLAN ID, and "RADIUS-assigned VLAN" enabled on the SSID | Static SSID→VLAN mapping is simpler and is what the IPAM reference documents |

### 7. Firewall

NPS listens on UDP 1812 (authentication) and 1813 (accounting).

```powershell
Get-NetUDPEndpoint -LocalPort 1812,1813                                               # confirm IAS is bound
Get-NetFirewallRule -DisplayGroup "Network Policy Server" | Format-Table DisplayName,Enabled,Direction   # Windows firewall rules created by the role
```

If the AP management subnet and 10.60.0.0/16 are on different pfSense interfaces, pfSense needs a rule permitting AP management subnet → 10.60.1.145 UDP 1812-1813 (and to auth2 if it is a secondary). Follow `Networking Guide/firewall-change-procedure.md`. [FILL IN: which pfSense rule currently carries this traffic, and whether partner-site APs reach auth4 across the IPsec tunnel or use a local server such as auth3.example.com (192.168.2.222)].

### 8. Controller side — Omada (from `Change Radius for WIfi.txt`)

1. Log in to omada.example.com and select the **main office** in the site menu, top left.
2. Create a RADIUS profile: Settings → Profiles (older versions: Network Profile) → RADIUS Profile → Create. Enter the NPS IP (10.60.1.145), auth port 1812, and the shared secret from 1Password. Add the secondary server here if one exists.
3. Settings → Wireless Networks → WLAN → edit **ExampleCorp-WiFi** → Security: WPA-Enterprise → RADIUS Profile: select the new profile. (The 2025 change flipped this from auth.example.com to auth4.example.com.)
4. Apply. APs re-provision within about a minute; already-connected clients usually keep their session until they roam or reconnect.

| Setting | Value | Reason |
|---------|-------|--------|
| Security | WPA-Enterprise, WPA2 (or WPA2/WPA3) — [CONFIRM current] | 802.1X with per-user credentials |
| Encryption | AES (CCMP) | TKIP is deprecated and breaks 802.11n/ac/ax rates |
| PMF (802.11w) | Capable/optional with WPA2/WPA3 mixed; required with WPA3-only — [CONFIRM] | Protection against deauthentication attacks without locking out older clients |
| RADIUS accounting | [CONFIRM: proposal — enable, pointing to the same NPS on 1813] | Session history in NPS accounting log |
| VLAN | Staff VLAN — [FILL IN: VLAN ID; PO Staff is 192.168.30.0/24 per `[internal KB: network subnet list]`] | Matches IPAM reference |
| NAS ID | [FILL IN or leave default] | Can be used as an NPS condition if sites ever need different policies |

### 9. Controller side — UniFi

Same model: Settings → Profiles → RADIUS → create a profile (server, port 1812, shared secret), then Settings → WiFi → the SSID → Security Protocol WPA2/WPA3 Enterprise → RADIUS profile. Each UniFi AP is a RADIUS client in NPS. Menu names moved in UniFi Network 9.x; see `Networking Guide/Network-Administration-Guide.md` §8. [FILL IN: which SSIDs, if any, are served by UniFi and whether they use NPS.]

---

## Configuration

### SSID and VLAN mapping

The authoritative table is the SSID-to-VLAN table in `Networking Guide/ipam-vlan-topology-reference.md`. Do not duplicate values here; update that table when anything changes. Known subnets from `[internal KB: network subnet list]`:

| Network | Subnet | Expected SSID / auth | Notes |
|---------|--------|---------------------|-------|
| PO Staff | 192.168.30.0/24 | ExampleCorp-WiFi — 802.1X via NPS | [CONFIRM mapping and VLAN ID] |
| PO Guest | 192.168.20.0/24 | Guest — [FILL IN: PSK or portal] | Must not reach internal VLANs |
| Partner Staff | 192.168.150.0/24 | [FILL IN: partner staff SSID, controller, RADIUS server] | |
| Partner Guest | 10.0.1.0/24 | [FILL IN] | |
| Servers and Systems | 10.60.0.0/16 | none (wired) | Hosts auth2/auth4 (NPS) |

### Client trust and profile deployment

A client is only as safe as its server validation. A PEAP client that accepts any certificate will hand an MSCHAPv2 exchange to any AP broadcasting "ExampleCorp-WiFi". The goal is that every managed device is told in advance: trust this root, and only for server name `radius.example.com`.

#### Windows (domain-joined) — Wireless GPO

GPO path: Computer Configuration → Policies → Windows Settings → Security Settings → Wireless Network (IEEE 802.11) Policies → Create a new Wireless Network Policy for Windows Vista and later releases. [FILL IN: name of the existing Wi-Fi GPO and the OU it is linked to, if one exists.]

| Setting | Value | Reason |
|---------|-------|--------|
| SSID | ExampleCorp-WiFi [CONFIRM exact] | |
| Authentication / Encryption | WPA2-Enterprise / AES (or WPA3-Enterprise if the WLAN is WPA3) | Must match the WLAN exactly or Windows ignores the profile |
| Network authentication method | Microsoft: Protected EAP (PEAP) | |
| Verify the server's identity by validating the certificate | Enabled | The control that stops evil-twin capture |
| Connect to these servers | `radius.example.com` | Name pinning; matches the cert subject |
| Trusted Root Certification Authorities | The root that issues the current radius.example.com chain — [CONFIRM: inspect the current cert's chain; Thawte RSA CA 2018 chains to a DigiCert root, and DigiCert is moving issuance to its G2/G3 roots, so the root may change at next renewal] | If the root changes, clients must be updated **before** the cert swap |
| Notifications before connecting | Don't ask user to authorize new servers or trusted CAs | Users cannot be talked into trusting a rogue server |
| Inner method | Secured password (EAP-MSCHAP v2) | |
| Automatically use my Windows logon name and password | [CONFIRM: currently on/off]. Note: Windows 11 22H2+ with Credential Guard (on by default on many domain-joined Enterprise/Education devices) blocks MSCHAPv2 single sign-on, so this will fail silently on those machines | Plan for user prompts, or EAP-TLS |
| Authentication mode | User authentication — [CONFIRM: or "User or computer" if machines should be on Wi-Fi at the login screen; computer auth also needs Domain Computers in the network policy] | |

Verification on a client:

```powershell
gpresult /r /scope computer | Select-String -Pattern "Wireless|Wi-Fi"   # is the Wi-Fi GPO applied
gpupdate /force                                                        # pull policy now
netsh wlan show profiles                                               # Group Policy profiles are listed separately from user profiles
netsh wlan show interfaces                                             # current SSID, BSSID, auth, signal
netsh wlan show wlanreport                                              # HTML report in C:\ProgramData\Microsoft\Windows\WlanReport\
Get-WinEvent -LogName "Microsoft-Windows-WLAN-AutoConfig/Operational" -MaxEvents 30 | Format-Table TimeCreated,Id,Message -Wrap   # client-side connect/fail reasons
```

#### macOS — configuration profile

A Wi-Fi payload with: SSID, Security type WPA2/WPA3 Enterprise, accepted EAP type PEAP, a Certificate payload carrying the root (and intermediate) CA, and in the Wi-Fi payload's Trust section the certificate selected plus Trusted Server Certificate Names = `radius.example.com`. Build it with Apple Configurator or iMazing Profile Editor. [FILL IN: path of the current staff Wi-Fi .mobileconfig, if one exists.]

Deployment: on macOS 11 and later, configuration profiles can only be installed silently by an MDM; without one, the user double-clicks the profile and approves it in System Settings → General → Device Management (Profiles on older releases). Munki can offer the profile but the user still approves it — [CONFIRM: current method]. MDM status is covered in `Windows & Mac Workstations/mdm-and-apple-business-manager-guide.md`.

```zsh
sudo profiles show -type configuration                     # installed profiles; look for the Wi-Fi payload
networksetup -removepreferredwirelessnetwork en0 "ExampleCorp-WiFi"   # clear a stale user-created entry that conflicts with the profile
log stream --predicate 'process == "eapolclient"'          # watch the EAP exchange while reproducing
```

Without a profile, macOS shows a "Verify Certificate" prompt on first join and **again every time the server certificate changes**. Users who click Cancel produce "wrong password" tickets. Client-side diagnosis: `Networking Guide/Wifi Guide/macos-wifi-troubleshooting-guide.md` §4.2 and §4.6.

#### Phones, tablets and unmanaged devices

iOS/iPadOS and Android need the same answers: EAP method PEAP, phase 2 MSCHAPv2, CA certificate (Android: "Use system certificates" works because radius.example.com is publicly issued) and domain `radius.example.com`, identity = AD username. [CONFIRM: policy on whether personal phones are permitted on the staff SSID at all, or only on guest.]

### Guest wireless

| Setting | Value | Reason |
|---------|-------|--------|
| SSID | [FILL IN] | |
| Auth | [FILL IN: WPA2-PSK, captive portal with vouchers, or open + portal] | |
| Controller guest isolation | Enabled (Omada: "Guest Network" toggle on the WLAN; UniFi: Guest network type / client device isolation) | Blocks guest clients from private subnets and from each other at the AP |
| VLAN | PO Guest 192.168.20.0/24 / Partner Guest 10.0.1.0/24 — [CONFIRM VLAN IDs] | Separate L3 network |
| pfSense rules | Guest interface: block to RFC1918, allow DNS to the resolver, allow internet | Isolation is enforced at the firewall too; the AP setting alone is not a security boundary |
| PSK storage and rotation | 1Password / IT vault; rotate every 365 days and when staff who knew it leave — [CONFIRM] per `Security & Hardening/secrets-management-guide.md` | |
| Bandwidth limit | [CONFIRM: proposal — per-client rate limit on the guest WLAN] | Protects staff traffic on the same APs |

Verify isolation from a device on the guest SSID (testing, not reading rules, per the IPAM guide):

```zsh
ping -c 3 192.168.30.1                    # staff gateway — must fail
nc -vz -w 3 10.60.1.145 445              # auth4 SMB — must fail
nc -vz -w 3 omada.example.com 443              # controller UI — must fail
curl -sI https://www.example.com | head -1     # internet — must succeed
```

Large-event guest Wi-Fi (the AGM) is a venue network the organization does not control — see `Networking Guide/AGM Wifi Survey/`.

---

## Server Certificate (EAP / PEAP)

### Why it matters

During PEAP the NPS server presents `radius.example.com` to the client inside the EAP exchange. Clients that are configured properly check (1) the chain ends in a trusted root, (2) the name is `radius.example.com`, (3) it has not expired. When the certificate expires, or is replaced by one the clients do not trust, **every** client fails on its next authentication — which for a roaming laptop is minutes. There is no partial failure and no grace period.

Note that this certificate cannot be checked with `openssl s_client` like a web or LDAPS cert: it is only ever presented inside RADIUS/EAP. Check it on the server, or capture it with `eapol_test -o` (see Testing).

### Lifecycle facts

| Item | Value |
|------|-------|
| Subject / SAN | radius.example.com |
| CA | DigiCert (Thawte RSA CA 2018 brand), ordered at digicert.com |
| Format | .pfx including private key; password set at CSR time, stored in 1Password / IT vault (per KB: if the password is lost, the certificate must be reissued) |
| Current expiry | [FILL IN: from the command below] |
| Renewal lead time | 60 days before expiry, client side planned 30 days before — per `Security & Hardening/certificate-pki-lifecycle-guide.md` ([CONFIRM] there) |
| Validity trend | Public TLS certificates were capped at 398 days, and CA/Browser Forum ballot SC-081 reduces this to 200 days from 2026-03-15, then 100 days (2027) and 47 days (2029). Renewals of a public RADIUS cert will become several times a year |

```powershell
Get-ChildItem Cert:\LocalMachine\My | Where-Object Subject -like "*radius.example.com*" | Sort-Object NotAfter | Format-Table NotAfter,Thumbprint,Issuer -AutoSize   # every copy, oldest first
```

[CONFIRM: proposal — because public validity is shrinking and every staff device that uses this SSID is either domain-joined or managed, move the RADIUS certificate to the internal AD CS CA (the KB notes the Certification Authority role was added for this deployment) with the root pushed by GPO and profile. That decouples Wi-Fi from public-CA policy changes. Decide before the next renewal.]

### Replacement procedure

Do this in a low-traffic window, with a second admin able to reach the NPS console. Expected impact if done right: none. If done wrong: all staff Wi-Fi.

**T-60 days — obtain the certificate**

1. Reissue/renew at digicert.com for `radius.example.com`. Generate the CSR on auth4 (IIS Manager → Server Certificates → Create Certificate Request, or `certreq -new`) so the private key never leaves the server, or follow the KB's .pfx route.
2. When issued, **inspect the chain before doing anything else**: is the issuing intermediate and root the same as the current certificate? If the root changed, stop and do the client step (T-30) first, with both old and new roots in the trust lists.

**T-30 days — client side**

3. Update the Windows Wireless GPO and the macOS profile so the trusted roots include the new chain's root (keep the old root too until cut-over is complete). Server name `radius.example.com` stays the same.
4. Tell staff on unmanaged devices that they may see a one-time "Verify Certificate" prompt on the cut-over date, and that the correct name to check is radius.example.com.

**Cut-over day — server side** (procedure from the KB and the 2024-02-12 screenshot)

5. Import the new .pfx into Local Computer\Personal (Setup step 3). Confirm `HasPrivateKey` is True.
6. Open `nps.msc` → Policies → Connection Request Policies → double-click **ORG_Wireless** → Settings → Authentication Methods → EAP Types: select Microsoft: Protected EAP (PEAP) → **Edit**.
7. In "Certificate issued to", pick the new certificate. Old and new show the same name; tell them apart by the **Expiration date** shown under the drop-down.
8. OK → Apply → OK.
9. If any Network Policy also has PEAP configured with a certificate (override disabled there), repeat the selection there. [CONFIRM: check whether `UbiVPN` uses a certificate too.]
10. If a secondary NPS server exists, repeat steps 5–9 on it. The certificate selection is not replicated.

**Verify immediately**

```bash
eapol_test -c peap-test.conf -a 10.60.1.145 -s "$RADIUS_SECRET" -o /tmp/radius-server.pem   # must end in SUCCESS; -o saves the cert NPS presented
openssl x509 -in /tmp/radius-server.pem -noout -subject -issuer -enddate                     # must show the NEW expiry date
```

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security';Id=6272,6273;StartTime=(Get-Date).AddMinutes(-15)} | Group-Object Id | Format-Table Name,Count   # grants should continue, no spike in 6273
```

Then join from one Windows laptop and one Mac. Keep watching 6273 for an hour.

**Rollback:** re-select the previous certificate in step 7 (still in the store, still valid) → Apply. Investigate before trying again.

**Afterwards:** remove the old certificate from the store once it has expired (not before), update the inventory row in `Security & Hardening/certificate-pki-lifecycle-guide.md`, record the new expiry in this guide's Quick Facts and in the change log.

---

## Operations (Day-2)

### Start / Stop / Restart

```powershell
Restart-Service IAS          # NPS; in-flight authentications fail, a few seconds of impact
Get-Service IAS              # expect Running
```

Omada software controller on Linux (if `omada.example.com` is a software controller rather than an OC200/OC300 appliance — [FILL IN]):

```bash
sudo tpeap status            # controller service state
sudo tpeap restart           # controller restart; APs and client traffic are unaffected
```

UniFi on a self-hosted Linux controller: `sudo systemctl restart unifi` (see `Networking Guide/Ubiquiti.txt`).

### Health checks

```powershell
Get-Service IAS                                                                   # NPS running
Get-NetUDPEndpoint -LocalPort 1812                                                # listening
Get-WinEvent -FilterHashtable @{LogName='Security';Id=6272} -MaxEvents 1 | Select-Object TimeCreated   # last successful auth should be minutes old during work hours
```

```bash
curl -sk -o /dev/null -w "%{http_code}\n" https://omada.example.com/     # controller UI answers — [CONFIRM: port; Omada software controller defaults to 8043]
```

### Add or replace an access point

1. Adopt the AP in the controller (Omada: Devices → pending → Adopt). Give it a reserved management IP — [FILL IN: where AP reservations are made, see `Networking Guide/dns-dhcp-administration-guide.md`].
2. Add the AP as a RADIUS client in NPS (Setup step 4), using the shared secret from the vault. Remove the entry for any AP being retired.
3. Record it in the AP inventory table in `Networking Guide/ipam-vlan-topology-reference.md`.
4. Test: join the staff SSID near the new AP, then look for a 6272 with that AP's friendly name as ClientName.

An AP that broadcasts the SSID but fails every authentication is almost always a missing or mistyped RADIUS client entry (NPS System log event 13).

### Change the RADIUS server the controllers use

This is the procedure in `Change Radius for WIfi.txt`, made safe:

1. Build and test the new NPS first (Setup steps 1–7), including its certificate, and run `eapol_test` against it.
2. Export the RADIUS client list from the old server and import on the new one (Backup & Restore below) so every AP is already known.
3. In Omada, create the new RADIUS profile; switch **one** non-critical WLAN or AP group first if possible, then ExampleCorp-WiFi.
4. Watch 6272/6273 on the new server and the Omada event log.
5. Leave the old server running, untouched, for at least a week as a rollback target. Then decommission it and update Quick Facts.

### Shared secret rotation

[CONFIRM: proposal — every 365 days, and on staff departure or suspected exposure, per `Security & Hardening/secrets-management-guide.md`.] Order matters because a mismatch rejects every user on the affected APs:

1. Generate the new secret in 1Password.
2. Update the NPS RADIUS client entries (`Set-NpsRadiusClient -Name <name> -SharedSecret $sec`, reading the secret as in Setup step 4).
3. Immediately update the controller RADIUS profile. The window between steps 2 and 3 is an outage for that site, so keep it to seconds and do it after hours.
4. Test with a client join and check for event 18 on NPS.

### Logs

```powershell
Get-WinEvent -FilterHashtable @{LogName='Security';Id=6273} -MaxEvents 20 | ForEach-Object { $d=@{}; ([xml]$_.ToXml()).Event.EventData.Data | ForEach-Object { $d[$_.Name]=$_.'#text' }; [pscustomobject]@{Time=$_.TimeCreated; User=$d.SubjectUserName; Mac=$d.CallingStationID; AP=$d.ClientName; Policy=$d.NetworkPolicyName; Code=$d.ReasonCode; Reason=$d.Reason} } | Format-Table -AutoSize   # last 20 rejects, one line each
Get-WinEvent -FilterHashtable @{LogName='System';ProviderName='NPS'} -MaxEvents 20 | Format-Table TimeCreated,Id,Message -Wrap   # unknown client (13), bad authenticator (18), etc.
Get-ChildItem "$env:SystemRoot\System32\LogFiles\IN*.log" | Sort-Object LastWriteTime | Select-Object -Last 3   # NPS accounting text logs, if enabled
```

Event IDs: Security 6272 = granted, 6273 = denied (Reason Code is the diagnosis, see Troubleshooting), 6274 = discarded. System log, source NPS: 13 = request from an IP that is not a RADIUS client, 18 = invalid Message-Authenticator (almost always a shared secret mismatch).

EAP-level tracing when the event log is not enough (verbose; turn it off afterwards):

```powershell
netsh ras set tracing * enabled        # writes RASTLS / IASSAM etc. logs to %windir%\tracing
netsh ras set tracing * disabled       # stop tracing
```

### Backup & Restore

| What | How | Where | Schedule |
|------|-----|-------|----------|
| NPS config (clients **with shared secrets**, policies) | `netsh nps export filename="C:\nps\nps-backup-$(Get-Date -f yyyyMMdd).xml" exportPSK=YES` | [FILL IN: backup destination]. The file contains shared secrets in clear text — encrypt it or store it in the vault, never on a share | [CONFIRM: proposal — weekly scheduled task, and before every change] |
| EAP server certificate | .pfx + password | 1Password / IT vault | On each renewal |
| Omada controller | Settings → Maintenance → Backup & Restore → Backup; enable Auto Backup — [CONFIRM: menu path for installed version; Global View in Omada 5.x] | [FILL IN: backup location — the IPAM guide also lists this as unknown] | [CONFIRM: proposal — weekly auto backup, plus manual before upgrades] |
| UniFi controller | Settings → Control Plane → Backups (cloud backups weekly and before updates) — see `Networking Guide/Network-Administration-Guide.md` §8.10 | [FILL IN] | Weekly |

```powershell
Export-NpsConfiguration -Path "C:\nps\nps-backup.xml"     # PowerShell equivalent of netsh nps export (includes secrets)
Import-NpsConfiguration -Path "C:\nps\nps-backup.xml"     # restore, or seed a secondary NPS server
```

Restore test: [FILL IN: date of last successful NPS import into a test server]. An untested restore is a rumour — see `Hardware & Backup/backup-restore-testing-procedure.md`.

### Updates / Upgrades

- NPS is patched with Windows on auth4/auth2 — see `Windows & Mac Workstations/windows-server-2019-runbook.md`. After each cumulative update, check 6272 events are still flowing. (NPS-affecting changes have shipped in monthly updates, e.g. the July 2024 BlastRADIUS fix.)
- AP firmware and controller upgrades: back up the controller first; upgrade the controller before the APs; upgrade one AP and test 802.1X before the rest.

---

## Testing

### eapol_test (end-to-end PEAP test without a laptop)

`eapol_test` (from wpa_supplicant) performs a real PEAP-MSCHAPv2 authentication against NPS, exactly as an AP would. Run it from a Linux host on the server network — [FILL IN: which admin/jump host]. That host must be added to NPS as a RADIUS client (name it `test-eapol-<host>`), and the network policy must allow a test account.

```bash
sudo apt install eapoltest                  # Debian/Ubuntu package that provides eapol_test
```

`peap-test.conf` (no secrets in the file):

```
network={
    key_mgmt=WPA-EAP
    eap=PEAP
    identity="[FILL IN: dedicated test account, member of the Wi-Fi group]"
    anonymous_identity="anonymous"
    password=""
    phase2="auth=MSCHAPV2"
    ca_cert="/etc/ssl/certs/ca-certificates.crt"
    domain_suffix_match="radius.example.com"
}
```

Supply the password at run time rather than storing it: copy the file to a mode-600 temp file and insert the password there, then delete it — or use a test account whose password is rotated after each use. [CONFIRM: proposal — a dedicated `svc-wifitest` account, member of the Wi-Fi group only, password in the vault.]

```bash
read -rs RADIUS_SECRET                                                                     # paste shared secret, not echoed
eapol_test -c peap-test.conf -a 10.60.1.145 -p 1812 -s "$RADIUS_SECRET" -o /tmp/radius-server.pem -r 0   # last line SUCCESS = full auth path works
openssl x509 -in /tmp/radius-server.pem -noout -subject -issuer -enddate                  # which cert NPS presented, and its expiry
unset RADIUS_SECRET                                                                        # clean up
```

`domain_suffix_match` makes the test fail if NPS presents the wrong certificate — exactly what a correctly configured client would do.

### radtest (reachability and client registration only)

`radtest` (package `freeradius-utils`) sends a simple PAP request. NPS does not allow PAP for wireless, so a **reject** is the expected answer — what matters is that an answer arrives.

```bash
sudo apt install freeradius-utils                                     # provides radtest
radtest testuser wrongpassword 10.60.1.145 0 "$RADIUS_SECRET"        # Access-Reject = NPS reachable and this host is a known client
```

No reply at all means firewall, the test host not registered as a RADIUS client (NPS System event 13), or a secret mismatch (event 18).

### From a real client

Windows: `netsh wlan show wlanreport`. macOS: `log stream --predicate 'process == "eapolclient"'` while joining. Correlate the timestamp and MAC (Calling-Station-Id) with a 6272/6273 on NPS. Beware private/random MAC addresses on macOS, iOS and Android — see `Networking Guide/Wifi Guide/macos-wifi-troubleshooting-guide.md` §2.4.

### Packet capture on the NPS server

```powershell
pktmon filter add -p 1812                       # RADIUS auth only
pktmon start --capture --pkt-size 0             # start capture
pktmon stop                                     # stop; writes PktMon.etl in the current directory
pktmon etl2pcap PktMon.etl --out radius.pcapng  # open in Wireshark with filter: radius || eap
pktmon filter remove                            # clear filters
```

Wireshark filters for EAP: `Networking Guide/Wifi Guide/wireshark-filter-primer.md` (`eap.code == 4` = EAP Failure).

---

## Troubleshooting

Start every investigation on NPS: the 6273 reason code usually names the fault outright. Client-side symptoms on macOS are covered in `Networking Guide/Wifi Guide/macos-wifi-troubleshooting-guide.md` §4.2/§4.6 and the triage chart `Networking Guide/Wifi Guide/network-triage-decision-tree.html`.

### Failed authentication by NPS reason code (event 6273)

| Code | Meaning | Usual cause in this environment | Fix |
|------|---------|--------------------|-----|
| 7 | Domain does not exist | User typed `user@wrongdomain` or a personal email | Identity = AD username |
| 8 | User account does not exist | Typo, or departed staff | Check spelling; `Get-ADUser <name>` |
| 16 | Authentication failed due to user credentials mismatch | Wrong or stale saved password — very common after an AD password change | User forgets the network and rejoins with current password; watch for lockouts |
| 22 | EAP type cannot be processed by the server | Client offering an EAP type NPS does not allow (e.g. EAP-TTLS, or EAP-TLS with no cert policy) | Fix the client/profile to PEAP |
| 23 | Error during NPS use of EAP | Server certificate problem: none selected, expired, private key unreadable, or chain broken | Check the cert (Server Certificate section); EAP tracing |
| 33 | User must change password | Password expired or "must change at next logon" | Change password on a domain machine or webmail first |
| 34 | User account disabled | Departed or disabled account | Expected; or re-enable if wrong |
| 36 | User account locked out | Repeated 16s from a device with a stale saved password | Unlock; find the device (CallingStationID) still retrying with the old password |
| 48 | Did not match any network policy | User not in "Office Group", or NAS-Port-Type/conditions mismatch | `Get-ADPrincipalGroupMembership <user>`; check policy conditions |
| 49 | Did not match any connection request policy | Request doesn't look wireless (wrong NAS-Port-Type) or `ORG_Wireless` disabled | Check CRP enabled and conditions |
| 65 | Dial-in permission denied on the user account | Account's Dial-in tab set to "Deny access" | Set to "Control access through NPS Network Policy", or enable "Ignore user account dial-in properties" in the policy |
| 66 | Authentication method not enabled on the matching policy | Client using a method the policy doesn't allow (e.g. radtest PAP) | Expected for radtest; otherwise fix client |
| 259 | CRL/revocation check offline | Only relevant with client certs (EAP-TLS) | Check CRL distribution point reachability |
| 265 | Certificate chain issued by an untrusted authority | Only with client certs (EAP-TLS) | Fix client cert issuance/trust |

Reason codes are from Microsoft's NPS reason-code list; verify any unfamiliar code there before acting.

```powershell
Get-ADUser <username> -Properties Enabled,LockedOut,PasswordExpired,PasswordLastSet,msNPAllowDialin | Format-List   # account state behind codes 16/33/34/36/65
Get-ADPrincipalGroupMembership <username> | Select-Object Name                                                        # group membership behind code 48
Unlock-ADAccount <username>                                                                                           # after finding the device causing lockouts
```

### Symptom: nobody can join staff Wi-Fi; guest works

- Likely cause: NPS down, NPS unreachable, or the server certificate expired/changed.
- Check: `Get-Service IAS` and `Get-WinEvent -FilterHashtable @{LogName='Security';Id=6272,6273;StartTime=(Get-Date).AddMinutes(-30)} | Group-Object Id` — no events at all = requests not arriving; a wall of 6273 code 23 = certificate; 6273 code 16 for everyone = AD/Netlogon problem.
- Fix: no events → firewall/AP path and service state; code 23 → Server Certificate section (rollback is re-selecting the previous cert); 16 for all → check DC health (`dcdiag /q` on auth4) and the secure channel.

### Symptom: one AP (or one site) rejects everyone, other APs fine

- Likely cause: the AP is not a RADIUS client in NPS, its IP changed, or the secret differs.
- Check: `Get-WinEvent -FilterHashtable @{LogName='System';ProviderName='NPS'} -MaxEvents 20` — event 13 names the unknown IP; event 18 names a client with the wrong secret.
- Fix: add/correct the RADIUS client (Operations → Add or replace an AP), or re-enter the secret on both sides.

### Symptom: Windows laptops repeatedly prompt or fail; Macs fine

- Likely cause: Windows 11 Credential Guard blocking MSCHAPv2 SSO, or a stale saved credential.
- Check: on the client `Get-WinEvent -LogName "Microsoft-Windows-WLAN-AutoConfig/Operational" -MaxEvents 30`; on NPS look for code 16 from that user, or no request at all.
- Fix: disable "Automatically use my Windows logon name and password" in the Wireless GPO so users are prompted once and credentials are saved; long term, EAP-TLS.

### Symptom: Macs show "Verify Certificate" or fail; Windows fine

- Likely cause: certificate changed and the Mac has no profile (or the profile's trusted names/anchors don't match the new chain).
- Check: `log stream --predicate 'process == "eapolclient"'` while joining; `sudo profiles show -type configuration`.
- Fix: update/install the Wi-Fi profile; for unmanaged Macs the user accepts the prompt after confirming the name is radius.example.com.

### Symptom: connected and authenticated, but no IP / 169.254.x.x / wrong subnet

- Likely cause: 802.1X succeeded (look for a 6272), the problem is after it: SSID mapped to the wrong VLAN, VLAN not trunked to the AP's switch port, or DHCP scope exhausted.
- Check: 6272 exists for the user → not an auth problem. Then `ipconfig getsummary en0` (Mac) / `ipconfig /all` (Windows) and the SSID→VLAN table in the IPAM reference.
- Fix: correct VLAN on the WLAN or switch trunk (`Networking Guide/ipam-vlan-topology-reference.md`), DHCP per `Networking Guide/dns-dhcp-administration-guide.md`.

### Symptom: wireless broke right after an NPS or Windows update

- Check: `Get-WinEvent -FilterHashtable @{LogName='System';ProviderName='NPS'} -MaxEvents 20` for event 18 across many clients — the Message-Authenticator requirement may have been tightened.
- Fix: update AP/controller firmware so it sends Message-Authenticator; as a temporary measure untick the requirement on affected RADIUS clients and record it as an exception.

---

## Security

- **Exposure:** internal only. NPS answers UDP 1812/1813 to registered RADIUS clients; restrict at pfSense to the AP management subnet(s). NPS runs on a domain controller, so anything that can reach 1812 is talking to a DC — keep the source list tight.
- **Auth:** AD user accounts, restricted by group in the network policy. The guest network has no path to AD.
- **Evil twin risk:** PEAP-MSCHAPv2 is only safe if clients validate the server certificate and name. Push trust via GPO and profile with "don't prompt" so users can't accept a rogue certificate. Detection and response to rogue/evil-twin APs: `Networking Guide/Wifi Guide/wireless-security-incident-response.md` §3 and §2.
- **Certificates:** see Server Certificate section; inventory and alerting in `Security & Hardening/certificate-pki-lifecycle-guide.md`.
- **Secrets:** RADIUS shared secrets, controller admin passwords, guest PSK, PFX password — 1Password / IT vault only. NPS exports contain secrets in clear text; treat export files as secrets.
- **Management plane:** controller UIs reachable only from the management network or VPN — [FILL IN: confirm current exposure of omada.example.com and the UniFi controller]. Controller admin accounts should be named, not shared — [CONFIRM: proposal].
- **Hardening proposals** ([CONFIRM] each): require Message-Authenticator on all RADIUS clients; untick MS-CHAP v1; enable PEAP cryptobinding; WPA2/WPA3 mixed with PMF; per-site shared secrets. Disabled AD accounts are already rejected automatically (code 34); also verify that offboarding removes Office Group membership.
- **Offboarding:** disabling the AD account blocks Wi-Fi at next authentication; an already-connected session can persist until re-auth. [CONFIRM: proposal — set a Session-Timeout (e.g. 8 hours) in the network policy so sessions re-authenticate daily.]

## Monitoring & Alerting

- **What exists:** [FILL IN: are NPS Security events (6272/6273) forwarded to graylog01.example.com? If so, which stream/dashboard.] See `SysAdmin Procedures/monitoring-alerting-guide.md`.
- **Certificate expiry:** the RADIUS cert is listed among the highest-value expiry checks in the monitoring guide. Because it can't be probed over the network with `openssl s_client`, use a scheduled check on the NPS server:

```powershell
Get-ChildItem Cert:\LocalMachine\My | Where-Object { $_.Subject -like "*radius.example.com*" -and $_.NotAfter -lt (Get-Date).AddDays(60) } | Select-Object NotAfter,Thumbprint   # any output = act now
```

  Or run the `eapol_test -o` test on a schedule from the Linux host and check `openssl x509 -checkend 5184000` (60 days) on the captured certificate.
- **Proposed alerts** ([CONFIRM] thresholds): 6273 count > [FILL IN: baseline × 5] in 15 minutes (mass failure); zero 6272 in 30 minutes during business hours (NPS or path down); any NPS System event 13 or 18 (unknown client / bad secret); IAS service stopped; same user > 10 code-16 rejects in an hour (lockout in progress or password spraying); certificate < 60 days.
- **Baseline:** [FILL IN: typical 6272 count per hour at mid-morning, number of APs, peak client count per AP from the controller].
- **Controller health:** AP disconnected alerts in Omada (Settings → Notifications / Alerts) — [FILL IN: configured recipient].

## Disaster Recovery

| Scenario | Impact | Recovery |
|----------|--------|----------|
| NPS service stopped | All staff Wi-Fi fails at next re-auth | `Restart-Service IAS`; check logs |
| auth4 lost entirely, secondary NPS exists | Brief failover delay per AP | Controllers fail over to secondary automatically if configured; rebuild auth4 at leisure |
| auth4 lost, no secondary | Staff Wi-Fi down | Install NPS on auth2 (Setup steps 1–3), `Import-NpsConfiguration` from latest export, import PFX from vault, re-select cert in `ORG_Wireless`, point the Omada RADIUS profile at auth2 (10.60.1.221). Check pfSense allows the AP subnet to reach auth2 |
| Server certificate expired | Staff Wi-Fi down for everyone | Emergency reissue at DigiCert (same name, same CA family), then the cut-over steps. While waiting: guest SSID for staff internet access, with VPN for internal resources |
| Omada controller lost | No config changes possible; Wi-Fi keeps running | Reinstall controller, restore latest backup, re-adopt APs if needed |
| Shared secret lost from vault | No immediate impact | Rotate (Operations) |

- **RTO / RPO:** [CONFIRM: proposal — RTO 2 hours for staff Wi-Fi; RPO = last weekly NPS export]. Guest Wi-Fi is the fallback and has no RADIUS dependency.
- **Break-glass:** [CONFIRM: proposal — no standing PSK-based staff SSID. In a prolonged RADIUS outage, staff use the guest SSID plus VPN. Do not create an emergency PSK SSID on the staff VLAN.]
- **Rebuild order:** AD/DNS healthy → NPS role + registration → config import → certificate → firewall path → controller RADIUS profile → test with eapol_test → test with real clients.
- **Escalation:** IT lead (it@example.com) → [FILL IN: second contact with Domain Admin and controller access] → [FILL IN: TP-Link/Ubiquiti support contract, if any]. Business continuity context: `SysAdmin Procedures/business-continuity-plan.md`.

## Wired 802.1X (pointer only)

The same NPS can authenticate wired ports. EdgeSwitch 802.1X commands, including the monitor-mode rollout (`dot1x system-auth-control monitor`), are in `Networking Guide/Network-Administration-Guide.md` §9.3. A wired rollout needs its own Connection Request Policy (NAS Port Type = Ethernet) and network policy, so it cannot accidentally change wireless behaviour. [FILL IN: is wired 802.1X enabled anywhere today?]

## A note on FreeRADIUS

This environment runs Windows NPS; FreeRADIUS is not in use. If it is ever proposed, the mapping is `clients.conf` = NPS RADIUS clients, `mods-available/eap` = the `ORG_Wireless` PEAP settings, Samba `ntlm_auth` = NPS's native MSCHAPv2-to-AD check. That would be a new design decision (ADR), not a drop-in swap.

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why / Source |
|------------|-------------------|--------------|
| [FILL IN]  | NPS on Windows Server with Aerohive APs (SSID ORG_Wireless), public CA certificate rather than AD CS | KB article "802.1x and Active Directory" (Windows 2012 RADIUS + Aerohive) |
| [FILL IN]  | Aerohive replaced by TP-Link Omada APs (omada.example.com) | KB marked "Deprecated — Needs to be rewritten for the TP-Link APs" |
| 2024-02-12 | radius.example.com certificate updated in `ORG_Wireless` PEAP settings on auth.example.com (cert shown expiring 2024-12-21) | `Update Radius Certificate.jpg` |
| 2024-02-12 | KB article last updated | KB footer |
| 2025-04-10 (approx.) | Omada RADIUS profile switched from auth.example.com to auth4.example.com | `Change Radius for WIfi.txt` (file last modified 2025-04-10) |
| 2026-10-04 | This guide written to replace the deprecated KB and consolidate the RADIUS notes | Library review |

## References

Primary sources (relative to the library root):

- `[internal KB: 802.1x and Active Directory]` — original NPS build and cert update (Aerohive era, deprecated)
- `Security & Hardening/Change Radius for WIfi.txt` — Omada RADIUS profile switch to auth4
- `Security & Hardening/Update Radius Certificate.jpg` — NPS console: CRPs, PEAP settings, cert selection
- `[internal KB: network subnet list]`, `Networking Guide/ipam-vlan-topology-reference.md` — subnets, SSID→VLAN and AP tables

Related guides (linked in the text above): `Security & Hardening/certificate-pki-lifecycle-guide.md`, `Security & Hardening/secrets-management-guide.md`, `Security & Hardening/LDAPs certificate replacement procedure.txt`, `Networking Guide/Network-Administration-Guide.md`, `Networking Guide/Ubiquiti.txt`, `Networking Guide/firewall-change-procedure.md`, `Networking Guide/dns-dhcp-administration-guide.md`, `Networking Guide/Wifi Guide/` (macOS troubleshooting, wireless security IR, Wireshark primer), `Networking Guide/UEWA_Training_Guide_V2.1.pdf` (UniFi WPA-Enterprise and guest portal background), `Active Directory/ldap-connection-reference.md`, `Windows & Mac Workstations/windows-server-2019-runbook.md` §15, `Windows & Mac Workstations/mdm-and-apple-business-manager-guide.md`, `SysAdmin Procedures/monitoring-alerting-guide.md`.

External: Microsoft Learn NPS documentation (RADIUS clients, CRPs, network policies, event IDs and reason codes); Microsoft guidance for CVE-2024-3596 (BlastRADIUS); Microsoft Credential Guard considerations (MSCHAPv2 SSO); hostap `eapol_test`; TP-Link Omada controller user guide; Ubiquiti help center (RADIUS profiles, backups); CA/Browser Forum ballot SC-081.

## Change log

| Date       | Author | Change |
|------------|--------|--------|
| 2026-10-04 | IT lead (it@example.com), drafted with Claude | Initial draft. Consolidates the KB article, Omada RADIUS note and NPS screenshot. UNVERIFIED — all [FILL IN]/[CONFIRM] markers outstanding |

---
Template v1 (Tier 3) | Maintained by IT Ops | Review annually, after any certificate renewal, and after any RADIUS server or controller change

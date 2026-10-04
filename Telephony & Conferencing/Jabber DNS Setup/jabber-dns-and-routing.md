> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# Cisco Jabber — DNS, Expressway and Carrier Routing

Promotes the 86-byte `_ARCHIVE/superseded-stubs/Carrier Jabber routes.txt` (archived 2026-09-11) note in this folder into a working document. The
original file's contents are preserved verbatim in the Carrier Routing section below and it should be
treated as superseded, not deleted.

---

## Overview

Cisco Jabber is the organization's softphone and instant-messaging client. Staff use it to make and receive calls
on their extension from a laptop or phone, on site and off. The call platform itself is
delivered by the carrier — the organization does not run Unified Communications Manager (CUCM) in house — and remote
clients reach it through Cisco Expressway edge infrastructure on the carrier's side at
`edge.carrier.example.net`.

The part the organization actually owns is DNS and the network path: publishing the SRV records that let Jabber
clients discover where to register, and making sure the carrier service subnets are reachable and not
blocked at the pfSense edge. Almost every Jabber problem is one of those two things.

## Quick Facts

| Field            | Value                                                                 |
|------------------|-----------------------------------------------------------------------|
| Owner            | IT (it@example.com) — service provided by the carrier      |
| Environment      | prod                                                                   |
| Location         | Carrier-hosted call platform; Expressway edge at `edge.carrier.example.net`. No on-premises call servers |
| Access           | DNS zone for `example.com` at [FILL IN: DNS provider/registrar]; internal DNS on [FILL IN: internal DNS servers — likely auth2/auth4.example.com]; pfSense firewall admin |
| Dependencies     | Public and internal DNS SRV records; internet path to the carrier subnets listed below; pfSense firewall/NAT rules and Snort not blocking them; carrier service availability; user accounts provisioned by the carrier |
| Dependents       | Softphone calling and IM for all Jabber users; [FILL IN: whether voicemail — see the Unity voicemail guide in the parent folder — and conferencing depend on the same path] |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                                |

| Item                       | Value                                            |
|----------------------------|--------------------------------------------------|
| Jabber client version      | [FILL IN: deployed client version and platforms — Windows/macOS/iOS/Android] |
| Deployment mode            | [FILL IN: full UC (phone + IM), IM-only, or phone-mode] |
| Expressway edge FQDN       | `edge.carrier.example.net`                            |
| Carrier account / circuit ID | [FILL IN: account number and support contact]     |
| User count                 | [FILL IN: number of provisioned Jabber users]     |

## How It Works

Jabber has no configuration wizard for users in normal operation — it bootstraps itself from DNS.
The client takes the domain part of what the user types (their email address or UPN, `@example.com`) and
performs a series of SRV lookups to find where to register. Which record it finds determines whether
it registers as an on-network client or through the edge.

    user enters name@example.com in Jabber
        -> SRV lookup on example.com
             _cisco-uds._tcp.example.com      # found on the internal network: register directly
             _collab-edge._tls.example.com    # found off-network: register via Expressway edge
        -> client contacts the returned host, authenticates, downloads its service profile
        -> media (RTP) flows to/from the carrier voice network over UDP

Two different answers, one for inside and one for outside, is the normal design. Internal DNS
resolves the internal record; public DNS resolves the edge record. A client on the corporate LAN
that finds the `_collab-edge` record instead will still work but will hairpin all its traffic out
and back through the edge, which shows up as poor audio quality rather than as an obvious failure.

Because the call platform is carrier-hosted, the "internal" path may not exist at all — clients
on the LAN may use the edge path like everyone else:
[FILL IN: confirm whether internal clients use `_cisco-uds` internally at all, or whether all registration goes through `_collab-edge` to edge.carrier.example.net].

Signalling is TLS to the edge; media is UDP RTP directly between the client and the carrier media
path. That split matters when troubleshooting: a client can register successfully (signalling works)
and still have no audio (media blocked), which is the classic symmetric-NAT/UDP-blocked symptom.

---

## DNS Requirements

### SRV records

Jabber looks these up in order. Record targets and ports must come from the carrier DNS zone
instructions — see `Customer DNS Zone Creation for Expressway.docx` and
`Jabber DNS instructions.pdf` in this folder, which are the authoritative source the carrier supplied.

| Record | Purpose | Where it must resolve | Target | Port | Priority / Weight |
|--------|---------|-----------------------|--------|------|-------------------|
| `_cisco-uds._tcp.example.com` | Internal service discovery (UDS) | Internal DNS only | [FILL IN: target, or "not used — carrier-hosted"] | [FILL IN: typically 8443] | [FILL IN: priority / weight] |
| `_collab-edge._tls.example.com` | Off-network discovery via Expressway (MRA) | Public DNS | [FILL IN: confirm target — expected to be `edge.carrier.example.net` or a per-customer edge name supplied by the carrier] | [FILL IN: typically 8443] | [FILL IN: priority / weight] |
| `_cuplogin._tcp.example.com` | IM and Presence login (legacy discovery) | [FILL IN: internal, public, or not used] | [FILL IN: target host from the carrier] | [FILL IN: typically 8443] | [FILL IN: priority / weight] |
| `_sips._tcp.example.com` | SIP over TLS, if used | [FILL IN: internal, public, or not used] | [FILL IN: target host from the carrier] | [FILL IN: typically 5061] | [FILL IN: priority / weight] |
| `_sip._tcp.example.com` | SIP, if used | [FILL IN: internal, public, or not used] | [FILL IN: target host from the carrier] | [FILL IN: typically 5060] | [FILL IN: priority / weight] |
| `_xmpp-client._tcp.example.com` | XMPP/IM, if used | [FILL IN: internal, public, or not used] | [FILL IN: target host from the carrier] | [FILL IN: typically 5222] | [FILL IN: priority / weight] |

Verify what is actually published before changing anything:

    dig +short SRV _collab-edge._tls.example.com @8.8.8.8       # what an off-network client sees
    dig +short SRV _cisco-uds._tcp.example.com                  # what an on-network client sees (run from inside)
    dig +short SRV _cuplogin._tcp.example.com
    dig +short A edge.carrier.example.net                      # resolve the carrier edge itself
    nslookup -type=SRV _collab-edge._tls.example.com            # Windows equivalent, for testing from a user's PC

Rules that catch people out:

- The SRV lookup uses the **domain part of the user's sign-in name**, not the server's hostname. If
  users sign in as `name@example.com`, the records must be on `example.com` — not on a subdomain, and not on
  an internal-only AD domain name.
- The edge record must resolve from the **public internet**. Test it from off-network, or with an
  external resolver as above; an internal resolver that has a forward zone for `example.com` can answer
  correctly while the public zone is empty.
- Split-horizon DNS is the expected design: the same name answers differently inside and outside.
  Confirm the internal zone for `example.com` on [FILL IN: internal DNS servers] and the public zone
  separately — they are edited in different places.
- The edge host's TLS certificate must match the name the SRV record points to, and the client must
  trust its issuing CA. A certificate name mismatch presents to the user as "cannot communicate with
  the server", not as a certificate error.

### Records the carrier must provide

Anything in the target column above is the carrier's to supply, not something to guess. Request the
current DNS zone instruction sheet from the carrier if the `.docx` in this folder is out of date —
it is dated 2024-03-21. Carrier support contact: [FILL IN: carrier support number/portal and account reference].

---

## Carrier Routing

The original `_ARCHIVE/superseded-stubs/Carrier Jabber routes.txt` (archived 2026-09-11) (2024-03-20) recorded the carrier service subnets and edge
hostname. Preserved verbatim:

    100.64.0.0/23
    100.64.2.0/23
    100.64.4.0/23
    100.64.6.0/23

    edge.carrier.example.net

These are the carrier-side networks Jabber signalling and media use. Notes:

- They are in the `100.64.0.0/10` carrier-grade NAT range. That matters because some firewall
  rulesets, bogon filters and IDS default policies treat CGNAT space as non-routable and drop it.
  If pfSense is configured to block bogon or private networks on the WAN interface, these ranges can
  be silently discarded — check that first when nothing works and DNS is confirmed good.
- These subnets must be permitted outbound (and their return traffic allowed) through pfSense.
  [FILL IN: the pfSense firewall rules/aliases that currently permit these networks, and the interface they are on].
- Snort on pfSense can block a host in these ranges after a false positive, which takes down Jabber
  for everyone while everything else looks healthy. To check and clear a Snort block, see
  the internal KB article "Unblocking an IP from Snort".
- Consider creating a pfSense alias containing these four subnets plus a suppression/pass-list entry
  in Snort, so an accidental block cannot recur: [FILL IN: whether such an alias and Snort pass list exist].

    dig +short A edge.carrier.example.net                      # confirm the edge resolves
    traceroute edge.carrier.example.net                        # where the path stops, if it does
    nc -vz edge.carrier.example.net 8443                       # can we reach the edge signalling port

Carrier-side configuration — dial plan, extension-to-user mapping, inbound DID routing, voicemail
integration — is changed by raising a ticket with the carrier, not in-house.
[FILL IN: the carrier change-request process and typical turnaround].

---

## Port Requirements

The authoritative port lists are the two Cisco documents already in this folder; consult them rather
than relying on a summary when opening firewall rules:

- `TCP and UDP Port Usage Guide for Cisco Unified Communications Manager, Release 10.0(1) - Cisco Unified Communications Manager TCP and UDP Port Usage [Cisco Unified Communications Manager (CallManager)] - Cisco.pdf`
- `TCP and UDP Port Usage Guide for Cisco Unified Communications Manager, Release 10.0(1) - Port Usage Information for the IM and Presence Service [Cisco Unified Communications Manager (CallManager)] - Cisco.pdf`
- `Planning Guide for Cisco Jabber 12.6 - Requirements [Cisco Jabber for Windows] - Cisco.pdf` — client-side requirements including supported OS versions and network prerequisites.

The practical minimum for an off-network Jabber client through Expressway:

| Purpose | Protocol / Port | Direction | Notes |
|---------|-----------------|-----------|-------|
| DNS SRV lookups | UDP/TCP 53 | outbound | Must reach a resolver that can answer public SRV |
| Expressway edge signalling (HTTPS/SIP over TLS) | TCP 8443 | outbound to edge | The registration path; if this fails the client never signs in |
| SIP over TLS | TCP 5061 | outbound | [FILL IN: confirm whether the carrier requires 5061 in addition to 8443] |
| XMPP (IM/presence) | TCP 5222 | outbound | Only if IM is in use |
| Media (RTP/RTCP) | UDP 36000–59999 | outbound, bidirectional flow | The range Expressway uses by default; audio fails without it while sign-in still works |
| Secure media (SRTP) | UDP, same range | bidirectional | — |
| Certificate validation (CRL/OCSP) | TCP 80/443 | outbound | Blocked OCSP shows as intermittent TLS failures |

Exact ranges in use for this deployment: [FILL IN: the media port range the carrier has configured for edge.carrier.example.net — confirm with the carrier rather than assuming the Cisco default].

Outbound UDP in a wide ephemeral range is the requirement most likely to be missing on a restrictive
guest or third-party network, and it is why Jabber often signs in but has one-way or no audio from a
hotel or client-site Wi-Fi. That is not an organization-side fault and usually cannot be fixed remotely.

---

## Client Setup

Jabber client installation and first sign-in for staff is a desktop-support task, not a server task.
The user flow is: install the client, enter `firstname.lastname@example.com`, authenticate, and let the
client discover its configuration by SRV.

- Client packages and deployment method: [FILL IN: how Jabber is distributed — Munki for Macs (see `_ARCHIVE/superseded-stubs/Adding a package to munki.txt` (archived 2026-09-11)), GPO/Intune for Windows, or manual].
- Credentials: [FILL IN: whether Jabber sign-in uses AD credentials via the carrier platform, or separate carrier-issued credentials].
- End-user quick-reference material lives in `Telephony & Conferencing/Tip Sheets/`; voicemail
  is covered by `Telephony & Conferencing/Unity Quick Reference Voicemail User Guide.pdf`.
- Screenshots of the working client and server configuration as captured during the 2024 setup are
  in this folder: `Screenshot 2024-03-08 at 12.07.49 PM.jpg`, the three
  `Screenshot 2024-03-08 at 2.08.*.jpg` files, and `Screenshot 2024-03-20 at 2.37.37 PM.png` /
  `Screenshot 2024-03-20 at 3.14.19 PM.png`. Treat them as a record of how it looked when it worked.
  [FILL IN: label what each screenshot actually shows, so they are useful without opening all of them].

To force a client to reset its discovered configuration, sign out, quit, and clear the local
profile, then sign in again:

    # macOS
    rm -rf ~/Library/Application\ Support/Cisco/Unified\ Communications/Jabber    # clears cached config and credentials
    # Windows
    # delete %APPDATA%\Cisco\Unified Communications\Jabber and %LOCALAPPDATA%\Cisco\Unified Communications\Jabber

This is the first thing to try on a single user whose client misbehaves while everyone else is fine
— it discards a stale cached server list.

---

## Troubleshooting

### Symptom: Jabber will not sign in — "cannot communicate with the server"
- Likely cause: SRV discovery failing, or the edge is unreachable.
- Check:  `dig +short SRV _collab-edge._tls.example.com @8.8.8.8` &nbsp;# empty means the public record is missing
- Check:  `nc -vz edge.carrier.example.net 8443` &nbsp;# connection refused/timeout means a network path problem
- Check:  pfSense firewall and Snort blocks against the four carrier subnets above
- Fix:    republish the SRV record, or clear the block. If DNS and the path are both good, raise it
  with the carrier.

### Symptom: one user cannot sign in, everyone else can
- Likely cause: stale cached client config, wrong sign-in address, or an account issue at the carrier.
- Check:  is the user entering `name@example.com` and not a bare username — the domain is what drives SRV
- Fix:    clear the local Jabber profile as above; if it persists, confirm with the carrier that the
  account is provisioned and not locked.

### Symptom: everyone lost Jabber at once, nothing changed on the clients
- Check:  DNS first — did the SRV or A records change or expire?

      dig +short SRV _collab-edge._tls.example.com @8.8.8.8    # confirm the record still exists publicly
      dig +short A edge.carrier.example.net                    # confirm the edge still resolves

- Check:  pfSense — a firewall or Snort change is the second most likely cause
- Check:  carrier service status before assuming it is on our side
- Fix:    per cause. Record what it was in Decisions & History.

### Symptom: signs in fine, but calls have no audio or one-way audio
- Likely cause: UDP media blocked or NAT-mangled. Signalling (TCP 8443) works, media (UDP) does not.
- Check:  whether the user is on a restrictive network — hotel, guest Wi-Fi, client site, or a VPN
  that forces all traffic through a tunnel
- Check:  the media port range is permitted outbound from the corporate network (see Port Requirements)
- Fix:    on the corporate network, open the media range. On a third-party network, the usual workaround is
  a mobile hotspot; there is no organization-side fix.
- Note:   one-way audio specifically points at asymmetric NAT or a firewall permitting outbound but
  not the return flow.

### Symptom: calls drop after a fixed interval (e.g. ~30 or ~60 seconds)
- Likely cause: a firewall UDP state timeout expiring mid-call, or a signalling keepalive being
  dropped.
- Check:  pfSense state table timeouts for UDP
- Fix:    [FILL IN: the UDP state timeout configured on pfSense, and whether it has been adjusted for voice traffic].

### Symptom: works off-network, fails on the corporate LAN (or the reverse)
- Likely cause: split-horizon DNS mismatch — one of the two zones is missing the record.
- Check:  run the same `dig` for the SRV record from an internal client and from an external
  resolver; compare the answers
- Fix:    correct whichever zone is wrong. Internal zone lives on [FILL IN: internal DNS servers], public zone at [FILL IN: DNS provider].

### Symptom: certificate warning, or sign-in fails with a trust error
- Likely cause: the edge certificate does not match the SRV target name, or the issuing CA is not
  trusted by the client.
- Check:  `openssl s_client -connect edge.carrier.example.net:8443 -servername edge.carrier.example.net` &nbsp;# shows the presented cert and chain
- Fix:    the certificate is the carrier's to renew — raise a ticket. If the CA is simply missing from
  organization-managed machines, deploy the root via [FILL IN: cert deployment mechanism — GPO, Munki, Intune].

### Symptom: presence/IM works but calling does not (or vice versa)
- Likely cause: the two use different records and ports. Presence uses XMPP, calling uses SIP/UDS.
- Check:  which SRV records resolve; a missing `_cuplogin`/`_xmpp-client` breaks IM only.

### Diagnostics to collect before calling the carrier

Have these ready — the carrier will ask:

    dig SRV _collab-edge._tls.example.com @8.8.8.8              # public discovery record
    dig A edge.carrier.example.net                             # edge resolution
    traceroute edge.carrier.example.net                        # path from the office
    nc -vz edge.carrier.example.net 8443                       # edge reachability

Plus: the affected user's sign-in address, the time of the failure, whether it is on- or
off-network, and the Jabber **Problem Report** from the client (Help > Report a Problem), which
bundles the client logs the carrier needs. Carrier case-opening details: [FILL IN: carrier support contact and account reference].

---

## Security

- **Exposure**: no organization-side service is published for Jabber. All connections are outbound from
  clients to the carrier edge. The attack surface the organization owns is the DNS records and the firewall rules.
- **Auth**: [FILL IN: whether Jabber credentials are AD-backed via the carrier or separate carrier-issued accounts]. If they are AD-backed, a Jabber lockout and an AD lockout are the same event.
- **Certificates**: the edge certificate belongs to the carrier. The organization's responsibility is ensuring managed
  machines trust the issuing CA. Expiry: [FILL IN: edge certificate expiry, if the carrier publishes it].
- **Secrets**: none held in-house for this service. DNS zone credentials are the sensitive item — see
  `Mail & Messaging/email-authentication-spf-dkim-dmarc.md` for who holds them.
- **Hardening note**: permitting the four carrier subnets should be scoped to those subnets and the
  required ports, not a blanket outbound-any rule.

## Monitoring & Alerting

- [FILL IN: whether anything monitors Jabber availability — an SRV record check, a synthetic
  registration, or nothing]. At minimum, an external check that the `_collab-edge` SRV record still
  resolves would catch the most damaging silent failure.
- Carrier-side outages: [FILL IN: whether the carrier provides a status page or proactive outage notification, and where those notices go].

## Disaster Recovery

- **If DNS records are lost**: rebuild from the carrier DNS zone instruction document in this folder
  and from the inventory table above — which is why the target values in the SRV table need to be
  completed. Keep a copy of the current zone export outside the DNS provider:
  [FILL IN: where the example.com zone export is kept].
- **If the carrier platform is down**: there is no organization-side failover. The fallback for staff is
  [FILL IN: fallback communication method during a telephony outage — mobile phones, conference bridge, other].
- **RTO/RPO**: dependent on the carrier. [FILL IN: any SLA in the carrier contract].
- **Escalation**: the sysadmin, then carrier support at [FILL IN: contact], then [FILL IN: internal escalation for a prolonged telephony outage].

## Decisions & History (ADR-lite)

| Date       | Decision / Change                                          | Why / Ticket |
|------------|------------------------------------------------------------|--------------|
| 2024-03-08 | Cisco Jabber requirements and port documentation collected  | Initial deployment planning |
| 2024-03-20 | Carrier service subnets and edge hostname recorded            | Original `_ARCHIVE/superseded-stubs/Carrier Jabber routes.txt` (archived 2026-09-11) |
| 2024-03-21 | Carrier DNS zone creation instructions received (Expressway) | Edge DNS setup |
| 2026-09-11 | Stub promoted to full document                              | Documentation consolidation |
| [FILL IN: date]  | [FILL IN: any subsequent DNS or firewall change affecting Jabber] | [FILL IN: why / ticket] |

## References

Files in this folder (`Telephony & Conferencing/Jabber DNS Setup/`):
- `_ARCHIVE/superseded-stubs/Carrier Jabber routes.txt` (archived 2026-09-11) — original stub, superseded by this document
- `Customer DNS Zone Creation for Expressway.docx` — the carrier's DNS instructions, authoritative for record targets
- `Jabber DNS instructions.pdf` — Carrier/Cisco DNS setup reference
- `Planning Guide for Cisco Jabber 12.6 - Requirements [Cisco Jabber for Windows] - Cisco.pdf`
- `TCP and UDP Port Usage Guide ... Cisco Unified Communications Manager TCP and UDP Port Usage ... .pdf`
- `TCP and UDP Port Usage Guide ... Port Usage Information for the IM and Presence Service ... .pdf`
- Screenshots from the March 2024 setup (see Client Setup section)

Elsewhere in the library:
- `Telephony & Conferencing/Unity Quick Reference Voicemail User Guide.pdf`
- `Telephony & Conferencing/Tip Sheets/` — end-user material
- `Telephony & Conferencing/Placing Conference  Calls.txt`
- the internal KB article "Unblocking an IP from Snort"
- `Networking Guide/pfsense.txt` and `Networking & Switches/` — firewall and edge configuration

---
Template v1 | Maintained by IT Ops | Review annually or after major changes

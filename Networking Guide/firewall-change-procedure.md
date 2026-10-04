> Status: initial draft, 2026-09-11. Contains [FILL IN] markers — see INDEX.md.

# pfSense Firewall Change Procedure

## Overview

This is the procedure for changing firewall policy on the pfSense edge firewall. It covers everything from adding a single rule to opening a NAT port forward to editing a Snort suppression, and it exists because a firewall change is the highest-risk routine change made here: it is fast to apply, it takes effect immediately for new connections, it can silently expose an internal service to the internet, and until now nothing in this library described how to do it safely or how to prove it worked.

The document is written for the person making the change — normally the sysadmin — and for whoever has to review or reverse it afterwards. It assumes familiarity with pfSense navigation; the GUI paths, CLI commands and rule-processing order are already documented in `Networking Guide/Network-Administration-Guide.md` (sections 4.6 and 4.7) and `Networking Guide/pfsense.txt`, and are not repeated here except where the procedure depends on them. What this guide adds is the wrapper around the change: intake, review, snapshot, apply, verify **both directions**, roll back, log.

## Quick Facts

| Field            | Value                                                                                                                                              |
|------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| Owner            | IT Ops (it@example.com)                                                                                                          |
| Environment      | prod — this is the production edge firewall; there is no staging firewall                                                                            |
| Location         | pfSense edge firewall. Web GUI recorded in existing runbooks as `https://fw01.example.com` — [CONFIRM: that this is still the current management name, and whether a second/HA node exists]. Config lives in `/cf/conf/config.xml` on the firewall |
| Access           | Web GUI from the management network or over WireGuard; SSH/console as documented in `Networking Guide/Network-Administration-Guide.md` §4.1. Credentials in 1Password — [FILL IN: 1Password vault and item name for the pfSense admin account] |
| Dependencies     | Management path (WireGuard or management VLAN), DNS for name resolution during testing, the config backup destination, Auto Config Backup if enabled |
| Dependents       | Every network segment and every internet-facing service — data, wiki, sign, appdb2, appdb1, erpdb, prod01, graylog01, files, forums, auth2, auth4, mail; the partner IPsec tunnel; WireGuard remote access; 802.1x/RADIUS traffic to AD; inter-VLAN routing for all staff, guest and server segments |
| Last reviewed    | 2026-09-11 (initial draft — UNVERIFIED)                                                                                                             |

## How It Works

Three things determine whether a packet gets through, and a change procedure has to respect all three.

**Rule evaluation order.** pfSense evaluates, in order: Ethernet rules, Outbound NAT, Inbound NAT, automatic/internal rules, then user rules — Floating first, then interface-group rules, then the interface tab — top-down, **first match wins** — and finally automatic VPN rules. Within an interface tab the first rule that matches ends evaluation. This is why "I added the rule and nothing changed" is almost always a rule-order problem rather than a rule-content problem: something above it already matched. Floating rules run after NAT, so on WAN they see post-NAT addresses.

**State.** pf is stateful. A rule change applies to *new* connections. Existing connections already have entries in the state table and keep flowing until those states expire or are killed. A block you just added will appear not to work against a session that was already open. This cuts both ways — it is also why an emergency block needs the states killed explicitly, not just the rule added.

**The layers above and below pfSense.** A packet reaching an internal web service may also pass the Barracuda Web Application Firewall, and may be dropped by Snort's block table on pfSense or by fail2ban on the host (currently mail.example.com). A pfSense rule change cannot fix a Barracuda policy and will not release a Snort-blocked address. Establishing *which* layer dropped the traffic is part of the change, not an afterthought — see "Snort and fail2ban Interaction" below.

Where the configuration lives:

| Item | Where |
|------|-------|
| Running ruleset | In memory; view with `pfctl -sr` |
| Generated ruleset | `/tmp/rules.debug` on the firewall |
| Persistent config | `/cf/conf/config.xml` |
| Local config history | Diagnostics → Backup & Restore → Config History (default 30 prior configs, with a diff view) |
| Firewall log | Status → System Logs → Firewall, or `clog /var/log/filter.log` |
| Snort block table | Services → Snort → Blocked, or `pfctl -T show -t snort2c` |

---

## What Counts as a Firewall Change

Anything in the list below is a firewall change and is covered by this procedure. The second column is the default handling.

| Change type | Handling |
|---|---|
| New firewall rule permitting traffic (any interface, any direction) | **Reviewed** |
| Editing an existing permit rule to widen source, destination, port or protocol | **Reviewed** |
| Any change to a WAN-tab rule or a Floating rule | **Reviewed** |
| New or edited NAT entry — port forward, 1:1, or Outbound NAT | **Reviewed** |
| Creating an alias, or adding a *host or network* to an alias used by a permit rule | **Reviewed** |
| Interface assignment, VLAN interface creation, or interface IP change | **Reviewed** |
| VPN policy — IPsec Phase 2 selectors, WireGuard AllowedIPs, mobile client network list | **Reviewed** |
| Enabling/disabling a Snort interface, changing rule categories, or adding a suppression | **Reviewed** |
| Rule reordering that moves a permit rule above a block rule | **Reviewed** |
| New **block** or **reject** rule that narrows access | Routine |
| Disabling or deleting a rule to remove access | Routine |
| Adding an address to a blocklist alias | Routine |
| Removing a single IP from the Snort block table (an unblock) | Routine — see the existing unblock procedure |
| Removing a single IP from a fail2ban jail | Routine — see the existing unban procedure |
| Toggling logging on an existing rule | Routine |
| Adding a DHCP static mapping | Routine (not strictly a firewall change, but record it if a rule depends on the address) |

The dividing line is deliberate and simple: **anything that lets more traffic through gets reviewed; anything that lets less traffic through is routine.** Routine changes still get logged (see "Change Logging"), they just do not wait for a second pair of eyes.

Two cautions on that line. First, a *narrowing* change can still cause an outage — deleting a rule that something depended on is routine by this definition but is not risk-free, so the verification step applies to routine changes too. Second, adding a host to an existing alias looks trivial and is not: an alias referenced by a permit rule inherits that rule's full permissions, so adding an address to it is functionally the same as writing a new permit rule for that address.

**Review, here, means a second person reads the change before it is applied.** With a single sysadmin that is frequently not possible. [CONFIRM: proposed default — where no second reviewer is available, the sysadmin completes the review checklist below in writing as part of the change record, and the record is sent to [FILL IN: who receives firewall change records for after-the-fact visibility — e.g. IT manager] within one business day. A written self-review that another person can later read is the minimum; an unwritten one is not a review.]

---

## Request Intake

Firewall changes arrive as requests: a vendor needs access, a new service needs a port, a user cannot reach something. A request that does not carry the information below cannot be reviewed, and should be sent back rather than guessed at.

**A requester must supply:**

1. **Source** — the specific host, subnet or user group that needs access, with addresses. "The office" is not a source; a named VLAN or subnet is.
2. **Destination** — the specific host and service, by hostname and address.
3. **Protocol and port(s)** — exact. "It uses a few ports" means the request is not ready.
4. **Direction** — who initiates the connection. This decides which interface tab the rule belongs on and whether NAT is involved at all.
5. **Business reason** — what breaks without it, and who is affected.
6. **Duration** — permanent, or until a date. Temporary access must carry an expiry date, because temporary rules with no expiry are the main source of the orphaned-rule problem described later in this document.
7. **Requester and approver** — who asked, and who is accountable for the access being appropriate.
8. **Vendor/third-party specifics**, if applicable — the vendor's source addresses, a contact, and whether the access is for a defined engagement window.

[CONFIRM: proposed default — firewall change requests are raised as a ticket and the ticket number becomes the reference in the change record. Where no ticketing system is in use, an email to it@example.com containing the eight items above serves the same purpose and is filed with the change record.]

[CONFIRM: proposed default lead times — Normal firewall changes: three business days from a complete request to the change window. Emergency changes: immediate, via the out-of-band path below. Anything touching the WAN tab or a NAT port forward: scheduled, never done ad hoc during business hours unless it is an emergency.]

A note on scope creep: requesters routinely ask for more than they need, usually because they do not know exactly what the application requires. The correct response is not to grant the wide rule and narrow it later — that narrowing never happens. It is to grant the narrow rule, enable logging on it, and use the firewall log to discover what else the application actually attempts.

---

## Review Checklist — Before Touching Anything

Work through this in order. Record the answers; they become the justification in the change record.

**1. Does an existing rule already cover this?**
Search the ruleset before adding to it. A duplicate rule is worse than no rule: it is a second thing to find when you later try to remove the access.

```
pfctl -sr | grep -i <destination-ip-or-port>   # does anything already permit this?
```

Also check aliases — the traffic may already be permitted through an alias whose name does not mention the host.

**2. Least privilege.** Is this the narrowest rule that solves the problem? Test each field:

- Source: a single host rather than a subnet, a subnet rather than `any`.
- Destination: a single host rather than a subnet, a subnet rather than `any`.
- Port: the specific port rather than a range, a range rather than `any`.
- Protocol: TCP or UDP specifically, not `any`. (The exception documented in `Networking Guide/Network-Administration-Guide.md` §3.2 is worth remembering in reverse: scoping an IPsec-interface rule to TCP only silently breaks ICMP and DNS across the tunnel. Narrow deliberately, not reflexively.)

**3. Source and destination specificity.** `any` on either side of a permit rule requires a written justification in the change record. `any` on *both* sides of a permit rule is not a rule, it is the absence of a firewall, and should not be applied.

**4. Alias or literal?** Anything likely to recur — a management host set, a vendor's address range, a group of servers — belongs in an alias. Anything one-off can be literal. Where an alias is used, confirm what *else* references that alias before editing it:

```
pfctl -sr | grep -i <alias-name>               # what rules depend on this alias
```

**5. Logging on or off?** Default is **on** for any newly added permit rule, and on for any block rule you expect to need evidence from. Leave it on for at least the first review cycle after the change; the log is how you discover whether the rule is used at all, which is how orphaned rules get identified later. Turn logging off only when a rule is high-volume enough that it drowns the log, and record why.

**6. Rule order — where must it sit?** This is the step most often skipped and the one that most often causes the "no effect" failure. Determine:

- Which tab does it belong on? Rules go on the interface where the traffic **enters** the firewall.
- Is there a block rule above it that will match first? If yes, the new rule must sit above that block rule, and moving it above a block rule is itself a reviewed change.
- Is there a broader permit rule above it that already matches? If so, the new rule is dead code — deal with the existing rule instead.
- Is there a Floating rule that will match before any interface rule? Floating rules are evaluated before interface rules and are easy to forget.
- Does the intended position accidentally place the new permit above a block that is protecting something else?

Write down the intended position — "immediately above rule N, the block to [FILL IN]" — before you open the editor.

**7. Blast radius.** If this rule is wrong in the most permissive plausible way, what becomes reachable? Answer that explicitly. A port forward typed with the wrong internal address exposes whatever *is* at that address.

**8. Is pfSense even the right place?** If the traffic is HTTP/HTTPS to a service behind the Barracuda Web Application Firewall, the policy may belong on the Barracuda instead — see `Security & Hardening/barracuda_web_application_firewall_best_practices_guide.pdf`. If the problem is a blocked source address rather than a missing permission, it is a Snort or fail2ban unblock, not a rule change.

**9. Reversibility.** Can this be undone by disabling one rule? If the change requires editing several rules, an interface, and a NAT entry together, it is not one change — it is several, and the rollback plan has to account for all of them.

---

## Pre-Change Snapshot

Never change firewall policy without a restore point and a written record of the current state. Both, every time, including for routine changes.

**Take a config backup.**

GUI: Diagnostics → Backup & Restore → Backup/Restore tab → Backup area: "All" → Download. Save it with a name that ties it to the change:

```
config-<hostname>-<timestamp>.xml              # pfSense names the download this way
```

[CONFIRM: proposed default — rename the downloaded file to `CHG-YYYY-NNN-pre.xml` and store it at [FILL IN: path on the Systems Admin share or other agreed location for pre-change firewall configs]. Retain for 90 days.]

CLI equivalent, if you are already on the box:

```
cp /cf/conf/config.xml /root/pre-CHG-YYYY-NNN.xml   # local copy before the change
```

pfSense also keeps a **Config History** automatically (Diagnostics → Backup & Restore → Config History, default 30 prior configurations, with a diff view between any two). That history is the fastest rollback path and it costs nothing — but it is on the firewall itself, so it does not help if the firewall is what fails. A downloaded copy is the one that survives. Auto Config Backup (ACB) is free on both CE and Plus and keeps up to 100 encrypted backups off-box; confirm it is enabled — [FILL IN: is ACB enabled on this firewall, and under which Netgate account?]

**Record current state.** Capture enough to compare against afterwards:

```
pfctl -sr > /tmp/rules-before.txt               # full ruleset, in evaluation order
pfctl -sn > /tmp/nat-before.txt                 # NAT rules
pfctl -si                                       # filter status and counters
pfctl -ss | wc -l                               # state table size, as a rough baseline
```

And, in the change record, note in plain words: which interface tab, the rule numbers immediately above and below the insertion point, and whether the target service is currently reachable from the intended source (it should not be — if it already works, the rule is unnecessary and the request needs rechecking).

**Confirm you have a way back in.** If the change touches the interface, VLAN or rule set that carries your own management session, confirm an alternate path — console, or a second route — before applying. `Networking Guide/Network-Administration-Guide.md` §7.3 lists the commands that will cut off your only path in.

---

## Applying the Change

1. Make **one** change at a time. If a request needs a rule and a NAT entry, apply them as one deliberate pair, verify, then move on — do not batch three unrelated requests into one Apply.
2. Enter the rule with logging on (per the review checklist) and a **description** that includes the change ID, the ticket reference, and the expiry date if temporary. The description field is the only place this context survives; a rule described as "allow 443" is an orphan the day after it is created. Use a consistent form:
   `CHG-2026-0NN | <ticket> | <requester> | purpose | expires YYYY-MM-DD or PERMANENT`
3. Place it at the position determined during review. Verify the position on the rules list before applying — pfSense adds new rules at the bottom of the tab by default, which is usually not where you decided it belongs.
4. Click **Apply Changes**. Rules take effect immediately for new connections.
5. Do not close the browser tab or the SSH session you came in on until verification is complete.

Applying from the CLI (`pfctl -f /tmp/rules.debug`) reloads the ruleset that the GUI generated. It does not create rules and it is not the way to make a change — it is a recovery command for when the ruleset in memory does not match the configuration.

---

## Verification — Test the Allow Path AND the Deny Path

**This is the step that makes the procedure worth following, and it is the step most commonly done wrong.**

A rule that permits more than intended verifies perfectly if you only test the happy path. If you asked for "host A to host B on 443" and you accidentally wrote "any to host B on 443", then testing from host A succeeds — exactly as it would have if the rule were correct. The test tells you nothing about the fault, and you will now believe the change was successful. The same trap applies to a port range typed instead of a single port, an alias that contains more members than you thought, and a destination `any` left in place because the form defaulted to it.

So every firewall change gets **two** verification tests:

**1. Allow path — the traffic that should now work, does.**
From the intended source, to the intended destination, on the intended port.

```
nc -vz <destination> <port>                      # from the intended source host
curl -sS -o /dev/null -w '%{http_code}\n' https://<destination>/   # for an HTTP service
```

Expect success. Then confirm the *firewall* is what allowed it — check the rule's counters moved, or find the pass entry in Status → System Logs → Firewall filtered to that rule. A successful connection that did not increment your new rule means something else permitted it, and your rule is doing nothing.

**2. Deny path — the traffic that should still be blocked, is.**
This is the test that catches an over-permissive rule. Pick at least the following, and pick them *before* you apply the change so you are not tempted to skip them:

- The same destination and port, from a source that should **not** have access. Expect a timeout or a reject.
- A *different* port on the same destination, from the permitted source. Expect a block — unless the request genuinely required a range.
- The same source and port to a *different* destination in the same subnet. Expect a block. This one catches a destination that was left as a subnet or as `any`.

```
nc -vz -w 3 <destination> <port>                 # from a host that should NOT be permitted
nc -vz -w 3 <destination> <other-port>           # permitted source, port that should stay closed
nc -vz -w 3 <other-host-same-subnet> <port>      # permitted source, neighbouring host
```

Expect failure on all three, and confirm the block appears in the firewall log. **A deny test that "fails to connect" for an unrelated reason — the service is not listening, the host is down, DNS did not resolve — is not a passed deny test.** Confirm the block in Status → System Logs → Firewall before recording it as verified. If there is no block entry, you have not proved the firewall denied it.

For a NAT port forward, add a third test from **outside** the network: confirm the forward reaches the intended internal host, and confirm that the neighbouring internal addresses are not also reachable.

**3. Confirm nothing else broke.** Check Status → System Logs → Firewall for a spike in blocks immediately after the change, and check the services on the affected segment. A permit rule inserted too high in the list can shadow a block rule that something else relied on.

Record all of this in the change record: what you tested, from where, and what you observed. "Verified working" is not a verification record.

---

## Rollback

**Rollback window.** [CONFIRM: proposed default — a firewall change is provisional for 60 minutes after apply. Within that window, if anything unexplained appears, revert first and diagnose afterwards. Do not close the change until the window has passed and the affected service has been confirmed working.]

Rollback options, fastest first:

1. **Disable the rule.** Uncheck it, Apply. This is instant and it is the right first move for a change that broke something — it removes the new behaviour without touching anything else. Disable rather than delete, so the rule is still there to examine.
2. **Restore from Config History.** Diagnostics → Backup & Restore → Config History → select the pre-change entry → Revert. Use the diff view first to confirm you are reverting only what you intend. This restores the whole configuration, so it also discards any other change made since that point — check the history for other entries before using it.
3. **Restore the downloaded pre-change config.** Diagnostics → Backup & Restore → Restore → upload the `CHG-...-pre.xml` file. Slower, and it may restart services, but it works when the config history is not usable.
4. **Console option 15, "Restore recent configuration"**, when the GUI is unreachable.

**Kill the states.** A rollback that only removes the rule leaves established connections running through the state table. If the change permitted traffic that should not be flowing, remove the rule *and* kill the states:

```
pfctl -k <source-ip>                             # kill all states from this source
pfctl -k <source-ip> -k <destination-ip>         # kill states between this pair only
```

Do not clear the whole state table (`pfctl -F state`) on a production firewall to fix one rule — it drops every session on the box, including your own management session and the partner tunnel.

**After a rollback**, the change record is completed with outcome "Failed — rolled back" and the reason. A rolled-back change is a normal outcome, not a failure of process; an undocumented rolled-back change is.

---

## Change Logging

Every firewall change gets a record in `SysAdmin Procedures/Change_Management_Log.txt`, using the change record template already defined there (CHANGE ID `CHG-YYYY-NNN`, categories Standard / Normal / Emergency, with the pre-change state, risk assessment, rollback plan, verification and outcome fields).

Note that the `SysAdmin Procedures/` set is currently unfilled boilerplate — the example record in it refers to `example.com` hosts. The *template* is sound and is what firewall changes should use; the *contents* are not yet a real log. Filling that file is a prerequisite for this procedure being followed end to end.

Map firewall work onto that template's categories as follows:

| This procedure's term | Change_Management_Log.txt category |
|---|---|
| Routine (narrowing change, unblock, logging toggle) | **Standard** — self-approved, still recorded |
| Reviewed (any widening change) | **Normal** — recorded with the review checklist answers |
| Out-of-band / emergency | **Emergency** — recorded within [CONFIRM: 24 hours] |

Firewall-specific fields to add into the record's free-text areas, because the generic template does not prompt for them:

- Interface tab and rule position (above/below which rules).
- The rule as written: source, destination, protocol, port, action, logging state.
- Alias names touched, and what else references them.
- Both verification tests — allow path and deny path — with the observed result of each.
- Expiry date, for temporary access.
- The filename and location of the pre-change config backup.

The pfSense Config History is a useful cross-check but is **not** the change log: it records that the configuration changed and lets you diff it, but it does not record who asked for it, why, what was tested, or when it expires.

---

## Emergency / Out-of-Band Changes

An emergency firewall change is one made to stop active harm or restore a down service, where following the normal path would make the outcome materially worse. Blocking an address that is actively attacking a service is an emergency change. "The vendor is on the phone and wants it now" is not.

**Emergency path:**

1. Take the config backup anyway. It is one click and it is the thing you will want in twenty minutes.
2. Make the narrowest change that stops the harm. For an active attack that is usually a block, not a permit — and blocks are routine by this procedure's own definition, so the emergency path is mostly used for emergency *permits*, which deserve correspondingly more suspicion.
3. Kill the relevant states if you are blocking something already connected (`pfctl -k <ip>`).
4. Verify — both paths, same as any other change. Under time pressure the deny test is the one people skip, and under time pressure is exactly when over-permissive rules get written.
5. Note the time, the rule, and what prompted it, immediately, in whatever form is available. Two lines in a text file at 2am is fine.

**Retroactive review is mandatory.** [CONFIRM: proposed default — within one business day of an emergency change: complete the full change record; walk the review checklist against the rule as applied; decide explicitly whether the rule stays, is narrowed, or is removed; and if it stays, set an expiry or mark it permanent with justification.] The failure mode here is well understood: emergency rules are written wide because narrowing takes time, and then they stay wide forever because nobody comes back to them. The retroactive review is the mechanism that prevents that, and it only works if it is scheduled rather than intended.

Emergency changes made from the console during an outage — where the GUI is unreachable — should be redone properly through the GUI once access is restored, so that the configuration and the config history reflect reality.

---

## Periodic Rule Review

Firewall rulesets accumulate. The two specific problems are:

**Orphaned rules** — permit rules whose purpose no longer exists: the vendor engagement ended, the host was decommissioned, the temporary access was never temporary. These are a live exposure, not just clutter. The strongest single cause of orphaned firewall rules is a host that was retired without anyone walking back its dependencies — which is why `Linux & Servers/server-build-standard.md` includes a decommissioning procedure that explicitly requires firewall rules to be removed.

**Shadowed rules** — rules that can never match because something above them matches first. Shadowed rules are dangerous in a subtler way: they make the ruleset lie. Someone reads the rule and believes the access is controlled by it, when in fact the behaviour is set by the rule above.

[CONFIRM: proposed default cadence — a full rule review quarterly, and a targeted review of WAN-tab and NAT rules monthly. Budget half a day for the quarterly pass. Record the review itself as a Standard change so there is evidence it happened.]

**Review procedure:**

1. Export the current ruleset for offline reading:

```
pfctl -sr > rules-review-$(date +%F).txt         # evaluation order, as pf sees it
pfctl -sn > nat-review-$(date +%F).txt           # NAT rules
```

2. **Find rules with no traffic.** Rule counters are the evidence. A permit rule that has passed nothing since the last review is a candidate for removal.

```
pfctl -vsr | grep -A1 'pass'                     # rules with their packet/byte counters
pfctl -z                                          # zero counters — do this at the START of a review period
```
Zeroing counters at the start of a review window and reading them at the end is the cleanest way to find dead rules. [CONFIRM: proposed default — zero the counters at the start of each quarter as part of the review.]

3. **Check every description.** Any rule whose description does not identify a change ID, a purpose and an owner is by definition unaccounted for. Either establish what it is for and fix the description, or propose it for removal.

4. **Check expiry dates.** Every rule marked with an expiry that has passed is removed, or explicitly renewed with a new date and a reason.

5. **Look for shadowing.** Read each interface tab top to bottom and, for each rule, ask whether anything above it already matches the same traffic. Pay particular attention to rules that were moved during a past change.

6. **Reconcile against reality.** Cross-check destination addresses against `Networking Guide/ipam-vlan-topology-reference.md` and the host inventory in `SysAdmin Procedures/Inventory_Asset_Reference.txt`. A rule pointing at an address that no longer has a host is an orphan; a rule pointing at an address that now has a *different* host is worse.

7. **Check aliases for drift.** List each alias's members and confirm each member is still current. Aliases grow and never shrink unless someone makes them.

8. **Remove by disabling first.** Disable candidate rules, leave them disabled for [CONFIRM: one review period — e.g. 30 days], then delete. Deleting straight away turns a cleanup into an incident when you are wrong.

Record the review outcome: rules removed, rules renewed, rules whose purpose could not be established. That last category is the one to escalate.

---

## Snort and fail2ban Interaction

Three separate mechanisms can drop a packet here, and they are not the same thing. Confusing them wastes time and produces changes that do not help.

**pfSense firewall rules** — policy. Deterministic, visible in the ruleset, changed via this procedure.

**Snort** — IDS on pfSense. When Snort blocks, it adds the offending address to a pf table (`snort2c`); the block is invisible in the rules list because it is not a rule. The classic symptom, recorded in the internal KB article "Unblocking an IP from Snort": a site or service is unreachable on both the wired and ORG_Wireless networks but works fine on the Guest network. That pattern means Snort, not a firewall rule — do not go looking for a rule to change.

Unblocking is already documented and should not be duplicated here. In short: Services → Snort → Blocked tab, find the address, remove it with the red X; the "Clear" button removes all blocks at once in an emergency, and known-bad addresses repopulate quickly. The full procedure, including using `dig` to find the address of the service being blocked, is in that PDF. See also Internal KB article "Large Meeting IP Whitelist Prep", which covers whitelisting ahead of a large meeting and enumerates the places an address can be blocked.

From the CLI:

```
pfctl -T show -t snort2c                         # list addresses Snort is currently blocking
```

Snort changes that *are* covered by this procedure: enabling or disabling Snort on an interface, changing enabled rule categories, and adding suppressions or pass-list entries. A suppression is a permanent reduction in detection coverage and should be reviewed like a permit rule — narrowest possible suppression, documented reason, expiry where appropriate. A one-off unblock is routine.

**fail2ban** — host-based, currently running on mail.example.com. It bans by adding a route, so banned addresses appear as unreachable in the host's routing table (`ip r`). It is entirely independent of pfSense: a pfSense rule change will not release a fail2ban ban, and a fail2ban unban will not help if pfSense or Snort is also blocking. The procedure is in Internal KB article on fail2ban filters and unbanning and Internal KB article on unbanning a user via Graylog: identify the jail in Graylog (Fail2Ban tab), then on the host:

```
sudo fail2ban-client status <jail>               # confirm the jail and see banned IPs
sudo fail2ban-client set <jail> unbanip <ip>     # release one address
```

**Triage order when something is unreachable and you suspect a block:**

1. Is it blocked on the Guest network too? If it works on Guest, suspect Snort.
2. Check the Snort Blocked list for the address.
3. Check Status → System Logs → Firewall for an explicit block, which points at a rule.
4. If the destination is mail.example.com, check the fail2ban jails and Graylog.
5. If the destination is a web service behind the Barracuda WAF, check the Barracuda before changing anything on pfSense.
6. Only then consider a rule change.

---

## Troubleshooting

### Symptom: the change had no effect — traffic still blocked
- Likely cause: rule order. Something above it matched first — a Floating rule, an interface-group rule, or a broader block higher in the same tab.
- Check: `pfctl -sr` and read the ruleset in evaluation order; find every rule that matches the traffic and note which is first.
- Check: Status → System Logs → Firewall, filtered to the source and destination — the log entry names the rule that acted.
- Check: did you put the rule on the interface where the traffic *enters* the firewall? A rule on the wrong tab is never evaluated for that traffic.
- Check: is the traffic even reaching the firewall? Rule out the physical and switch layers first — `Networking Guide/Network-Administration-Guide.md` §3.1.
- Fix: move the rule above the shadowing rule (and treat the move as a reviewed change), or correct the interface.

### Symptom: the change had no effect — traffic still permitted after adding a block
- Likely cause: existing states. pf applies rules to new connections; established sessions keep flowing.
- Check: `pfctl -ss | grep <ip>` — is there still a state entry for the connection?
- Fix: `pfctl -k <source-ip>` to kill states from that source, or `pfctl -k <src> -k <dst>` for the pair. Do not flush the entire state table on prod.

### Symptom: the change broke something unrelated
- Likely cause: the new rule sits above a rule that other traffic depended on, or an alias you edited is referenced by more rules than you realised.
- Check: `pfctl -sr | grep -i <alias-name>` — every rule that uses the alias.
- Check: diff the ruleset against the pre-change capture: `diff /tmp/rules-before.txt <(pfctl -sr)`.
- Check: Diagnostics → Backup & Restore → Config History diff view, between the pre-change and current configuration.
- Fix: disable the new rule first, confirm the unrelated breakage clears, then re-approach the change.

### Symptom: rule looks correct, traffic passes, but the wrong source can also reach it
- Likely cause: the rule is more permissive than intended — `any` left in a source or destination field, a port range instead of a port, or an alias with unexpected members.
- This is the failure the two-path verification exists to catch. If it surfaced in production rather than in testing, the deny test was skipped.
- Check: re-read the rule field by field against the request. Expand every alias and read its members.
- Fix: narrow the rule; then re-run both verification paths.

### Symptom: works from one host, not from another on the same subnet
- Likely cause: asymmetric routing, or a host-level firewall on the failing host.
- Check: is there more than one path between the two networks — a second router, a VPN carrying the same subnet, an interface with an overlapping route? If the reply leaves by a different interface than the request arrived on, pf sees a packet with no matching state and drops it.
- Check: `netstat -rn` on the firewall, and the default gateway on the destination host. A destination host whose default gateway is not pfSense sends replies out a path that never re-enters the tunnel or the state.
- Check: packet capture on both interfaces: Diagnostics → Packet Capture, or `tcpdump -ni <if> host <ip>`.
- Fix: correct the routing or the host's gateway. State policy relaxations exist for genuinely asymmetric designs but they weaken the firewall — treat them as a last resort and a reviewed change.

### Symptom: connection works, then stops after a while
- Likely cause: state expiry on a long-idle connection, or an MTU/fragmentation problem on a VPN path rather than a rule problem.
- Check: for VPN traffic that works for small transfers and hangs on large ones, this is the MSS/fragmentation case documented in `Networking Guide/Network-Administration-Guide.md` §3.2 — not a firewall rule.

### Symptom: cannot reach the pfSense GUI after a change
- Likely cause: the change affected the management path, or sshguard has blocked you after repeated failed logins.
- Check: Diagnostics → Tables → sshguard, or at the shell `pfctl -T flush -t sshguard`.
- Fix: console access (serial 115200 8N1) and console option 15 to restore the recent configuration. See `Networking Guide/Network-Administration-Guide.md` §6 for the full console menu.

### Symptom: ruleset in memory does not match the configuration
- Likely cause: an Apply that did not complete, or a manual edit.
- Check: `pfctl -sr` against what the GUI shows.
- Fix: `pfctl -f /tmp/rules.debug` reloads the generated ruleset. If they still disagree, re-apply from the GUI so the configuration regenerates `/tmp/rules.debug`.

---

## Security

- **Exposure:** the firewall is the internet boundary. Every change here is a change to the organization's attack surface, which is why widening changes are reviewed and narrowing changes are not.
- **Management access:** the GUI should not be reachable from the internet on any port. Management is from the internal management network or over WireGuard. See `Networking Guide/Network-Administration-Guide.md` §4.8.
- **Auth:** [FILL IN: how pfSense admin authentication is configured — local accounts, or RADIUS/LDAP against AD]. MFA on the WebGUI comes via an external RADIUS or LDAP authentication server, not a native TOTP toggle.
- **Accountability:** individual admin accounts, not a shared `admin` login, are what make the config history and the system log usable as an audit trail. [FILL IN: confirm whether individual accounts are in use]
- **Secrets:** never record credentials, PSKs or WireGuard private keys in a change record or in this document. See `Security & Hardening/secrets-management-guide.md`.
- **Snort suppressions** reduce detection coverage permanently. Review them alongside permit rules.

## Monitoring & Alerting

- Firewall logs should reach Graylog so that blocks and passes are searchable outside the firewall's own limited local buffer. [FILL IN: is pfSense currently configured for remote syslog to graylog01.example.com? Status → System Logs → Settings → Remote Logging]
- What to watch after a change: block-rate on the affected interface, the new rule's counters, and the service's own logs.
- [CONFIRM: proposed default alerting — an alert on any configuration change to the firewall outside a change window, built from the pfSense system log in Graylog. This is the single highest-value firewall alert and does not exist yet.]
- See `SysAdmin Procedures/monitoring-alerting-guide.md` for the wider monitoring gap analysis; the short version is that the organization has centralised logging, not centralised monitoring, so post-change monitoring is currently a human reading logs.

## Disaster Recovery

- **RTO/RPO for firewall configuration:** [CONFIRM: proposed default — RPO is the last config backup, which should be no older than the last change; RTO is the time to load a config onto replacement hardware, target four hours.]
- **Rebuild path:** install pfSense on replacement hardware at the same version, restore `config.xml`, verify interface assignments (they map by NIC and will need reassigning if the hardware differs), reinstall packages (Snort, and anything else in System → Package Manager), then verify the ruleset against the last known-good export.
- **The backup that matters is the off-box one.** Config History and ACB both help, but a downloaded `config.xml` stored outside the firewall is what survives a hardware loss. [FILL IN: where firewall config backups are stored, and whether they are included in the Retrospect backup set]
- **Escalation:** [FILL IN: escalation contact and Netgate support entitlement, if any, when the sysadmin is unavailable]

## Decisions & History (ADR-lite)

| Date       | Decision / Change | Why |
|------------|-------------------|-----|
| 2026-09-11 | Procedure created | No documented firewall change process existed; highest-risk routine change |
| 2026-09-11 | Widening changes reviewed, narrowing changes routine | Simple, memorable line that matches actual risk |
| 2026-09-11 | Two-path verification (allow **and** deny) made mandatory | An over-permissive rule passes a happy-path test; only the deny test catches it |
| 2026-09-11 | Rule descriptions must carry change ID, purpose and expiry | Descriptions are the only in-band record; undescribed rules become orphans |
| [FILL IN] | [FILL IN: record each material change to this procedure] | [FILL IN] |

## References

- `Networking Guide/Network-Administration-Guide.md` — pfSense rule/NAT mechanics, rule processing order, console menu, backup/restore, troubleshooting
- `Networking Guide/pfsense.txt` — pfSense GUI paths and CLI cheat sheet
- `Networking Guide/ipam-vlan-topology-reference.md` — VLANs, subnets and addresses referenced by rules
- `Networking Guide/dns-dhcp-administration-guide.md` — name resolution behind rule destinations
- `Networking Guide/the-pfsense-documentation.pdf` — the full Netgate manual
- Internal KB article "Large Meeting IP Whitelist Prep" — whitelisting, and every place an address can be blocked
- the internal KB article "Unblocking an IP from Snort" — Snort unblock procedure
- Internal KB article on fail2ban filters and unbanning — fail2ban jails and unban procedure
- Internal KB article on unbanning a user via Graylog — identifying a jail from Graylog
- `Security & Hardening/barracuda_web_application_firewall_best_practices_guide.pdf` — WAF policy, the other place HTTP traffic is filtered
- `SysAdmin Procedures/Change_Management_Log.txt` — change record template and log (currently unfilled boilerplate)
- `SysAdmin Procedures/Inventory_Asset_Reference.txt` — intended source of truth for hosts referenced by rules (currently unfilled boilerplate)
- `Linux & Servers/server-build-standard.md` — build and decommissioning standard; decommissioning is where stale firewall rules are prevented

---
*Tier 3 document. Review annually or after any material change to the procedure.*

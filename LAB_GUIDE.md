# Sophos Firewall Lab Guide

**Created:** 18 October 2026  
**Updated:** 11 November 2026  
**Purpose:** Build a hands-on Sophos Firewall lab for learning network
routing, firewall rules, NAT, VPNs, and Security+ concepts.

This guide converts the lab notes into GitHub-friendly Markdown. It retains the
recorded network values and test outcomes. Configuration screens and labels
can differ by Sophos or VMware version; verify the live configuration before
relying on a table as an export of the appliance.

## Contents

1. [VMware network configuration](#1-vmware-workstation-network-configuration)
2. [Adding the third network adapter](#2-how-we-added-the-third-network-adapter)
3. [Desktop firewall rules](#3-sophos-firewall-rules)
4. [Desktop NAT rule](#4-sophos-nat-rule)
5. [Verification performed](#5-verification-performed)
6. [Troubleshooting desktop internet access](#6-troubleshooting-log)
7. [DMZ build: Debian and Nginx](#7-dmz-build-debian-and-nginx)
8. [DMZ firewall policy](#8-dmz-firewall-policy)
9. [DMZ isolation testing](#9-dmz-isolation-testing)
10. [DMZ troubleshooting and lessons](#10-dmz-troubleshooting-log-and-lessons)
11. [Final rule order](#11-final-rule-order)
12. [Task 2.4: custom service and IP host group](#12-task-24--custom-service--ip-host-group)
13. [SSL VPN remote access lab](#13-sophos-firewall--ssl-vpn-remote-access-lab)
14. [Completion summary](#14-completion-summary-and-follow-up)

## How the lab works

Each VMware network is a separate virtual segment. Sophos has an interface on
each segment and acts as the router between them. A client uses the Sophos
interface on its own segment as its gateway when it needs to reach another
network.

When a packet enters Sophos, routing determines where it can be forwarded and
the firewall policy determines whether the connection is allowed. Applicable
firewall rules are checked in order; a broad rule placed too early can permit
traffic that a narrower rule was intended to control. NAT is evaluated
separately. For typical outbound access, source NAT (MASQ) changes a private
source address to an address appropriate for the outgoing interface. Stateful
tracking lets replies return through the existing connection.

Traffic to a service hosted by Sophos itself is different from traffic passing
through Sophos. Device Access / Local ACL settings govern services on the
firewall, while firewall rules govern routed traffic between network zones.
The DMZ's separate subnet helps organize the topology, but policy between the
DMZ and LAN is what enforces isolation.

## 1. VMware Workstation network configuration

VMware Workstation provides several virtual network types:

- **Host-only (VMnet1):** Private network for the host and connected VMs.
- **NAT (VMnet8):** Lets a VM reach external networks through the host's
  network connection.
- **Bridged (VMnet0):** Connects a VM's virtual adapter to a selected physical
  network adapter.

These are standard VMware networking modes. See Broadcom's VMware networking
documentation.

### VMware network mapping

| VMware network | Sophos interface | Purpose |
|---|---|---|
| VMnet1 — Host-only | Adapter 1 → Port1 | Existing private LAN |
| VMnet8 — NAT | Adapter 2 → Port2 | WAN / internet access |
| VMnet0 — Bridged to laptop Ethernet | Adapter 3 → Port3 | Direct desktop-to-firewall test network |
| VMnet2 — Host-only, no DHCP or host adapter | Adapter 4 → Port4 | Isolated DMZ for the Debian web server |

The firewall needs separate virtual interfaces for different network
segments. Port2 acts as WAN, Port3 provides a LAN connection for the desktop
attached to the laptop, and Port4 is the isolated DMZ interface added later.

## 2. How we added the third network adapter

The third adapter was added to the Sophos VM in VMware Workstation:

1. Power off the Sophos VM before changing its virtual hardware.
2. Select the Sophos VM and open **VM → Settings**.
3. On the **Hardware** tab, click **Add**.
4. Select **Network Adapter**, then click **Finish**.
5. Select the new adapter and configure it as **Custom: Specific Virtual
   Network → VMnet0**.
6. In **Edit → Virtual Network Editor**, bridge VMnet0 specifically to the
   laptop's physical Ethernet adapter connected to the desktop rather than
   relying on automatic bridging.
7. Ensure the adapter is connected and set to connect at power-on.
8. Start Sophos, open WebAdmin, and go to **Network → Interfaces** to locate
   and configure Port3.

VMware's documented procedure is available in **Using the Virtual Network
Editor**.

### Port3 configuration

| Setting | Value |
|---|---|
| Zone | LAN |
| IPv4 assignment | Static |
| IP address | `10.10.10.3` |
| Subnet mask | `255.255.255.0` |
| Network | `10.10.10.0/24` |

The directly connected desktop uses:

| Setting | Value |
|---|---|
| IPv4 address | `10.10.10.2` |
| Subnet mask | `255.255.255.0` |
| Default gateway | `10.10.10.3` |

The laptop's Ethernet interface is `10.10.10.1/24`; it is used for the direct
physical connection to the desktop.

**How this fits together:** Bridged networking attaches the VM to the selected
physical network. It does not create a Sophos firewall policy or provide NAT;
those are configured within Sophos.

## 3. Sophos firewall rules

### Allow rule: `Lab Desktop to Internet`

| Field | Configuration |
|---|---|
| Rule name | `Lab Desktop to Internet` |
| Position | First, above the deny rule and `#Default_Network_Policy` |
| Source zone | LAN |
| Source network/device | `Lab-Desktop` |
| Source IP | `10.10.10.2` |
| Destination zone | WAN |
| Destination network | Any |
| Services | DNS, HTTPS (narrowed from Any) |
| Schedule | All the time |
| Action | Accept |
| Log firewall traffic | Enabled |

The rule is intended to let the lab desktop (`10.10.10.2`) initiate
connections from LAN toward WAN. The source matches the desktop, the
destination zone limits the rule to WAN-bound traffic, the service list limits
the protocols/ports, and Accept permits matching traffic. Logging records
matching traffic subject to Sophos logging behavior.

The rule was initially broad for testing and later narrowed to DNS and HTTPS.
Narrowing an allow rule only causes traffic that no longer matches it to
continue down the rule list; it does not block that traffic by itself.

### Deny rule: `Denied_Lab-Desktop`

A second rule was placed directly below the allow rule and above
`#Default_Network_Policy`. It drops desktop traffic that did not match the
allow rule.

| Field | Configuration |
|---|---|
| Rule name | `Denied_Lab-Desktop` (firewall rule ID 4) |
| Position | Second, below the allow rule and above `#Default_Network_Policy` |
| Source | `Lab-Desktop` (`10.10.10.2`) |
| Destination | Any |
| Services | Any |
| Action | Deny (Drop or Reject; record which is set) |
| Log firewall traffic | Enabled |

**Why the explicit deny was needed:** Port 80 no longer matched the first rule
after it was narrowed to DNS and HTTPS. It continued to
`#Default_Network_Policy` (rule ID 2), which allows Any service to Any
destination, and was accepted. Adding the desktop deny above that default
policy blocked the unmatched HTTP traffic. Tests to
`http://example.com` and `http://1.1.1.1` then failed, with log entries naming
`Denied_Lab-Desktop`.

## 4. Sophos NAT rule

A linked NAT rule was created from the firewall rule using **Create linked NAT
rule**.

| Field | Configuration |
|---|---|
| Linked firewall rule | `Lab Desktop to Internet` |
| Source translation | MASQ / masquerading |
| Service translation | Original |
| Destination translation | Original |
| Inbound interface | Any interface |
| Outbound interface | Any interface |

The desktop's source address (`10.10.10.2`) is private. For typical outbound
internet access, Sophos masquerades that source address to an address
appropriate for its outbound interface.

```text
10.10.10.2 → Sophos Port3 → firewall policy → source NAT → Port2 WAN → internet
```

The reply returns through the translated connection, allowing Sophos to
associate it with the original session.

Firewall policy and NAT are separate:

- The firewall rule decides whether traffic is permitted.
- The NAT rule translates addresses when its conditions match.
- A NAT rule does not authorize traffic by itself.

### NAT rule order

The NAT rule list is separate from the firewall rule list. Both are evaluated
top-down and the first matching rule wins. Initially, desktop traffic matched a
default NAT rule (`#NAT_Default_Network_Policy`), so the custom rule appeared
Unused. Moving the custom rule to the top caused allowed log entries for
`10.10.10.2` to appear.

| Position | NAT rule | Notes |
|---|---|---|
| 1 | `Lab Desktop to Internet` (#4) | Source `Lab-Desktop`, MASQ, any interface. Used 2306 times; last used 24 Oct 19:49 |
| 2 | `#NAT_Default_Network_Policy` (#3) | Linked to firewall rule ID 2. Source any, MASQ. Used 1249 times |
| 3 | Auto-added NAT rule for MTA (#1) | Linked to firewall rule ID 1. SMTP and SMTP(S) only. Unused |
| 4 | `Default SNAT IPv4` (#2) | Source any, MASQ, outbound Port2 only. Unused |

Changing the NAT rule service to DNS and HTTPS did not block port 80. The
service filter belongs on the firewall rule; the NAT rule stays at Any service
/ Original. The custom rule showed no Firewall rule ID, unlike the default
linked NAT rules, so it acted as an independent NAT rule and translated
traffic that firewall rule 2 had allowed.

## 5. Verification performed

The following checks were reported as complete:

- The desktop can reach Sophos Port3 at `10.10.10.3`.
- The desktop's IPv4 default route points to `10.10.10.3`.
- `Test-NetConnection 1.1.1.1 -Port 443` returns
  `TcpTestSucceeded: True`.
- Internet browsing from the desktop works.
- Logging is enabled on the custom firewall rule.
- After correcting NAT rule order, green (Allowed) log entries for
  `10.10.10.2` appeared and counters increased.
- Rebooting Sophos did not change the behavior, consistent with a rule-order
  issue rather than a VM fault.
- `tracert -d 8.8.8.8` shows hop 1 as `10.10.10.3` (Sophos) and hop 2 as
  `192.168.209.2` (VMware NAT gateway). Hop 3 timing out is normal in this lab.
- Before the deny rule, port 80 tests to `1.1.1.1` and `1.0.0.1` succeeded.
  The log showed `#Default_Network_Policy` (rule ID 2) and NAT rule 4.
- After the deny rule, `curl -I -m 5 http://example.com` and
  `curl -I -m 5 http://1.1.1.1` failed to connect, and Log Viewer showed
  denied entries for `Denied_Lab-Desktop`.
- A denied UDP 443 (QUIC / HTTP/3) entry appears. The allowed HTTPS service is
  TCP 443 only, so browsers can fall back to TCP 443.

Sophos also logged denied Invalid Traffic entries with reason
`Could not associate packet to any connection`. These can be stray packets,
such as late FIN/ACK packets or packets for sessions cleared by a rule change.
They are normal noise unless they occur constantly while nothing is changing.

## 6. Troubleshooting log

The logging discrepancy from the first version of the notes was resolved.

| Symptom | Cause | Fix |
|---|---|---|
| Custom firewall and NAT rules showed Unused; logs only showed denied Invalid Traffic | NAT rule was below other NAT rules that matched first | Moved custom NAT rule to the top; allowed log entries and counters appeared |
| Setting NAT rule service to DNS and HTTPS did not block HTTP | NAT translates addresses; it does not authorize traffic | Restrict services on the firewall rule. Keep NAT at Any / Original |
| Port 80 to `1.1.1.1` and `1.0.0.1` still worked after narrowing the allow rule | `#Default_Network_Policy` (rule ID 2, Any service, Any destination) matched | Added `Denied_Lab-Desktop` above the default policy |
| Some port 80 tests failed while HTTP was still allowed | `neverssl.com` and some random IPs did not answer from this network | Use known-good targets such as `example.com` and `1.1.1.1` with `curl -I -m 5` |
| Invalid Traffic: Could not associate packet to any connection | Stray packets, such as late FIN/ACK or packets for sessions cleared by a rule change | No fix needed unless they appear constantly while nothing changes |
| Denied UDP 443 in logs | QUIC (HTTP/3) is not covered by the TCP-based HTTPS service | Left denied; browsers fall back to TCP 443. Revisit during web filtering |

### Diagnostic method

1. Run a known-good test, such as `curl -I -m 5 http://example.com` or
   `Test-NetConnection <ip> -Port <port>`.
2. Run `tracert -d <ip>` and confirm hop 1 is `10.10.10.3`.
3. In Log Viewer, filter on source `10.10.10.2` and inspect the firewall rule
   ID and NAT rule on the matching entry.
4. If there is no log entry, inspect Live connections while a slow request
   runs; the rule allowing traffic may have logging disabled.
5. Compare the rule ID with the order of both the firewall rule list and the
   NAT rule list.

`Test-NetConnection` showed a blank InterfaceAlias and SourceAddress on this
desktop for every test; this was not treated as a symptom.

**Status:** VMware topology configured; firewall and NAT rules created and
ordered; desktop internet limited to DNS and HTTPS by an allow rule plus an
explicit deny; logging working. Phase 0 snapshot and configuration backup are
complete. The DMZ is built, its web server is reachable from the LAN, and
DMZ-to-LAN isolation was verified for TCP 445. See sections 7–10.

## 7. DMZ build: Debian and Nginx

A DMZ was added so a Debian web server could be reached from the LAN without
being trusted as part of the LAN. Sophos Port4 is the DMZ gateway.

### DMZ network and interface

| Setting | Value |
|---|---|
| VMware network | VMnet2 (host-only) |
| Subnet | `10.0.0.0/24` |
| VMware DHCP | Disabled |
| Host virtual adapter | Disabled |
| Connected devices | Sophos Port4 and Debian VM |
| Sophos Port4 | DMZ zone, `10.0.0.3/24` |

VMnet0 was not used for the DMZ because it is bridged to the laptop Ethernet
adapter where the LAN lives. VMnet2 has no host adapter and no DHCP, so Sophos
is the only way in or out.

### Debian server

| Setting | Value |
|---|---|
| Operating system | Debian 12 (Bookworm), headless install |
| Network interface | `ens33` |
| IP address | `10.0.0.4/24` (static) |
| Default gateway | `10.0.0.3` (Sophos Port4) |
| Web server | Nginx |
| Services tested | SSH, HTTP, HTTPS |

The routing table has two relevant entries:

- `default via 10.0.0.3 dev ens33 onlink`: traffic for other networks goes to
  Sophos Port4.
- `10.0.0.0/24 dev ens33 proto kernel scope link src 10.0.0.4`: DMZ hosts are
  reached directly without using the gateway.

### Package repositories

The Debian package sources initially pointed only at the install media, so
`apt` had no internet repositories. Debian 12 (Bookworm) repositories were
added to `/etc/apt/sources.list` (Bookworm, Bookworm updates, and
Bookworm security from `deb.debian.org` and `security.debian.org`), then
`sudo apt update` was run.

Interface configuration, routing, DNS, firewall access, and repository
configuration are separate things to check; one can fail while the others
work.

### Nginx and HTTPS

- **Local test:** `curl http://127.0.0.1` returned the default Debian Nginx
  page. This proves the service works on the server, not that it is reachable
  from another network.
- **Self-signed certificate:** `/etc/nginx/ssl/server.crt` and
  `/etc/nginx/ssl/server.key` were used, with the key set to `chmod 600`. An
  HTTPS server block listening on `443 ssl` was added to
  `/etc/nginx/sites-available/default`.
- **403 error:** The first HTTPS request returned HTTP 403 because the
  `index` directive did not list `index.nginx-debian.html`. The fix was to add
  it to the `index` line, then run `sudo nginx -t` and
  `sudo systemctl reload nginx`.
- **Reachability:** From the desktop,
  `Test-NetConnection 10.0.0.4 -Port 443` returned
  `TcpTestSucceeded: True`. This proves the TCP path, not the page response.
  Confirm the page with `curl.exe -k -I https://10.0.0.4` and look for a 200
  response. The `-k` option is needed because the certificate is self-signed.

## 8. DMZ firewall policy

The following is the intended DMZ policy, not an export of the configured
rules. Check names, order, and objects in WebAdmin.

| Source | Destination | Services | Action | Purpose |
|---|---|---|---|---|
| Desktop `10.10.10.2` | Debian `10.0.0.4` | SSH, HTTP, HTTPS | Allow | Administration and web access |
| Debian `10.0.0.4` | WAN | DNS, HTTP, HTTPS (confirm not Any) | Allow + MASQ | Updates and DNS |
| DMZ | LAN | Any | Deny (default) | Protect the internal network |
| WAN | DMZ | Any | Deny (default) | No unsolicited inbound traffic |
| DMZ | Sophos Port4 | Ping | Allow (Device Access) | Gateway diagnostics |

### Source of the outbound rule

The DMZ-to-WAN rule first used the wrong source. Debian packets enter Sophos
through Port4 but retain Debian's own address as their source.

| Address | Represents |
|---|---|
| `10.0.0.3` | Sophos Port4 (the firewall itself) |
| `10.0.0.4` | Debian DMZ server (the correct source) |

With MASQ, source `10.0.0.4` is translated to the outgoing interface address,
so the external side needs no route back to `10.0.0.0/24`. DNS was verified
with `getent hosts deb.debian.org`.

### Firewall rule, NAT rule, or Device Access?

| Traffic | Control that applies |
|---|---|
| Debian pings Sophos Port4 | Device Access: Ping |
| Debian connects to a host on the LAN | Firewall policy |
| Debian reaches the internet | Firewall policy, routing, and usually NAT |
| Desktop connects to Nginx on Debian | Firewall policy |

Debian initially could not ping `10.0.0.3` because Ping was not enabled for
the DMZ zone under Device Access. Enabling it fixed that without opening
anything toward the LAN: Device Access covers services on the firewall
itself. Debian could also ping Sophos's own LAN-side addresses `10.10.10.3`
and `172.16.16.16`, apparently due to the same DMZ setting. Keep DMZ Device
Access to the minimum needed.

## 9. DMZ isolation testing

The goal was to confirm that the DMZ server cannot start connections into the
LAN. The test used Debian (`10.0.0.4`) to the Windows desktop (`10.10.10.2`)
on TCP 445, the SMB file-sharing port.

### Ping tests from Debian

| Destination | Result | Interpretation |
|---|---|---|
| `10.10.10.1` | No replies | Laptop Ethernet adapter; not conclusive |
| `10.10.10.2` | No replies | Desktop did not answer ping; not proof of a firewall block |
| `10.10.10.3` | Success | Sophos Port3 answered |
| `172.16.16.1` | No replies | Windows VMnet1 adapter did not answer |
| `172.16.16.16` | Success | Sophos Port1 answered |

A failed ping can result from a host firewall, disabled ICMP, or routing, so
it cannot prove isolation. A TCP test was needed.

### TCP 445 test and packet captures

- `nc -vz -w 3 10.10.10.2 445` timed out. A timeout does not identify which
  device dropped the packet.
- On Debian,
  `sudo tcpdump -ni ens33 'host 10.10.10.2 and tcp port 445'` showed repeated
  SYN packets (Flags `[S]`) and no reply. This shows Debian sent the attempts,
  not where they were dropped.
- On Sophos, packet capture showed incoming interface Port4, outgoing
  interface Port3, source `10.0.0.4`, destination `10.10.10.2`, TCP port 445,
  Status: Violation, Reason: Firewall.

The Sophos capture is the strongest evidence in this lab that the firewall
blocked the connection.

### What the tests proved

| Test | Result | Conclusion |
|---|---|---|
| Debian pings Sophos Port4 | Success after Device Access change | DMZ host reaches its gateway |
| Debian to internet | Success | Outbound path works |
| Desktop to Debian HTTP | Works | LAN-to-DMZ web access works |
| Desktop to Debian HTTPS (TCP 443) | Reachable | HTTPS path is open |
| Debian to desktop TCP 445 | Timed out; Sophos shows Port4-to-Port3 Violation | Firewall blocked this connection |

This proves the behavior of one connection (`10.0.0.4` to `10.10.10.2` on TCP
445), not every port, host, or protocol.

### Why no explicit DMZ-to-LAN rule was needed

No rule allows DMZ-to-LAN traffic, so Sophos drops it by default. Allow only
what is needed instead of allowing everything and removing items later.
Sophos is stateful: replies to a connection the desktop starts belong to that
session and do not require a rule allowing Debian to start its own connections
to the LAN. An explicit deny rule is optional and can help with documentation
and logging, but review the existing policy before adding one.

## 10. DMZ troubleshooting log and lessons

| Symptom | Cause | Fix |
|---|---|---|
| Debian could not ping `10.0.0.3` | Ping not enabled for DMZ zone in Device Access | Enabled Ping for the DMZ zone |
| `apt` could not reach any repository | `sources.list` only listed install media | Added Debian 12 Bookworm repositories and ran `apt update` |
| Outbound DMZ rule did not match Debian traffic | Source was Sophos Port4 (`10.0.0.3`), not Debian | Set source to Debian (`10.0.0.4`) |
| SSH login failed | Port 22 was reachable; authentication is separate, and root password login may be disabled | Use an ordinary user with sudo |
| HTTPS returned 403 | `index` directive omitted `index.nginx-debian.html` | Added it, tested with `nginx -t`, and reloaded Nginx |
| Timeout to LAN on TCP 445 did not show who blocked it | A timeout has many possible causes | Used tcpdump on Debian and Sophos packet capture |

Lessons from the tests:

- Routing decides where a packet goes. Firewall policy decides whether it may
  go.
- Device Access protects services on the firewall itself; it does not control
  traffic passing through.
- NAT and firewall rules are separate. Outbound access from a private subnet
  usually needs both.
- A DMZ is isolated by the firewall policy between networks, not merely
  because it has a separate subnet.
- Test the specific service and confirm with a packet capture or log rather
  than relying only on a timeout or failed ping.
- A TCP connect test does not prove the application works. Check the HTTP
  status and authentication separately.

## 11. Final rule order

Existing firewall rules for the lab desktop are evaluated top-down. Other
default rules, such as the auto-added MTA rule, were not changed. Task 2.4 is
complete, as confirmed by the lab author. Its documented order places
`LAN_Admins_to_DMZ_8080` and the existing internet and LAN-to-DMZ allows above
`Denied_Lab-Desktop`.

| Position | Rule | Action | Notes |
|---|---|---|---|
| 1 | `Lab Desktop to Internet` | Accept | `Lab-Desktop` to WAN. DNS and HTTPS. Logging on |
| 2 | `Denied_Lab-Desktop` (ID 4) | Deny | `Lab-Desktop` to any; any service; logging on |
| Below | `#Default_Network_Policy` (ID 2) | Accept | Shipped default; any service, any destination |

NAT rules are listed in section 4. The custom MASQ rule is first.

### DMZ rules

| Rule | Source to destination | Services | Action |
|---|---|---|---|
| LAN to DMZ allow | `Lab-Desktop` to `10.0.0.4` | SSH, HTTP, HTTPS | Accept |
| DMZ to WAN allow | `10.0.0.4` to WAN | DNS, HTTP, HTTPS (confirm) | Accept + MASQ |

Check the order in WebAdmin. `Denied_Lab-Desktop` denies the desktop to any
destination, so the LAN-to-DMZ allow must sit above it. There is no DMZ-to-LAN
allow rule; that traffic is dropped by default.

## 12. Task 2.4 — Custom Service + IP Host Group

**Goal:** Make access to TCP 8080 depend on membership in an IP host group,
rather than allowing the entire LAN. This task and its tests are complete, as
confirmed by the lab author.

### Part 1: Configure Debian to listen on TCP 8080

Use Python's temporary HTTP server instead of changing Nginx. On Debian
(`10.0.0.4`), run:

```bash
python3 -m http.server 8080 --bind 0.0.0.0
```

Keep the terminal open while testing. In another Debian terminal, run
`curl http://localhost:8080`; an HTTP directory listing is expected. Check
the listening socket with:

```bash
sudo ss -tlnp | grep 8080
```

`127.0.0.1` accepts local connections only. `0.0.0.0` listens on all IPv4
interfaces, allowing Windows to reach the server through Sophos.

> **Lab safety:** Run the temporary server from a directory containing only
> non-sensitive test files, because the directory listing exposes that
> directory's contents. Stop it with Ctrl+C after testing. Do not publish it
> to WAN.

### Part 2: Create a custom service in Sophos

In Sophos Firewall, open **Hosts and services → Services** (labels may vary
slightly by version) and create:

| Setting | Value |
|---|---|
| Name | `TCP_8080_Custom` |
| Protocol | TCP |
| Source port | `1:65535` (default/any) |
| Destination port | `8080` |

This reusable definition matches TCP traffic with destination port 8080. It
does not open the port automatically; the firewall rule determines whether
traffic is permitted.

### Part 3: Create IP host objects

In **Hosts and services → IP host**, create:

| Setting | First host | Second host |
|---|---|---|
| Name | `LAN_Desktop` | `LAN_Second_Device` |
| IP version | IPv4 | IPv4 |
| Type | IP | IP |
| IP address | `10.10.10.2` | Actual address of a second LAN test device |

The second-device prerequisite is complete, as confirmed by the lab author.
Use that device's actual address. `10.10.10.5` is illustrative only; do not
assign an address already used by another device. These objects match source
IP addresses, not usernames or Windows logins.

### Part 4: Create the IP host group

In **Hosts and services → IP host group**, create:

| Setting | Value |
|---|---|
| Name | `LAN_Admins` |
| Members | `LAN_Desktop`; `LAN_Second_Device` |

The group contains `LAN_Desktop` (`10.10.10.2`) and
`LAN_Second_Device` (the actual second-device address; illustrative example:
`10.10.10.5`).

**Why use a group?** If multiple rules refer to the group, removing one device
from the group updates membership for all those rules. This centralizes
IP-based access management; it is not user authentication.

### Part 5: Create the LAN-to-DMZ firewall rule

In **Rules and policies → Firewall rules**, create, save, and activate:

| Setting | Value |
|---|---|
| Rule name | `LAN_Admins_to_DMZ_8080` |
| Source zone | LAN |
| Source networks/devices | `LAN_Admins` |
| Destination zone | DMZ |
| Destination networks | Debian (`10.0.0.4`) |
| Services | `TCP_8080_Custom` |
| Action | Accept |
| Log firewall traffic | Enabled |

Leave unrelated settings at appropriate lab defaults. This is routed LAN-to-DMZ
access; do not add WAN publishing as part of this exercise.

#### Rule order

Place `LAN_Admins_to_DMZ_8080` above `Denied_Lab-Desktop`. Keep the existing
internet and LAN-to-DMZ allow rules above that deny too.

| Relative order | Rule | Purpose |
|---|---|---|
| Above desktop deny | `LAN_Admins_to_DMZ_8080` | Allow group members to Debian TCP 8080 |
| Above desktop deny | Existing LAN-to-DMZ allow | Preserve SSH, HTTP, and HTTPS access |
| Above desktop deny | `Lab Desktop to Internet` | Preserve DNS and HTTPS to WAN |
| Below applicable allows | `Denied_Lab-Desktop` | Deny remaining desktop traffic |
| Below desktop deny | `#Default_Network_Policy` and other applicable rules | Avoid a bypass of the membership test |

The first matching applicable rule handles traffic. Check for broader rules
that might permit TCP 8080 independently. `Denied_Lab-Desktop` protects the
desktop test specifically; it does not deny every LAN device. For
membership-only access across the LAN, confirm non-members have no alternate
allow path and are denied by applicable policy.

### Part 6: Test access from Windows

With `LAN_Desktop` in `LAN_Admins`, run on Windows (`10.10.10.2`):

```powershell
curl.exe -I http://10.0.0.4:8080
```

Expected response (version text may differ):

```text
HTTP/1.0 200 OK
Server: SimpleHTTP/0.6 Python/3.x
```

To retrieve the directory listing, run `curl.exe http://10.0.0.4:8080`.
The intended path is Windows `10.10.10.2` → Sophos Port3 → source group and
`TCP_8080_Custom` checks → Sophos Port4 → Debian `10.0.0.4:8080`.

Confirm that the allowed attempt matches `LAN_Admins_to_DMZ_8080`: source
`LAN_Desktop` through `LAN_Admins`, destination Debian, service
`TCP_8080_Custom`. Record the actual response and matching rule.

### Part 7: Remove the desktop from the group

In **Hosts and services → IP host group → LAN_Admins**, remove
`LAN_Desktop`, keep `LAN_Second_Device`, and save. Do not delete the individual
`LAN_Desktop` host object. Retest using a new connection:

```powershell
curl.exe -I --connect-timeout 5 http://10.0.0.4:8080
```

Expected: timeout or another connection failure, depending on the firewall
response. The source no longer matches `LAN_Admins`, so the group allow should
not match; the applicable lower-priority deny policy should handle the
attempt, provided no other rule permits it. Use fresh requests because
existing stateful sessions can obscure a membership change.

### Check Sophos logs

In **Log Viewer → Firewall**, find the connection attempt and check:

| Field | Expected |
|---|---|
| Source IP | `10.10.10.2` |
| Destination IP | `10.0.0.4` |
| Destination port | `8080` |
| Incoming interface | Port3 |
| Outgoing interface | Port4 (where displayed) |
| Action | Denied / Violation |
| Rule | Applicable deny rule, expected `Denied_Lab-Desktop` if it matches |

Displayed names, interface fields, and log wording depend on version and
configuration. If an unexpected allow rule appears, investigate rule order
and broader policies. A timeout alone does not prove the firewall blocked the
request; identify the denying policy in the log or packet capture.

### Part 8: Restore desktop access

Add `LAN_Desktop` back to `LAN_Admins`, save, and run:

```powershell
curl.exe -I http://10.0.0.4:8080
```

Expected: HTTP 200 again. The included → removed → restored membership test
sequence is complete, as confirmed by the lab author.

### Part 9: What this task demonstrates

| Concept | What it demonstrates |
|---|---|
| Custom service | Define a reusable protocol and port combination |
| IP host object | Represent a specific device or IP address |
| IP host group | Combine multiple IP host objects into one reusable object |
| Firewall rule order | Determine which applicable policy handles traffic |

A firewall rule can reference a group instead of listing every individual IP
address. Changes to the group affect every rule that uses it. This is
IP-based access control, not user authentication.

### Task 2.4 completion checklist

All checklist items are complete, as confirmed by the lab author.

| Requirement | Expected result | Status |
|---|---|---|
| Debian listens on TCP 8080 | Listening socket on `0.0.0.0:8080` | Complete |
| Local Debian test works | HTTP response / directory listing | Complete |
| Custom Sophos service created | `TCP_8080_Custom` | Complete |
| Two IP host objects created | Desktop + real second LAN device | Complete |
| IP host group created | `LAN_Admins` | Complete |
| Firewall rule created | LAN → DMZ, TCP 8080 | Complete |
| Allow rule placed above deny rule | Correct order; no alternate allow bypass | Complete |
| Desktop included in group | Access succeeds | Complete |
| Desktop removed from group | Access fails on a new connection | Complete |
| Sophos log checked | Deny policy identified | Complete |
| Desktop added back | Access succeeds again | Complete |

## 13. Sophos Firewall — SSL VPN Remote Access Lab

### 13.1 Objective and scope

This extends the VMware, LAN, and DMZ lab with Sophos SSL VPN remote access.
A client on the external-side VMware NAT network connects through the VPN to
selected internal resources rather than accessing them directly.

The work covered:

- Enabling WAN access for SSL VPN and the VPN Portal.
- Creating a dedicated VPN user and user group.
- Configuring an SSL VPN remote access policy with restricted destinations.
- Downloading and importing client profiles for Windows and headless Debian.
- Testing remote connectivity and checking tunnel and firewall activity.

SSL VPN configuration and connectivity checks are complete, as confirmed by
the lab author. Configuration instructions and expected results are retained
for reference.

### 13.2 Lab environment and topology

- VMware Workstation with the existing Sophos Firewall VM.
- Debian 12 DMZ web server at `10.0.0.4` and Windows LAN host at `10.10.10.2`.
- Separate headless Debian VM as remote VPN client at `192.168.209.129`.
- Windows testing with Sophos Connect; record the Windows client's external
  address and attachment when testing remote access.

External-side path:

```text
Remote Debian client 192.168.209.129
  → VMnet8 / VMware NAT 192.168.209.0/24
  → Sophos Port2 WAN 192.168.209.128
  → SSL VPN address pool 10.250.250.0/24
  → permitted LAN and DMZ hosts
```

Internal destinations: Sophos Port3 `10.10.10.3` → Windows LAN host
`10.10.10.2`; Sophos Port4 `10.0.0.3` → Debian DMZ server `10.0.0.4`.
Port1 remains on VMnet1. The LAN uses VMnet0 bridged to laptop Ethernet, and
the DMZ uses isolated VMnet2.

The remote client shares VMware NAT with Sophos WAN to simulate an
external-side client. This is a lab network, not evidence of access from the
public internet. The VPN is intended to provide access only to permitted
internal resources.

### 13.3 Network configuration

| Device / interface | IP address | Subnet mask | Gateway |
|---|---|---|---|
| Sophos Port1 | `172.16.16.16` | `255.255.255.0` | — |
| Sophos Port2 (WAN) | `192.168.209.128` | `255.255.255.0` | `192.168.209.2` |
| Sophos Port3 (LAN) | `10.10.10.3` | `255.255.255.0` | — |
| Sophos Port4 (DMZ) | `10.0.0.3` | `255.255.255.0` | — |
| Windows LAN host | `10.10.10.2` | `255.255.255.0` | `10.10.10.3` |
| Debian DMZ server | `10.0.0.4` | `255.255.255.0` | `10.0.0.3` |
| Remote VPN client (Debian) | `192.168.209.129` | `255.255.255.0` | `192.168.209.2` |

### 13.4 Sophos Firewall configuration

#### WAN interface and Device Access

Configure Port2 as WAN with `192.168.209.128/24` and gateway
`192.168.209.2`. Place the remote Debian client on VMnet8, not on the LAN or
DMZ.

In **Administration → Device Access**, the notes record these WAN settings:

- Enable SSL VPN access where required.
- Enable VPN Portal access where required.
- Keep Admin Console and SSH access disabled on WAN.
- Keep WAN Ping disabled.

WAN-directed ICMP was rejected by the Sophos Local ACL while WAN Ping was
disabled. Device Access controls services on Sophos itself; firewall policy
controls traffic passing through Sophos. A failed WAN ping alone does not
establish whether the VPN service is reachable.

#### Dedicated VPN user and group

Create a dedicated local VPN account and assign it to a dedicated VPN group.
`VPN-Users` is the example group name in the notes; record the actual group
and non-secret account identifier used in WebAdmin. Use that group in the SSL
VPN remote access policy instead of granting VPN access to all users. Do not
record the password.

#### SSL VPN address pool

Configure the SSL VPN address pool as `10.250.250.0/24`. Avoid overlapping
these networks:

- `10.10.10.0/24`
- `10.0.0.0/24`
- `172.16.16.0/24`
- `192.168.209.0/24`
- `192.168.32.0/24` (listed in the VPN notes but not otherwise mapped in the
  existing topology)

#### SSL VPN remote access policy

Associate the dedicated VPN group with the remote access policy. Restrict
permitted destinations to the existing Windows LAN host and Debian DMZ server
rather than permitting the entire LAN. Record the actual policy name, host
objects, and firewall rules used; the supplied notes do not provide their
names or service restrictions.

| Host | IP address | Purpose |
|---|---|---|
| Windows LAN host | `10.10.10.2` | Internal workstation |
| Debian DMZ server | `10.0.0.4` | DMZ web server |

Keep this VPN policy separate from the LAN-originated desktop policy in
sections 3 and 11 and the `LAN_Admins` TCP 8080 policy in section 12. The notes
do not identify changes to those earlier rules.

### 13.5 VPN Portal and client profile

The following ports are recorded for this lab, not as universal defaults:

| Service | Port | Purpose |
|---|---:|---|
| Admin Console | 4444 | Firewall administration; not exposed on WAN |
| User Portal | 4443 | User portal |
| VPN Portal | 443 | Obtain the VPN client configuration |

From the remote network, access the VPN Portal at
`https://192.168.209.128:443`. Authenticate with the dedicated VPN account and
download the client configuration. The notes identify a download compatible
with Sophos Connect and OpenVPN Connect v2, plus a separate download for
OpenVPN Connect v3. Record which profile was actually downloaded and tested.

#### Windows client

The notes identify Sophos Connect 2.5.1 MR, described there as supporting
IPsec and SSL VPN. Import the downloaded SSL VPN configuration and
authenticate with the dedicated account. Record the connection result and
assigned VPN address. Keep this remote-client test separate from testing
directly on the existing Windows LAN host.

### 13.6 Headless Debian VPN client

Use the Linux OpenVPN client on the separate headless Debian remote VM, not
the Debian DMZ web server. The terminal client avoids requiring a graphical
desktop.

Install OpenVPN:

```bash
sudo apt update
sudo apt install openvpn -y
```

Transfer the downloaded `.ovpn` profile using a USB drive. Identify the
actual USB partition before mounting it:

```bash
lsblk
```

The notes use `/dev/sdb1` as the USB partition and `client.ovpn` as the
profile filename. Substitute the actual device and filename if different.

```bash
sudo mkdir -p /mnt/usb
sudo mount /dev/sdb1 /mnt/usb
cp /mnt/usb/client.ovpn ~/
sudo umount /mnt/usb
```

Start the VPN connection and authenticate when prompted:

```bash
sudo openvpn --config ~/client.ovpn
```

The VPN connection test is complete, as confirmed by the lab author.

> **Sensitive configuration:** Treat VPN credentials and downloaded client
> profiles as sensitive. Do not include passwords or profile contents in this
> lab record. Verify the USB partition rather than assuming `/dev/sdb1`.

### 13.7 VPN connectivity testing and evidence

All VPN checks below are complete, as confirmed by the lab author. Expected
results are retained as reference criteria; no additional test outputs or
addresses have been invented.

| Test | Expected result | Status |
|---|---|---|
| Access VPN Portal from remote network | Successful | Complete |
| Authenticate VPN user | Successful | Complete |
| Import VPN configuration | Successful | Complete |
| Establish SSL VPN connection | Successful | Complete |
| Obtain VPN address | Address from `10.250.250.0/24` | Complete |
| Access permitted LAN host, `10.10.10.2` | Allowed | Complete |
| Access permitted DMZ server, `10.0.0.4` | Allowed | Complete |
| Access unauthorized internal hosts | Blocked | Complete |
| Disconnect VPN | Internal VPN access terminated | Complete |

#### Client network verification

After establishing the tunnel, inspect the client interfaces and routes.

On Windows:

```powershell
ipconfig
```

On Debian:

```bash
ip -br addr
ip route
```

Record the assigned VPN address and relevant routes. The expected VPN address
belongs to `10.250.250.0/24`. Do not confuse the external-side Debian address
`192.168.209.129` with its VPN address.

#### Sophos firewall verification

The notes state that Sophos Log Viewer was used to inspect activity. For each
permitted and denied test, record the source IP, destination IP, destination
port, incoming and outgoing interfaces, firewall action, connection status,
and matching policy or rule where displayed.

Keep expected results separate from observed evidence. A connection failure
alone does not identify which device or policy blocked it, and one successful
or blocked test does not prove every host, port, or protocol behaves the same
way.

### 13.8 Security considerations

- Use a dedicated VPN user group instead of granting access to all users.
- Limit VPN access to the specifically required Windows LAN host and Debian
  DMZ server.
- Use the separate `10.250.250.0/24` VPN pool and check it against the listed
  networks.
- Keep Admin Console access disabled on WAN.
- Keep WAN Ping disabled.
- Do not directly expose internal LAN and DMZ resources to the external
  network as part of this VPN lab.
- Use firewall logging to monitor connection attempts.
- Treat VPN credentials and client configuration files as sensitive.

### 13.9 Skills demonstrated

- Sophos Firewall administration, SSL VPN configuration, and troubleshooting.
- WAN, LAN, and DMZ segmentation; VMware virtual networking and NAT.
- Firewall rules versus Local ACL and Device Access controls.
- VPN user/group management and least-privilege destination access.
- Windows Sophos Connect and headless Linux OpenVPN configuration.
- Connectivity testing, firewall log analysis, and remote access security.

### 13.10 Conclusion and completion status

The SSL VPN remote access environment was successfully configured and tested
using a dedicated VPN account and restricted destination policy, as confirmed
by the lab author. It provides VPN access to selected internal resources.

Windows with Sophos Connect and headless Debian with OpenVPN were used for
client tests. The lab author confirms the VPN tasks and connectivity checks
are complete. This extends the routed LAN and isolated DMZ lab without
replacing the earlier desktop firewall policy. Task 2.4 and the DNAT and
hairpin NAT tasks are also reported complete.

## 14. Completion summary and follow-up

- Task 2.4 is complete: the included → removed → restored group-membership
  sequence was tested, as confirmed by the lab author.
- Tasks 3.2 and 3.3 are complete: DMZ web-server DNAT and LAN hairpin NAT
  testing, as confirmed by the lab author.
- Decide whether to keep QUIC (UDP 443) blocked; it matters for web filtering
  and TLS inspection.
- The SSL VPN lab in section 13 is complete, including client connectivity,
  permitted and denied access, firewall activity, and disconnect checks, as
  confirmed by the lab author.

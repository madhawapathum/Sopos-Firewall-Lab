# Sophos Firewall Lab

A hands-on Sophos Firewall lab built with VMware Workstation, covering network
segmentation, firewall policy, NAT, a DMZ web server, IP-based access control,
and SSL VPN remote access.

The lab notes are based on the working setup described in
[LAB_GUIDE.md](./LAB_GUIDE.md). They preserve the recorded configuration and
test results, with additional explanations of how traffic moves through the
lab.

## What this lab demonstrates

- Connecting virtual machines to isolated, NAT, and bridged VMware networks.
- Routing traffic between a LAN, WAN, and DMZ through Sophos Firewall.
- Controlling traffic with ordered firewall rules and service restrictions.
- Translating private source addresses with masquerading (source NAT).
- Hosting a Debian/Nginx web server in a DMZ while limiting access to the LAN.
- Using an IP host group to manage access to a custom TCP service.
- Connecting remote clients to selected internal resources with SSL VPN.
- Verifying behavior with connection tests, logs, and packet captures.

## Topology at a glance

| Segment | VMware network | Sophos interface | Addressing / role |
|---|---|---|---|
| Existing private network | VMnet1, host-only | Port1 | Sophos: `172.16.16.16/24` |
| WAN / internet side | VMnet8, NAT | Port2 | Sophos: `192.168.209.128/24`; gateway: `192.168.209.2` |
| LAN | VMnet0, bridged to laptop Ethernet | Port3 | Sophos: `10.10.10.3/24`; desktop: `10.10.10.2` |
| DMZ | VMnet2, host-only, no DHCP or host adapter | Port4 | Sophos: `10.0.0.3/24`; Debian server: `10.0.0.4` |
| SSL VPN clients | SSL VPN pool | VPN tunnel | `10.250.250.0/24` |

The remote Debian VPN client uses VMnet8 to simulate an external-side client.
This is a VMware lab path, not a test of access from the public internet.

## How traffic is handled

For a typical outbound connection, the desktop sends traffic to its default
gateway, Sophos Port3. Sophos evaluates the firewall rules from top to bottom;
the first matching rule determines whether the traffic is permitted or denied.
For permitted internet traffic, a matching NAT rule can then replace the
private source address with the address used on the outbound interface. The
reply returns through that translated session and Sophos associates it with the
original connection.

Firewall policy and NAT have different jobs:

- **Firewall policy** decides whether traffic is allowed.
- **NAT** changes addresses or ports when its conditions match; it does not
  grant permission by itself.
- **Device Access** (also described as Local ACL behavior in the notes) controls
  access to services running on the firewall itself. It is not the policy for
  traffic passing through the firewall.
- **Routing** selects the next hop. A route alone does not permit traffic.

The DMZ is a separate network, but the subnet separation alone is not what
protects the LAN. Sophos policy and the absence of an allow path from DMZ to LAN
provide that boundary. The documented isolation test covers one specific
connection (DMZ host to LAN desktop on TCP 445); it should not be read as proof
that every host, port, or protocol has been tested.

## Getting started

1. Review the VMware network mapping and addresses in
   [LAB_GUIDE.md](./LAB_GUIDE.md).
2. Confirm the virtual adapters are attached to the intended VMnet networks
   before starting the Sophos and Debian VMs.
3. Check the firewall and NAT rule order in Sophos WebAdmin; order is part of
   the configuration and can change which policy handles a connection.
4. Run the tests from the relevant client and verify the matching Sophos log
   entry. A timeout or failed ping by itself does not identify what blocked a
   connection.

Menu names and behavior can vary between Sophos Firewall versions. Treat
configuration tables marked as intended policy or examples as design notes,
not as exports of the live configuration.

## Lab safety and privacy

- Use this configuration only in an authorized lab environment.
- Keep Sophos administration services disabled on WAN, as described in the
  guide.
- Do not publish the temporary Python HTTP server or internal services to WAN.
- The temporary HTTP server exposes the contents of its working directory;
  run it only in a directory containing non-sensitive test files and stop it
  after testing.
- Do not commit passwords, VPN profile contents, private keys, or other
  credentials to GitHub. The guide intentionally omits them.
- Verify the actual USB device name before mounting removable media. Device
  names such as `/dev/sdb1` are examples and may not match your system.

## Status

The source notes report the desktop firewall/NAT path, DMZ build and isolation
test, TCP 8080 group-membership exercise, DNAT and hairpin NAT tasks, and SSL
VPN connectivity tests as complete. See the guide for the recorded evidence,
scope, caveats, and follow-up items.

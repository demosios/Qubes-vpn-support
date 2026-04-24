# Qubes 4.3 Firewall Verification

This document explains how to verify the default Qubes firewall state and how this project layers additional rules on top.

Qubes OS 4.3 uses `nftables`. The relevant Qubes firewall logic is installed inside the relevant networking qube, not in dom0 itself.

## Official References

- Qubes firewall documentation:
  https://doc.qubes-os.org/en/latest/user/security-in-qubes/firewall.html
- Qubes OS 4.3 release notes:
  https://doc.qubes-os.org/en/latest/developer/releases/4_3/release-notes.html

These references support the statements in this document about:

- Qubes 4.3 using `nftables`
- dom0 storing policy while firewall rules are enforced in networking qubes
- use of Qubes-managed `qubes` firewall chains and user rule hooks

## Non-official References

The exact rule shapes discussed in this repository were compared against the upstream project history:

- Upstream repository:
  https://github.com/tasket/Qubes-vpn-support
- `nftables` migration branch:
  https://github.com/tasket/Qubes-vpn-support/tree/replace-iptables-with-nftables
- Patch discussion referenced during this review:
  https://github.com/tasket/Qubes-vpn-support/pull/71

These links are implementation-history references, not authoritative documentation for Qubes OS internals.

## What To Check In dom0

Verify which qube acts as the firewall and which qube acts as the VPN ProxyVM:

```bash
qvm-prefs <vpn-proxyvm> netvm
qvm-prefs sys-firewall netvm
```

Inspect the dom0-managed firewall policy for a qube:

```bash
qvm-firewall <qube>
qvm-firewall --list <qube>
```

Inspect the raw XML if you want to see what dom0 stores:

```bash
sudo cat /var/lib/qubes/appvms/<qube>/firewall.xml
```

If you have never customized `sys-firewall` or the target qube, these outputs should reflect the default Qubes policy for your installation.

Reference:

- `qvm-firewall` usage and dom0-managed firewall policy model are described in the official Qubes firewall documentation:
  https://doc.qubes-os.org/en/latest/user/security-in-qubes/firewall.html

## What To Check Inside the ProxyVM

Open a terminal in the VPN ProxyVM and inspect the Qubes-managed `nftables` table:

```bash
sudo nft list tables
sudo nft list table ip qubes
sudo nft list table ip6 qubes
```

The main chains of interest are:

```bash
sudo nft list chain ip qubes custom-forward
sudo nft list chain ip qubes custom-input
sudo nft list chain ip qubes output
sudo nft list chain ip qubes dnat-dns
sudo nft list chain ip6 qubes custom-forward
```

Reference:

- The Qubes firewall documentation describes the current `nftables`-based firewall model and where local rules may be layered into Qubes networking qubes:
  https://doc.qubes-os.org/en/latest/user/security-in-qubes/firewall.html

## What This Project Adds

After this project is installed and active, you should see rules equivalent to:

```nft
oifgroup 1 drop
iifgroup 1 drop
iifgroup 2 oifgroup 9 accept
iifgroup 9 oifgroup 2 ct state established,related accept
```

You should also see:

- IPv6 drops in the `ip6 qubes` chains
- a locked-down `ip qubes output` chain
- `dnat-dns` rules only after the VPN link is up
- Qubes virtual DNS values sourced either from `/var/run/qubes/qubes-ns` or, on current Qubes 4.3 templates, from QubesDB keys exposed as `/qubes-netvm-primary-dns` and `/qubes-netvm-secondary-dns`

## Verify IPv6 Is Disabled

Inside the ProxyVM:

```bash
cat /proc/sys/net/ipv6/conf/all/disable_ipv6
cat /proc/sys/net/ipv6/conf/default/disable_ipv6
cat /proc/sys/net/ipv6/conf/lo/disable_ipv6
```

Expected value for strict mode:

```text
1
```

## Verify the Service Is Refusing Incomplete Configurations

Before `vpn-client.conf` exists:

```bash
systemctl status qubes-vpn-handler.service
# WireGuard uses:
# systemctl status qubes-wg-handler.service
ls -l /rw/config/vpn/backend-openvpn /rw/config/vpn/backend-wireguard
```

The service should not start because the unit now requires:

```text
/rw/config/vpn/vpn-client.conf
```

That prevents useless restart loops before deployment is complete.

## Verify Post-Connect DNS Rules

After the VPN comes up:

```bash
sudo nft list chain ip qubes dnat-dns
cat /var/run/qubes/qubes-vpn-ns
```

You should see DNAT rules redirecting downstream DNS traffic to the configured VPN DNS servers.

On Qubes 4.3 systems where `/var/run/qubes/qubes-ns` is absent, this repository now falls back to reading:

```bash
qubesdb-read /qubes-netvm-primary-dns
qubesdb-read /qubes-netvm-secondary-dns
```

That is the expected modern path for determining Qubes virtual DNS addresses in this deployment.

If the VPN is down:

```bash
sudo nft list chain ip qubes dnat-dns
```

The chain should be empty or flushed.

## Verify the Tunnel Interface Group

For OpenVPN, the helper script assigns the tunnel interface to group `9`.

Check link groups:

```bash
ip -d link show
```

Look for the active VPN interface and verify that it is assigned to group `9`.

## What “Default Qubes Firewall” Means Here

The important nuance is:

- dom0 stores the policy
- Qubes applies firewall state inside networking qubes
- this project modifies the in-VM `qubes` chains created by Qubes

So the verification path is split across:

- dom0 for policy definition
- the VPN ProxyVM for the actual `nftables` state that enforces the local hardening

Reference:

- Official Qubes firewall model:
  https://doc.qubes-os.org/en/latest/user/security-in-qubes/firewall.html

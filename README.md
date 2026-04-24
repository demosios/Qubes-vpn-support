# Qubes-vpn-support

Hardened VPN ProxyVM tooling for Qubes OS using `nftables`.

This codebase turns a Qubes ProxyVM into a fail-closed VPN gateway for downstream VMs. Its primary design goal is simple:

- downstream VMs must not reach clearnet before the VPN is up
- downstream VMs must not bypass the VPN while it is up
- downstream VMs must lose connectivity again if the VPN goes down
- DNS from downstream VMs must be redirected to VPN-provided DNS only
- IPv6 must be disabled, not “supported carefully”

The current implementation targets modern Qubes networking with `nftables` and is intended for Debian 13 and current Fedora templates used with Qubes OS 4.3.

## Install Quickstart

Use this section for the short path. For the full deployment walkthrough, see [ProxyVM Template Quickstart](docs/PROXYVM_TEMPLATE_QUICKSTART.md).

In the TemplateVM:

```bash
cd Qubes-vpn-support
sudo bash ./install
```

Shut down the TemplateVM, create a VPN ProxyVM from it, enable `provides network`, and start the ProxyVM.

In the VPN ProxyVM:

```bash
sudo /usr/lib/qubes/qubes-vpn-setup --config-openvpn
# or:
# sudo /usr/lib/qubes/qubes-vpn-setup --config-wireguard
```

Successful configuration selects the backend for that ProxyVM by creating one persistent marker:

- `/rw/config/vpn/backend-openvpn`
- `/rw/config/vpn/backend-wireguard`

Add provider files to `/rw/config/vpn/`, including:

- `vpn-client.conf`
- required certs, keys, and CRLs
- optional username/password via `sudo /usr/lib/qubes/qubes-vpn-setup --userpass`

Then start the handler:

```bash
sudo systemctl restart qubes-firewall.service
sudo systemctl restart qubes-vpn-handler.service
sudo systemctl status qubes-vpn-handler.service
# WireGuard uses qubes-wg-handler.service instead.
```

Verify before attaching downstream VMs:

```bash
ip -br link
sudo nft list chain ip qubes custom-forward
sudo nft list chain ip qubes dnat-dns
cat /var/run/qubes/qubes-vpn-ns
```

## Tested Versions

Last explicitly observed working combination during this work:

- Qubes OS: 4.3.0
- Template: `debian-13-minimal` from the Qubes repository
- Firewall backend: `nftables`
- OpenVPN: 2.6.14

Version information that matters for this repository:

- Qubes OS release, because firewall internals and helper paths can change across major releases
- Template distribution and version, because package names and helper availability differ between Debian and Fedora
- VPN backend and version, because OpenVPN and WireGuard interact with hooks and routing differently
- `nftables`-based Qubes firewall model, because this repository no longer targets the legacy `iptables` path

## Fundamental Changes From The Original Project

This repository no longer behaves like the older upstream `iptables`-based implementation. The most important changes are architectural, not cosmetic.

### Firewall Backend

Original project behavior:

- legacy `iptables` / `ip6tables`
- more reliance on older helper paths and older Qubes assumptions

Current behavior:

- `nftables` only
- direct integration with the Qubes `qubes` table and custom chains
- explicit startup validation of the installed firewall state

### IPv6 Policy

Original project behavior:

- attempted IPv6 anti-leak handling while still supporting some IPv6 logic

Current behavior:

- IPv6 is intentionally disabled
- kernel IPv6 knobs are forced off
- IPv6 forward/input/output paths are dropped
- IPv6 DNS and IPv6 endpoints are rejected in strict mode

### Forwarding Policy

Original project behavior:

- downstream anti-leak relied more heavily on inherited Qubes behavior
- reverse flow handling was less explicit

Current behavior:

- downstream traffic is allowed only from interface group `2` to VPN group `9`
- return traffic from the VPN side is allowed only for `ct state established,related`
- downstream traffic to the upstream interface group is explicitly dropped

### DNS Handling

Original project behavior:

- downstream DNS redirection existed, but relied on older helper assumptions
- hostname handling before tunnel establishment was less strict

Current behavior:

- downstream DNS DNAT is installed only after VPN link-up
- VPN startup fails closed if strict DNS information is missing
- OpenVPN `remote` hostnames are pre-resolved through configured VPN DNS
- WireGuard `Endpoint` hostnames are pre-resolved through configured VPN DNS
- Qubes virtual DNS is read from `/var/run/qubes/qubes-ns` when available and from QubesDB on current Qubes 4.3 templates

### Service Lifecycle

Original project behavior:

- the service could start before a real `vpn-client.conf` existed
- first-run firewall timing was more fragile

Current behavior:

- the service requires `/rw/config/vpn/vpn-client.conf`
- the Qubes firewall service is reloaded when the local firewall hook is installed
- the startup firewall check uses semantic `nft -j` parsing instead of brittle text matching

### Configuration Workflow

Original project behavior:

- setup assumed a more immediate “install and configure everything now” flow

Current behavior:

- staged deployment is supported
- install first, configure the ProxyVM, then add config/certs/userpass later
- `--userpass` allows credentials to be added after initial setup
- dependency checking is built into the installer and is distro-aware

### Backend Support

Original project behavior:

- OpenVPN was the main path
- WireGuard support was more limited and less aligned with the current firewall model

Current behavior:

- OpenVPN uses `qubes-vpn-handler.service`
- WireGuard uses `qubes-wg-handler.service`
- backend selection is explicit in the ProxyVM via `--config-openvpn` or `--config-wireguard`
- backend persistence is stored in `/rw/config/vpn/backend-openvpn` or `/rw/config/vpn/backend-wireguard`

## Why This Is In README Instead Of Only A Changelog

A changelog is still useful for release-by-release history, but it is not enough for this transition.

The differences here affect:

- threat model
- firewall semantics
- DNS resolution behavior
- deployment workflow
- runtime assumptions about modern Qubes OS

That makes a README architecture/delta section the right place for the explanation. If release discipline becomes important later, a separate `CHANGELOG.md` can summarize versioned milestones while this section explains the design-level differences.

## Official References

The Qubes-specific assumptions in this repository are based on the following official Qubes OS documentation:

- Qubes firewall documentation:
  https://doc.qubes-os.org/en/latest/user/security-in-qubes/firewall.html
- Qubes OS 4.3 release notes:
  https://doc.qubes-os.org/en/latest/developer/releases/4_3/release-notes.html
- Debian template documentation:
  https://doc.qubes-os.org/en/latest/user/templates/debian/debian.html

## Non-official References

The implementation port and compatibility review also used upstream project materials from the original repository:

- Upstream project repository:
  https://github.com/tasket/Qubes-vpn-support
- Discussion referenced by the user about the `nftables` patch direction:
  https://github.com/tasket/Qubes-vpn-support/pull/71
- Upstream `replace-iptables-with-nftables` branch:
  https://github.com/tasket/Qubes-vpn-support/tree/replace-iptables-with-nftables
- Upstream commit introducing the `nftables` migration:
  https://github.com/tasket/Qubes-vpn-support/commit/d05c0f1
- Upstream commit refining DNS handling on the `nftables` line:
  https://github.com/tasket/Qubes-vpn-support/commit/4141e94

These non-official references were used to understand upstream intent and to port project-local behavior into this repository. They were not used as authority for Qubes OS behavior itself.

## What This Code Does

At install time, the project installs a systemd service and helper scripts into a TemplateVM or directly into a ProxyVM.

At runtime, it does the following inside the VPN ProxyVM:

- adds a restrictive `nftables` profile on top of Qubes' own `qubes` table/chains
- disables IPv6 through kernel sysctls and drops IPv6 in firewall chains
- blocks downstream forwarding to the upstream network interface group
- allows downstream forwarding only to the VPN tunnel interface group
- allows reverse VPN-to-downstream forwarding only for `ct state established,related`
- restricts upstream egress from the ProxyVM itself to the VPN process group
- redirects downstream DNS to VPN DNS only after the link is up
- refuses to continue if VPN DNS is missing in strict mode
- resolves OpenVPN `remote` hostnames and WireGuard `Endpoint` hostnames against configured VPN DNS before starting the client
- tags the active VPN interface into link group `9`, which becomes the only allowed downstream egress path
- resolves Qubes virtual DNS addresses from `/var/run/qubes/qubes-ns` when available and falls back to QubesDB (`/qubes-netvm-primary-dns`, `/qubes-netvm-secondary-dns`) on Qubes OS 4.3 environments where the legacy helper file is absent

The end result is an in-VM enforcement model built for Qubes' network topology rather than a generic Linux laptop VPN setup.

## Security Model

This project assumes:

- Qubes OS manages the base `qubes` firewall table and interface groups
- the ProxyVM uses the normal Qubes networking model and a default `sys-firewall`
- the operator wants stronger local restrictions inside the VPN ProxyVM, not less

This project does not claim mathematically perfect leakproof behavior. It is designed to fail closed, but the final security claim still depends on runtime verification on the target Qubes release, template, and VPN provider configuration.

The important design choices are:

- explicit `nftables` rules over legacy `iptables`
- explicit conntrack return handling rather than broad reverse forwarding
- IPv6 removal instead of mixed IPv4/IPv6 policy complexity
- delayed DNS redirection until VPN DNS is known
- hostname resolution pinned to configured VPN DNS where supported

## Technologies Used

- `systemd` for lifecycle management
- `nftables` for packet filtering and DNS DNAT
- Qubes OS `qubes-firewall.d` integration
- Qubes interface grouping (`iifgroup` / `oifgroup`)
- Linux conntrack (`ct state established,related`)
- OpenVPN `up`/`down` hooks
- WireGuard `wg-quick` override hooks
- `notify-send` and `xterm` for operator prompts and status messages

## File Map

- `install`
  Thin wrapper that enters `files-main` and runs the installer.
- `files-main/proxy-firewall-restrict`
  Main `nftables` hardening script applied by Qubes firewall hooks.
- `files-main/qubes-vpn-setup`
  Installer, dependency checker, lifecycle helper, and runtime validator.
- `files-main/qubes-vpn-ns`
  DNS translation logic for downstream VMs after the link comes up.
- `files-main/qubes-vpn-openvpn-script`
  OpenVPN helper that runs the DNS hook and assigns the tunnel interface to group `9`.
- `files-main/qubes-vpn-handler.service`
  Systemd service for OpenVPN.
- `files-main/qubes-wg-handler.service`
  Systemd service for WireGuard.
- `files-main/vpn/vpn-client.conf-example`
  Example OpenVPN config with strict DNS notes.

## Firewall Behavior

The firewall profile is layered on top of Qubes' own `qubes` table.

IPv4 forwarding policy:

- drop downstream traffic to upstream: `oifgroup 1 drop`
- drop upstream traffic to downstream unless stateful return rule matches: `iifgroup 1 drop`
- allow new downstream traffic only from downstream group `2` to VPN group `9`
- allow VPN-to-downstream traffic only when `ct state established,related`

IPv6 policy:

- IPv6 is disabled in kernel sysctls
- IPv6 forward/input/output paths are dropped in firewall chains
- IPv6 endpoints and DNS values are rejected in strict mode

ProxyVM egress policy:

- loopback is allowed
- upstream egress is allowed only for traffic running as group `qvpn`
- this is strongest for userspace VPNs like OpenVPN

## DNS Behavior

Strict DNS handling is part of the threat model.

For downstream VMs:

- before VPN up: no DNS DNAT rules are installed
- after VPN up: downstream DNS to Qubes virtual DNS is DNATed to VPN DNS
- if VPN DNS is absent: the link hook fails closed
- the Qubes virtual DNS addresses are sourced from the legacy helper file when present, otherwise from QubesDB on current Qubes 4.3 templates

For hostname resolution before the tunnel exists:

- OpenVPN `remote hostname` entries are rewritten to IPv4 addresses resolved through `setenv vpn_dns`
- WireGuard `Endpoint = hostname:port` entries are rewritten to IPv4 addresses resolved through `DNS =`

This avoids relying on ordinary pre-tunnel resolver state for endpoint lookup when a hostname is used.

## OpenVPN and WireGuard

### OpenVPN

OpenVPN is the stronger fit for this design because it is a userspace client and the ProxyVM egress rule can restrict upstream access to the VPN process group directly.

Recommended practice:

- use `tun`
- use `remote-cert-tls server`
- provide `setenv vpn_dns "X.X.X.X Y.Y.Y.Y"` when `remote` uses a hostname
- provide `remote` as an IPv4 address if you want to avoid pre-connect DNS entirely

### WireGuard

WireGuard is supported through the dedicated `qubes-wg-handler.service`, not by auto-detecting inside the OpenVPN unit.

Important differences:

- WireGuard traffic is kernel-driven rather than a userspace socket owned by the VPN process
- the project adjusts the Qubes output policy for the WireGuard session, marks the tunnel interface, and installs DNS hooks around `wg-quick`
- strict hostname handling for WireGuard requires `DNS =` to be set in the config when `Endpoint` uses a hostname
- for the strongest and simplest deployment, use an IPv4 `Endpoint` address instead of a hostname

Functional status:

- OpenVPN: first-class path
- WireGuard: supported, but more operationally sensitive and still best treated as experimental compared to OpenVPN

## Dependency Checks

`qubes-vpn-setup` now includes a distro-aware dependency checker:

```bash
sudo /usr/lib/qubes/qubes-vpn-setup --check-deps
```

It currently checks for:

- Qubes networking agent package
- `nftables`
- `python3`
- `xterm`
- notification support package (`libnotify-bin` on Debian, `libnotify` on Fedora)
- `openvpn`
- `wireguard-tools`

The installer runs this check automatically and stops if required packages are missing.

## Staged Deployment

The codebase now supports staging configuration after AppVM creation.

This means you can:

1. install the package in the TemplateVM
2. create the ProxyVM
3. run `--config-openvpn` or `--config-wireguard` in the ProxyVM
4. add `vpn-client.conf`, certs, keys, and optional user/password later
5. start the service only after the config actually exists

The service now requires `/rw/config/vpn/vpn-client.conf`, so it will not loop pointlessly before configuration has been added.

To add username/password later:

```bash
sudo /usr/lib/qubes/qubes-vpn-setup --userpass
```

## Notifications and Operator UX

This project currently uses:

- `notify-send` for status popups
- `xterm` for credential prompts when needed

For notification support in a TemplateVM, this project checks for the package that provides `notify-send`, but actual display still depends on the template having a desktop session capable of showing notifications. Full Debian/Fedora templates are a better fit than minimal templates.

Observed behavior on Qubes 4.3:

- direct `notify-send` testing may report a notifier service problem depending on the session state
- service-triggered notifications can still appear during actual VPN lifecycle events such as `Ready to start link` and `LINK IS UP`

So notification testing should be treated as a runtime/session concern, not a reliable indicator that the VPN hook path itself has failed.

## Verification

See:

- [Qubes 4.3 Firewall Verification](docs/QUBES_4_3_FIREWALL_VERIFICATION.md)
- [ProxyVM Template Quickstart](docs/PROXYVM_TEMPLATE_QUICKSTART.md)

Those documents include more specific citations placed next to the verification and deployment procedures they support.

For backend selection itself, a successful config command should leave exactly one of these files present:

- `/rw/config/vpn/backend-openvpn`
- `/rw/config/vpn/backend-wireguard`

## Command Summary

```bash
sudo bash ./install
sudo /usr/lib/qubes/qubes-vpn-setup --check-deps
sudo /usr/lib/qubes/qubes-vpn-setup --config-openvpn
sudo /usr/lib/qubes/qubes-vpn-setup --config-wireguard
sudo /usr/lib/qubes/qubes-vpn-setup --userpass
ls -l /rw/config/vpn/backend-openvpn /rw/config/vpn/backend-wireguard
sudo systemctl restart qubes-vpn-handler.service
sudo systemctl status qubes-vpn-handler.service
sudo journalctl -u qubes-vpn-handler.service -b
# WireGuard uses qubes-wg-handler.service for the systemctl/journalctl commands.
```

# ProxyVM Template Quickstart

This quickstart is for deploying the project from a TemplateVM into a dedicated VPN ProxyVM on Qubes OS 4.3.

## Tested Versions

Last explicitly observed working baseline during this work:

- Qubes OS: 4.3.0
- Template: `debian-13-minimal` from the Qubes repository
- OpenVPN: 2.6.14
- Firewall backend: `nftables`

## Official References

- Debian template documentation:
  https://doc.qubes-os.org/en/latest/user/templates/debian/debian.html
- Qubes firewall documentation:
  https://doc.qubes-os.org/en/latest/user/security-in-qubes/firewall.html
- Qubes OS 4.3 release notes:
  https://doc.qubes-os.org/en/latest/developer/releases/4_3/release-notes.html

These references support the Qubes-version and template assumptions used in this deployment guide.

## Non-official References

The backend-specific deployment notes in this guide were also informed by upstream project materials:

- Upstream project repository:
  https://github.com/tasket/Qubes-vpn-support
- Upstream `nftables` branch:
  https://github.com/tasket/Qubes-vpn-support/tree/replace-iptables-with-nftables
- Upstream patch discussion referenced during this work:
  https://github.com/tasket/Qubes-vpn-support/pull/71

These references were used for project compatibility and migration context, especially around `nftables`, OpenVPN hooks, and WireGuard examples.

## 1. Choose a Template

Recommended:

- `debian-13-xfce`
- a current Fedora full template

Full templates are preferred over minimal templates because notifications and `xterm` prompts are expected to work.

Reference:

- Qubes Debian template documentation:
  https://doc.qubes-os.org/en/latest/user/templates/debian/debian.html

## 2. Install Required Packages In the Template

Copy this repo into the TemplateVM and run:

```bash
cd Qubes-vpn-support
sudo bash ./install
```

The installer will stop and tell you which packages are missing.

You can also run the dependency check directly:

```bash
sudo /usr/lib/qubes/qubes-vpn-setup --check-deps
```

Typical Debian command:

```bash
sudo apt-get install -y qubes-core-agent-networking nftables python3 xterm libnotify-bin openvpn wireguard-tools
```

Typical Fedora command:

```bash
sudo dnf install -y qubes-core-agent-networking nftables python3 xterm libnotify openvpn wireguard-tools
```

## 3. Install Into the Template

Still in the TemplateVM:

```bash
cd Qubes-vpn-support
sudo bash ./install
```

Then shut down the TemplateVM.

## 4. Create the VPN ProxyVM

Create a new AppVM that:

- uses the prepared template
- has `provides network` enabled
- uses your normal firewall qube as `netvm`, usually `sys-firewall`
- has the Qubes service `vpn-handler-openvpn` enabled if you are using the shipped OpenVPN service model

Do not attach downstream VMs yet.

Reference:

- Qubes firewall and networking model:
  https://doc.qubes-os.org/en/latest/user/security-in-qubes/firewall.html

## 5. Initialize the ProxyVM

Start the new ProxyVM and run:

```bash
sudo /usr/lib/qubes/qubes-vpn-setup --config
```

This prepares:

- `/rw/config/vpn/`
- the Qubes firewall hook symlink
- the local service integration

It does not require `vpn-client.conf` or credentials yet.

## 6. Add VPN Configuration Later

Place your provider files into:

```bash
/rw/config/vpn/
```

At minimum:

- `vpn-client.conf`

As needed:

- CA files
- client certificates
- client keys
- CRLs
- static key files

## 7. Add Authentication Later If Needed

If your provider requires username/password:

```bash
sudo /usr/lib/qubes/qubes-vpn-setup --userpass
```

If your provider uses certificate-only auth, leave `userpassword.txt` disabled.

## 8. Configure Strict DNS

For OpenVPN hostname remotes:

```conf
setenv vpn_dns "X.X.X.X Y.Y.Y.Y"
remote vpn.example.net 1194
```

For WireGuard hostname endpoints:

```ini
DNS = X.X.X.X, Y.Y.Y.Y
Endpoint = vpn.example.net:51820
```

For the strictest deployment, use IPv4 endpoint addresses instead of hostnames.

## 9. Start the Service

Once `vpn-client.conf` exists:

```bash
sudo systemctl restart qubes-vpn-handler.service
sudo systemctl status qubes-vpn-handler.service
```

The unit now requires `/rw/config/vpn/vpn-client.conf`, so it will stay inactive until the config is actually present.

## 10. Verify Before Attaching Downstream VMs

In the ProxyVM:

```bash
sudo journalctl -u qubes-vpn-handler.service -b
sudo nft list chain ip qubes custom-forward
sudo nft list chain ip qubes dnat-dns
cat /proc/sys/net/ipv6/conf/all/disable_ipv6
```

Verify:

- tunnel interface is up
- `dnat-dns` rules exist after connect
- IPv6 is disabled
- forwarding rules show the downstream-to-VPN allow and stateful return rule
- `cat /var/run/qubes/qubes-vpn-ns` contains the VPN DNS values captured by the up hook

Only then should you attach downstream AppVMs to this ProxyVM.

If `dnat-dns` is empty even though the tunnel is up, verify whether the ProxyVM provides Qubes virtual DNS through:

```bash
qubesdb-read /qubes-netvm-primary-dns
qubesdb-read /qubes-netvm-secondary-dns
```

This repository now supports that Qubes 4.3 path directly and does not require `/var/run/qubes/qubes-ns` to exist.

Reference:

- Qubes firewall documentation for local and managed firewall verification:
  https://doc.qubes-os.org/en/latest/user/security-in-qubes/firewall.html

## WireGuard Notes

WireGuard is supported through the included override example:

```text
files-main/qubes-vpn-handler.service.d/10_wg.conf.example
```

Copy it into a live override with whatever naming scheme you prefer and adjust it to your deployment.

OpenVPN remains the simpler and stronger match for this project’s egress restriction model. WireGuard works, but it is more sensitive to config details and should be tested carefully before trusting it for the same operational guarantees.

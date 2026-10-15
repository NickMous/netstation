# netstation

An Ansible collection that turns a Debian server into an [LTSP](https://ltsp.org)
server for **stateless, network-booted RDP thin clients**.

Clients have no OS of their own. They boot from the network into a small
Debian/XFCE desktop that logs in automatically, opens a web portal in Firefox
and opens downloaded `.rdp` files in FreeRDP. Nothing is saved: every boot and
every logout starts clean. All maintenance happens in one place, on the server.

- **Reproducible.** The server and the client image are described entirely by
  your inventory. A broken server is fixed by running the playbook again.
- **Fresh images.** Every client image is built from scratch with `mmdebstrap`,
  so it contains exactly what the configuration says and nothing left over.
- **Safe rollout.** New images go to *staging* first. A test client boots
  staging; everyone else stays on *production* until you promote. Previous
  production images are kept for rollback.
- **Scheduled updates.** A weekly staging build picks up Debian security
  updates; promoting it stays a manual decision.
- **Remote management.** Optional WireGuard VPN, Cockpit web console, SSH and
  Veyon on the clients.

## How it fits together

```
 client ──PXE──▶ dnsmasq (proxy or full DHCP, TFTP) ──▶ iPXE
        ──HTTP─▶ nginx: LTSP menu, kernel, initrd
        ──NFS──▶ /srv/ltsp/images/netstation.img (squashfs, read-only)
                 + a RAM overlay, so every boot starts clean
```

| Playbook | What it does |
|---|---|
| `nickmous.netstation.installer_iso` | Build an unattended Debian installer ISO for the server |
| `nickmous.netstation.server` | Install or repair the server (LTSP, dnsmasq, NFS, nginx, image tooling, WireGuard, Cockpit) |
| `nickmous.netstation.image_build` | Build a new staging image from scratch |
| `nickmous.netstation.image_promote` | Make the staging image the production image |
| `nickmous.netstation.image_rollback` | Restore a previous production image |
| `nickmous.netstation.status` | Images, booted clients, services |
| `nickmous.netstation.wake` | Wake clients with Wake-on-LAN |

On the server, the same actions are available as `sudo netstation-image
build|promote|rollback|status`.

## Getting started

You need a machine to run Ansible from (Debian/Ubuntu below), a server or VM
for netstation, and clients that can network-boot. Everything else is set up
by the playbooks.

**0. Your machine.** Install the tools and start a private configuration
repository from `examples/`:

```bash
sudo apt install ansible git xorriso
cp -r examples ~/my-netstation && cd ~/my-netstation
git init
ansible-galaxy collection install -r requirements.yml
```

**1. Settings.** Edit `inventory/hosts.yml` (the server's name, address and
admin user) and `inventory/group_vars/netstation_servers/main.yml`. Then
create the vault with the admin password:

```bash
cp inventory/group_vars/netstation_servers/vault.yml.example inventory/group_vars/netstation_servers/vault.yml
nano inventory/group_vars/netstation_servers/vault.yml
ansible-vault encrypt inventory/group_vars/netstation_servers/vault.yml
```

**2. Installer.** Build an ISO that installs Debian without any questions,
with the admin account, your SSH key and the network settings from step 1:

```bash
ansible-playbook nickmous.netstation.installer_iso --ask-vault-pass
```

The ISO ends up in `build/<server name>.iso`.

**3. Install the server.** Boot the server or VM from the ISO. It erases the
first disk, installs Debian and reboots, which takes about ten minutes.
Remove the ISO afterwards. For a Hyper-V VM: Generation 2, with Secure Boot
on the "Microsoft UEFI Certificate Authority" template (or off).

**4. Set up netstation and build the first image:**

```bash
ansible-playbook nickmous.netstation.server --ask-vault-pass
ansible-playbook nickmous.netstation.image_build --ask-vault-pass
ansible-playbook nickmous.netstation.image_promote --ask-vault-pass
```

**5. Boot a client** from the network. PXE boot must be enabled in its
BIOS/UEFI and Secure Boot disabled.

Requirements: the server must be on the same network segment as the clients.
With `netstation_dhcp_mode: proxy` (the default), another DHCP server on that
network hands out the IP addresses.

## Configuration

Every variable is documented, with its default, in `roles/*/defaults/main.yml`:

- `roles/common/defaults/main.yml`: image names, session user, Debian release,
  locale, Veyon
- `roles/server/defaults/main.yml`: DHCP mode, NAT for two-network-card
  setups, known clients and test clients, extra `ltsp.conf` settings
- `roles/image/defaults/main.yml`: homepage, wallpaper, RDP options, Firefox
  policies, packages, extra files, scheduled builds
- `roles/installer/defaults/main.yml`: admin account, network settings and
  extra network cards for the installed server
- `roles/wireguard/defaults/main.yml` and `roles/cockpit/defaults/main.yml`

All variables share the `netstation_` prefix, so you can set them in one
`group_vars` file.

## Notes

- The server manages `/etc/ltsp/ltsp.conf` and `/etc/dnsmasq.d/ltsp-dnsmasq.conf`.
  Don't run `ltsp dnsmasq` by hand; it would overwrite the HTTP-boot setup.
- The session user exists only on the server, with an empty password. LTSP
  merges it into each client, where its home folder lives in RAM.
- [docs/runbook.md](docs/runbook.md) covers day-to-day operation and
  troubleshooting.

## License

MIT

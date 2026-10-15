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
| `nickmous.netstation.server` | Install or repair the server (LTSP, dnsmasq, NFS, nginx, image tooling, WireGuard, Cockpit) |
| `nickmous.netstation.image_build` | Build a new staging image from scratch |
| `nickmous.netstation.image_promote` | Make the staging image the production image |
| `nickmous.netstation.image_rollback` | Restore a previous production image |
| `nickmous.netstation.status` | Images, booted clients, services |

On the server, the same actions are available as `sudo netstation-image
build|promote|rollback|status`.

## Getting started

Your configuration lives in a separate, private repository. `examples/` is a
complete starting point for one:

```bash
cp -r examples ~/my-netstation && cd ~/my-netstation
git init
ansible-galaxy collection install -r requirements.yml
# edit inventory/hosts.yml and inventory/group_vars/netstation_servers/main.yml
cp inventory/group_vars/netstation_servers/vault.yml.example inventory/group_vars/netstation_servers/vault.yml
ansible-vault encrypt inventory/group_vars/netstation_servers/vault.yml

ansible-playbook nickmous.netstation.server --check --diff   # preview
ansible-playbook nickmous.netstation.server
ansible-playbook nickmous.netstation.image_build
ansible-playbook nickmous.netstation.image_promote
```

Requirements: a Debian 13 (trixie) server with SSH access and sudo, on the
same network segment as the clients. Clients need PXE boot enabled and Secure
Boot disabled.

## Configuration

Every variable is documented, with its default, in `roles/*/defaults/main.yml`:

- `roles/common/defaults/main.yml`: image names, session user, Debian release,
  locale, Veyon
- `roles/server/defaults/main.yml`: DHCP mode, known clients and test clients,
  extra `ltsp.conf` settings
- `roles/image/defaults/main.yml`: homepage, wallpaper, RDP options, Firefox
  policies, packages, extra files, scheduled builds
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

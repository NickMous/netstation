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
- **Secure defaults.** A firewall that only lets in what netstation needs, and
  automatic security updates on the server.
- **Power schedule.** Wake the clients in the morning; they switch themselves
  off when nobody uses them.
- **Remote Mac.** Optional: people control their own Mac from any client with
  RustDesk, on the client's keyboard, mouse and screens, through a RustDesk
  server on this server.
- **Alerts.** Optional Pushover alerts for failed builds and services, a
  hanging dnsmasq, a full disk, failed security updates and pending reboots.

## How it fits together

```
 client ──PXE──▶ dnsmasq (proxy or full DHCP, TFTP) ──▶ iPXE
        ──HTTP─▶ nginx: LTSP menu, kernel, initrd
        ──NFS──▶ /srv/ltsp/images/netstation.img (squashfs, read-only)
                 + a RAM overlay, so every boot starts clean
```

Your settings live in a private configuration repository (an Ansible
inventory). From there you run the playbooks of this collection against the
server, which builds the client images itself.

## Quick look

```bash
ansible-playbook nickmous.netstation.installer_iso    # unattended Debian installer for the server
ansible-playbook nickmous.netstation.server           # set up or repair the server
ansible-playbook nickmous.netstation.image_build      # build a staging image
ansible-playbook nickmous.netstation.image_promote    # make it production
```

## Documentation

- [Getting started](docs/getting-started.md): from an empty machine to the
  first client booting
- [Configuration](docs/configuration.md)
- [Updating the clients](docs/updating-clients.md): build, test, promote,
  roll back
- [Managing clients](docs/clients.md): test clients, Wake-on-LAN, power
  schedule
- [Status and alerts](docs/monitoring.md)
- [Remote management](docs/remote-management.md): Cockpit, WireGuard, SSH,
  Veyon
- [Remote Mac](docs/remote-mac.md)
- [Troubleshooting](docs/troubleshooting.md)
- [Commands](docs/commands.md): every playbook and server command

Every variable is documented, with its default, in `roles/*/defaults/main.yml`.

## License

MIT

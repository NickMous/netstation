# netstation

netstation is an Ansible collection that turns a Debian server into an
[LTSP](https://ltsp.org) server for **stateless, network-booted RDP thin
clients**.

Clients have no OS of their own. They boot from the network into a small
Debian/XFCE desktop that logs in automatically, opens a web portal in Firefox
and opens downloaded `.rdp` files in FreeRDP. Nothing is saved: every boot and
every logout starts clean. All maintenance happens in one place, on the
server.

## What you get

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
  RustDesk, on the client's keyboard, mouse and screens.
- **Alerts.** Optional Pushover alerts for failed builds and services, a
  hanging dnsmasq, a full disk, failed security updates, pending reboots and
  Remote Desktop problems on the clients.

## How it fits together

```
 client ──PXE──▶ dnsmasq (proxy or full DHCP, TFTP) ──▶ iPXE
        ──HTTP─▶ nginx: LTSP menu, kernel, initrd
        ──NFS──▶ /srv/ltsp/images/netstation.img (squashfs, read-only)
                 + a RAM overlay, so every boot starts clean
```

You keep your settings in a private **configuration repository** (an Ansible
inventory) on your own machine. From there you run the playbooks of this
collection against the server. The server builds the client images itself.

There are always two images on the server:

- **production**: what every client boots.
- **staging**: the newest build. Only test clients boot it, until you promote
  it to production.

## Documentation

| Page | What's in it |
|---|---|
| [Getting started](getting-started.md) | From an empty machine to the first client booting |
| [Configuration](configuration.md) | How settings work, and the most important ones per topic |
| [Updating the clients](updating-clients.md) | Build, compare, test, promote and roll back images |
| [Managing clients](clients.md) | Adding clients, test clients, waking them and the power schedule |
| [Status and alerts](monitoring.md) | Seeing what's going on, and Pushover alerts |
| [Remote management](remote-management.md) | Cockpit, WireGuard, SSH into clients and Veyon |
| [Remote Mac](remote-mac.md) | Controlling your own Mac from a client with RustDesk |
| [Troubleshooting](troubleshooting.md) | What to check when something doesn't work |
| [Commands](commands.md) | Every playbook and every command on the server |

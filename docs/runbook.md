# Runbook

Day-to-day operation of a netstation server. Run the `ansible-playbook`
commands from your configuration repository. Add `--ask-vault-pass` if you
haven't set up a vault password file.

## Updating the clients

1. **Build** a staging image. This also happens automatically every week.

   ```bash
   ansible-playbook nickmous.netstation.image_build
   ```

2. **Test** it: reboot the test client. It boots staging by default; its boot
   menu also lets you pick production.
3. **Promote** it. Clients switch to it on their next reboot.

   ```bash
   ansible-playbook nickmous.netstation.image_promote
   ```

4. Something wrong after all? **Roll back** to the previous production image:

   ```bash
   ansible-playbook nickmous.netstation.image_rollback
   # or pick one from the status list:
   ansible-playbook nickmous.netstation.image_rollback -e netstation_rollback_build=20261015T0300Z
   ```

   Rolling back archives the current production image, so running it twice
   brings you back to where you started.

## Changing settings

| What | Where | Then run |
|---|---|---|
| Homepage, wallpaper, RDP options, packages, Firefox | `group_vars` (`netstation_homepage`, ...) | `image_build`, test, `image_promote` |
| Files in the image | `files/rootfs/...` | `image_build`, test, `image_promote` |
| Known clients, test client, per-client settings | `netstation_clients` | `server` |
| DHCP, WireGuard peers, other server settings | `group_vars` | `server` |

Always preview server changes first:

```bash
ansible-playbook nickmous.netstation.server --check --diff
```

## Status

```bash
ansible-playbook nickmous.netstation.status
```

Or on the server: `sudo netstation-image status`. It shows the production and
staging builds, the rollback targets, the scheduled build, which clients are
booted right now, and whether dnsmasq, NFS and nginx are running.

Cockpit is at `https://<server>:9090` (log in with your server account): logs,
services, updates and a terminal.

## Waking clients

Clients listed in `netstation_clients` can be switched on from the server:

```bash
ansible-playbook nickmous.netstation.wake                                  # all of them
ansible-playbook nickmous.netstation.wake -e '{"netstation_wake": ["pc01"]}' # by hostname or MAC
```

Each client's BIOS/UEFI must allow Wake-on-LAN (often under power
management). The image keeps the network card armed when the client shuts
down. If clients don't wake, set `netstation_wake_broadcast` to the
broadcast address of their network (e.g. `192.168.1.255`).

## Veyon computer list

Veyon Master needs a list of the clients. Generate it from the clients that
are booted right now, then import it with the two commands the playbook
prints:

```bash
ansible-playbook nickmous.netstation.veyon_computers
```

Client IP addresses can change, so run it again when Veyon shows clients as
offline that are switched on. DHCP reservations for the clients avoid that.

## Adding a client

PXE boot must be enabled in the BIOS/UEFI and Secure Boot disabled. A new
client then works without any configuration. Add it to `netstation_clients`
only to give it a fixed name, make it a test client, or give it its own
settings:

```yaml
netstation_clients:
  - mac: "d8:9e:f3:37:53:a1"
    hostname: reception01
```

Then run the `server` playbook.

## Troubleshooting

### A client doesn't network-boot

Watch the server while the client boots:

```bash
sudo journalctl -u dnsmasq -f
```

- **Nothing about the client's MAC address:** its boot request doesn't reach
  the server. Check that it really network-boots (boot menu, often F12) and
  that it's on the same network segment.
- **The request arrives but the client gets no answer:** set
  `netstation_dnsmasq_log_dhcp: true`, run `server`, and look at the detailed
  log. In proxy mode, also check that the other DHCP server doesn't set
  options 60, 66 or 67.
- **dnsmasq hangs (high CPU, clients stop booting):** some dnsmasq versions
  hang in proxy mode. Pin a working version with `netstation_dnsmasq_version`.

### A client boots but stops with an error

Add a debug shell for one client, run `server`, and reboot that client:

```yaml
netstation_clients:
  - mac: "d8:9e:f3:37:53:a1"
    settings:
      DEBUG_SHELL: 1
```

### The client shows a login screen or loops back to it

On the client, press Ctrl+Alt+F2, log in with your server account, and run:

```bash
ls -la /home/kiosk                       # must exist and be owned by kiosk
findmnt /home/kiosk                      # must print nothing (home in RAM)
grep -rn autologin /etc/lightdm/
sudo tail -40 /var/log/lightdm/lightdm.log
```

### Logging in to a client

```bash
ssh <your-server-account>@<client-ip>
```

Clients get new SSH host keys on every boot, so expect a host key warning.

### The scheduled staging build failed

```bash
sudo journalctl -u netstation-staging-build.service -n 100
```

Production is never touched by a failed build.

# Runbook

Day-to-day operation of a netstation server. Run the `ansible-playbook`
commands from your configuration repository. Add `--ask-vault-pass` if you
haven't set up a vault password file.

## Updating the clients

1. **Build** a staging image. This also happens automatically every week.

   ```bash
   ansible-playbook nickmous.netstation.image_build
   ```

2. **Check** what changed compared with production. `image_build` shows this
   at the end; after a scheduled build, run:

   ```bash
   ansible-playbook nickmous.netstation.image_diff
   ```

   It lists updated, added and removed packages, and the changes to the files
   netstation puts in the image (settings, Firefox policies, scripts). It also
   copies both versions to `build/image-diff/` in your configuration
   repository, to open in a diff viewer you install yourself:

   ```bash
   git diff --no-index build/image-diff/production build/image-diff/staging
   meld build/image-diff/production build/image-diff/staging
   ```

   On the server, `sudo netstation-image diff` shows the same, colored and
   paged like `git diff`.

3. **Test** it: reboot the test client. It boots staging by default; its boot
   menu also lets you pick production.
4. **Promote** it. Clients switch to it on their next reboot.

   ```bash
   ansible-playbook nickmous.netstation.image_promote
   ```

5. Something wrong after all? **Roll back** to the previous production image:

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
| Homepage, wallpaper, boot splash, desktop shortcuts, RDP options and pass-through, idle logout, packages, Firefox | `group_vars` (`netstation_homepage`, ...) | `image_build`, test, `image_promote` |
| Files in the image | `files/rootfs/...` | `image_build`, test, `image_promote` |
| Known clients, test client, per-client settings | `netstation_clients` | `server` |
| DHCP, firewall, WireGuard peers, other server settings | `group_vars` | `server` |
| Remote Mac on or off | `netstation_remote_mac_enabled` | `server`, then `image_build`, test, `image_promote` |

Always preview server changes first:

```bash
ansible-playbook nickmous.netstation.server --check --diff
```

## Status

```bash
ansible-playbook nickmous.netstation.status
```

Only the clients that are on, with IP and MAC address:

```bash
ansible-playbook nickmous.netstation.clients
```

Or on the server: `sudo netstation-image status`. It shows the production and
staging builds, the rollback targets, the scheduled build, which clients are
booted right now, and whether dnsmasq, NFS and nginx are running.

Cockpit is at `https://<server>:9090` (log in with your server account): logs,
services, updates and a terminal.

## Alerts

With `netstation_pushover_app_token` and `netstation_pushover_user_key` set,
the server sends Pushover alerts:

| Alert | When |
|---|---|
| *unit* failed | The scheduled build, dnsmasq, NFS or nginx failed, with its last log lines |
| Problem / Resolved | Every 10 minutes the server checks: a service stopped, a disk over 85% full, the automatic security updates failed, or the server needs a reboot for a new kernel. It alerts once when a problem appears and once when it's gone. |
| dnsmasq restarted / still hangs / works again | The dnsmasq watchdog (every 2 minutes) found dnsmasq hanging and restarted it |
| Staging build ready to test | A staging build finished, with what changed compared with production |
| Clients still on | At the `netstation_power_report` times, when any client is still on |
| *client*: Remote Desktop crashed | xfreerdp3 crashed on a client (with a sound), with its last errors |
| *client*: Remote Desktop error / can't connect / account blocked / password expired | xfreerdp3 stopped with an error (without a sound). Wrong passwords and normal disconnects aren't reported. |

Alerts about a client name it by `label` and `hostname` from
`netstation_clients`, or by its hostname or IP address. A client sends at most
one alert every 10 minutes (`netstation_alerts_client_quiet_minutes`); the
others are only logged: `sudo journalctl -t netstation-client-report`.

Security updates install by themselves, but the server never reboots by
itself: clients stop working while it restarts. Reboot it outside opening
hours when the reboot alert comes:

```bash
ssh <server> sudo systemctl reboot
```

To see the alerts the server sent, also when Pushover couldn't be reached:

```bash
sudo journalctl -t netstation-alert
```

Run a check by hand with `sudo netstation-health`.

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

## Power schedule

With a power schedule the server wakes the clients and the clients switch
themselves off when nobody uses them:

```yaml
netstation_power_wake: ["08:45"]          # Wake-on-LAN for all known clients
netstation_power_off:
  - at: "10:00"                           # not used since switched on
  - at: "18:30"
    idle_minutes: 15                      # no input for 15 minutes
netstation_power_report: ["22:00"]        # Pushover: clients still on
```

Times are systemd calendar times, so `"Mon..Fri 08:45"` works too. The wake
reaches the clients in `netstation_clients` and every client the server has
seen before (`/var/lib/netstation/known-clients`).

A client in `netstation_clients` can have its own wake times. They replace
`netstation_power_wake` for that client:

```yaml
netstation_power_wake: ["Mon..Fri 08:45"]
netstation_clients:
  - mac: "d8:9e:f3:37:53:a1"
    label: Reception
    wake: ["Mon..Fri 07:30", "Sat 08:45"]   # earlier, and also on Saturday
  - mac: "d8:9e:f3:37:53:a2"
    label: Spare
    wake: []                                # never woken by the schedule
```

Clients with the same `wake` list share a timer
(`netstation-power-wake-N.timer`); `systemctl list-timers 'netstation-power-*'`
shows when each fires next. Switching off is part of
the image, so after changing `netstation_power_off` run `image_build`, test,
and `image_promote`; the other two only need `server`.

Try it by hand on the server:

```bash
sudo netstation-power wake      # or: wake-1 (a group), learn, report
```

## Veyon computer list

Veyon Master needs a list of the clients. Generate it from the clients that
are booted right now, then import it with the two commands the playbook
prints:

```bash
ansible-playbook nickmous.netstation.veyon_computers
```

Client IP addresses can change, so run it again when Veyon shows clients as
offline that are switched on. DHCP reservations for the clients avoid that.

## Remote Mac

With `netstation_remote_mac_enabled`, people control their own Mac from any
client: the client's keyboard, mouse and screens work the Mac, as with
Remote Desktop. It uses [RustDesk](https://rustdesk.com): the server runs a
RustDesk ID and relay server (`rustdesk-hbbs`, `rustdesk-hbbr`), reachable
only from `netstation_remote_mac_networks`, and the image has the RustDesk
client and a desktop icon (`netstation_remote_mac_icon_name`).

At the client: click the icon, type the Mac's RustDesk ID, then the
password RustDesk on the Mac shows. Each Mac screen opens full screen on its
own client screen. The XFCE panels are gone during the session, and Alt and
the Windows key swap places, so the key next to the space bar is Cmd. Close
the RustDesk windows (the toolbar at the top of the screen) to get the
desktop back.

**Turning it on.** Set `netstation_remote_mac_enabled: true`, run the
`server` playbook, then build, test and promote an image. The `server`
playbook prints the ID server and key for the Macs. It makes a key on its
first run; save it in the vault right away, or a reinstalled server gets a
new one and every Mac has to be set up again:

```bash
ssh netadmin@<server> sudo cat /var/lib/rustdesk-server/id_ed25519
```

```yaml
# vault.yml
vault_rustdesk_private_key: "..."
# main.yml
netstation_rustdesk_private_key: "{{ vault_rustdesk_private_key }}"
```

**Setting up the Macs.** Make the handout for the Mac users:

```bash
ansible-playbook nickmous.netstation.mac_handout
```

It writes two files to `build/mac-handout/` in your configuration
repository: a manual (HTML; open it in a browser, print it or save it as
PDF) and a setup script. Hand them out, e.g. by email or AirDrop. Nothing in
them is secret: the script has the server's address and its public key, and
the Mac users need no access to the server or the vault.

On the Mac, the user runs the script in Terminal, as the manual explains. It
asks for the Mac's password once, and:

- installs RustDesk and [SimpleDisplay](https://github.com/SamuelRioTz/SimpleDisplay)
  (pinned versions, checksums checked) in Applications;
- sets this server and its key in RustDesk, before RustDesk's service is
  installed, so the service takes them over;
- opens the settings for Screen Recording and Accessibility, which only the
  user can allow on a Mac without MDM;
- adds an **office mode** app (`netstation_mac_office_mode_name`): clicked
  once, it adds virtual screens (`netstation_mac_office_displays`) and keeps
  the Mac awake; clicked again, it removes them. RustDesk shows as many Mac
  screens as the client has, starting with the Mac's main screen, usually
  its own. `netstation_mac_office_builtin_off: true` also switches the Mac's
  own screen off, so the client shows only the virtual screens;
- shows the Mac's RustDesk ID.

Running the script again updates the apps and the settings. Make and hand
out a new handout after changing a version or the server's key. The manual
is English; `netstation_mac_handout_manual_template` points at your own (e.g.
translated) template, and `netstation_mac_handout_messages` holds what the
script and office mode say.

**Updating RustDesk.** Change `netstation_rustdesk_client_version` and
`netstation_rustdesk_client_checksum` (image),
`netstation_rustdesk_server_version` and `netstation_rustdesk_server_checksums`
(server), or `netstation_mac_rustdesk_version` and
`netstation_mac_rustdesk_checksums` (Macs, then a new handout). The checksums
are on the GitHub release pages.

## Adding a client

PXE boot must be enabled in the BIOS/UEFI and Secure Boot disabled. A new
client then works without any configuration. Add it to `netstation_clients`
only to give it a fixed name, label it, make it a test client, or
give it its own settings:

```yaml
netstation_clients:
  - mac: "d8:9e:f3:37:53:a1"
    hostname: reception01
    label: Anna de Vries      # who or what it's for: shown in alerts and by the clients playbook
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
  hang in proxy mode. A watchdog checks every 2 minutes and restarts it (see
  `sudo journalctl -u netstation-dnsmasq-watchdog`). Check by hand with
  `sudo netstation-dnsmasq-check` after a client has booted; it measures CPU
  use, unread packets and whether TFTP still answers. If it keeps happening,
  pin a working version with `netstation_dnsmasq_version`.

### A client boots but stops with an error

Add a debug shell for one client, run `server`, and reboot that client:

```yaml
netstation_clients:
  - mac: "d8:9e:f3:37:53:a1"
    settings:
      DEBUG_SHELL: 1
```

### A client hangs while it starts

The boot splash hides the messages. Press Esc on the client to see them.
To boot one client without the splash, give it its own kernel parameters,
run `server`, and reboot that client:

```yaml
netstation_clients:
  - mac: "d8:9e:f3:37:53:a1"
    settings:
      KERNEL_PARAMETERS: nosplash
```

### The client shows a login screen or loops back to it

On the client, press Ctrl+Alt+F2, log in with your server account, and run:

```bash
ls -la /home/kiosk                       # must exist and be owned by kiosk
findmnt /home/kiosk                      # must print nothing (home in RAM)
grep -rn autologin /etc/lightdm/
sudo tail -40 /var/log/lightdm/lightdm.log
```

### Remote Desktop fails on a client

The dialog on the client says roughly what went wrong, and crashes and errors
come in as alerts. For xfreerdp3's full output, log in to the client (below)
while it's still on, and run:

```bash
sudo journalctl -t netstation-rdp
```

The client keeps its journal in memory, so it's gone after a reboot.

### Remote Mac doesn't work

- **"ID does not exist" or it can't connect:** the Mac isn't using this
  server (Settings > Network on the Mac), isn't on the network, or RustDesk
  isn't running on it. Check the server with
  `sudo systemctl status rustdesk-hbbs rustdesk-hbbr` and
  `sudo journalctl -u rustdesk-hbbs`.
- **"Key mismatch":** the key on the Mac isn't the server's. Show the right
  one with `sudo cat /var/lib/rustdesk-server/id_ed25519.pub`.
- **The screen shows, but mouse and keyboard do nothing:** RustDesk on the
  Mac lacks Accessibility (see Setting up the Macs). Reopen RustDesk after
  allowing it.
- **Lag:** usually the Mac's Wi-Fi. A network cable fixes it; Retina
  resolutions on the Mac also cost speed.
- **Alt and Cmd aren't swapped,** or still swapped after the session: log in
  to the client and check `systemctl status keyd`. A logout always stops it.

On the client, `journalctl -t netstation-mac` shows each session and
RustDesk's output.

### Logging in to a client

```bash
ssh <your-server-account>@<client-ip>
```

Clients get new SSH host keys on every boot, so expect a host key warning.

### Something can't reach the server

The server's firewall only lets in SSH, network boot, NFS, and Cockpit and
WireGuard when they're on. It logs what it drops:

```bash
sudo journalctl -k --grep netstation-firewall
```

Allow more with `netstation_firewall_extra_rules` (nftables rules, e.g.
`tcp dport 8080 accept`) and run `server`.

### The scheduled staging build failed

```bash
sudo journalctl -u netstation-staging-build.service -n 100
```

Production is never touched by a failed build.

# Troubleshooting

## A client doesn't network-boot

Check that PXE boot is enabled in its BIOS/UEFI and Secure Boot is disabled.
Then watch the server while the client boots:

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
  pin a working version with `netstation_dnsmasq_version` (see
  [Configuration](02-configuration.md#network)).

## A client boots but stops with an error

Add a debug shell for one client, run `server`, and reboot that client:

```yaml
netstation_clients:
  - mac: "d8:9e:f3:37:53:a1"
    settings:
      DEBUG_SHELL: 1
```

## A client hangs while it starts

The boot splash hides the messages. Press Esc on the client to see them.
To boot one client without the splash, give it its own kernel parameters,
run `server`, and reboot that client:

```yaml
netstation_clients:
  - mac: "d8:9e:f3:37:53:a1"
    settings:
      KERNEL_PARAMETERS: nosplash
```

## The client shows a login screen or loops back to it

On the client, press Ctrl+Alt+F2, log in with your server account, and run:

```bash
ls -la /home/kiosk                       # must exist and be owned by kiosk
findmnt /home/kiosk                      # must print nothing (home in RAM)
grep -rn autologin /etc/lightdm/
sudo tail -40 /var/log/lightdm/lightdm.log
```

## Remote Desktop fails on a client

The dialog on the client says roughly what went wrong, and crashes and errors
come in as [alerts](05-monitoring.md#alerts). For xfreerdp3's full output,
[log in to the client](06-remote-management.md#ssh-into-a-client) while it's
still on, and run:

```bash
sudo journalctl -t netstation-rdp
```

The client keeps its journal in memory, so it's gone after a reboot.

Exit code 136 means a screen is mirrored: keep
`netstation_displays_side_by_side` on.

## Remote Mac doesn't work

- **"ID does not exist" or it can't connect:** the Mac isn't using this
  server (Settings > Network on the Mac), isn't on the network, or RustDesk
  isn't running on it. Check the server with
  `sudo systemctl status rustdesk-hbbs rustdesk-hbbr` and
  `sudo journalctl -u rustdesk-hbbs`.
- **"Key mismatch":** the key on the Mac isn't the server's. Show the right
  one with `sudo cat /var/lib/rustdesk-server/id_ed25519.pub`.
- **The screen shows, but mouse and keyboard do nothing:** RustDesk on the
  Mac lacks Accessibility (see [Setting up the Macs](07-remote-mac.md#setting-up-the-macs)).
  Reopen RustDesk after allowing it.
- **Lag:** usually the Mac's Wi-Fi. A network cable fixes it; Retina
  resolutions on the Mac also cost speed.
- **Alt and Cmd aren't swapped,** or still swapped after the session: log in
  to the client and check `systemctl status keyd`. A logout always stops it.

On the client, `journalctl -t netstation-mac` shows each session and
RustDesk's output.

## Clients don't wake

Check that Wake-on-LAN is allowed in the client's BIOS/UEFI (often under
power management). If it is, set `netstation_wake_broadcast` to the broadcast
address of the clients' network (e.g. `192.168.1.255`) and run `server`.

## Something can't reach the server

The server's firewall only lets in SSH, network boot, NFS, and Cockpit,
WireGuard and RustDesk when they're on. It logs what it drops:

```bash
sudo journalctl -k --grep netstation-firewall
```

Allow more with `netstation_firewall_extra_rules` (nftables rules, e.g.
`tcp dport 8080 accept`) and run `server`.

## The scheduled staging build failed

```bash
sudo journalctl -u netstation-staging-build.service -n 100
```

Production is never touched by a failed build.

## The server is broken

Run the `server` playbook again. It's safe to run at any time and puts every
setting back. If the server is beyond repair, reinstall it from the
[installer ISO](01-getting-started.md#3-build-the-installer), run `server`, then
build and promote a new image.

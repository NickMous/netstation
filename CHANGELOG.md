# Changelog

## 0.9.6 (2026-11-16)

- dnsmasq 2.91 to 2.93 no longer hang in proxy mode: the proxy `dhcp-range`
  has no `set:proxy` tag any more, which is what triggered the hang (Debian
  #1151644, Ubuntu LP #2166430). A dnsmasq pin is no longer needed for this.
- Changing `netstation_dnsmasq_version` from one pinned version to another
  works: the server role used to fail on the hold of the old version.
- The configuration and troubleshooting docs explain the dnsmasq hang and
  when a pin still makes sense.

## 0.9.5 (2026-11-16)

- The pages in `docs/` start with a number, so imlostincode.com shows them
  in reading order. `docs/index.md` is gone: the README is the overview. No
  changes to the roles or playbooks.

## 0.9.4 (2026-11-16)

- The documentation is split into pages in `docs/`: getting started,
  configuration, updating the clients, managing clients, status and alerts,
  remote management, Remote Mac, troubleshooting and commands. They replace
  `docs/runbook.md`, and the README is now a short introduction that links to
  them. No changes to the roles or playbooks.

## 0.9.3 (2026-11-16)

- A published release, or a change to the README, `docs/`, `examples/` or
  `imlostincode.json` on main, asks imlostincode.com to rebuild, so the docs
  there stay current. No changes to the roles or playbooks.

## 0.9.2 (2026-11-16)

- The repository has an `imlostincode.json` with the collection's card for
  the imlostincode.com website, where the docs will be. No changes to the
  roles or playbooks.

## 0.9.1 (2026-11-12)

- Signing out of Windows inside a Remote Desktop session (xfreerdp3 exit
  code 12) counts as a normal end of the session: no error dialog and no
  report. Takes an image build.

## 0.9.0 (2026-11-12)

- The new `mac_handout` playbook writes a handout for Remote Mac users to
  `build/mac-handout/`: a manual and a setup script, with nothing secret in
  them. The script installs RustDesk and SimpleDisplay on the Mac, sets the
  RustDesk server and its key, walks the user through the permissions only
  they can give, and adds an office mode app that switches virtual screens
  for the client on, keeping the Mac awake, and back off. The manual's template, the texts and the screens are settings.

## 0.8.0 (2026-11-12)

- **Remote Mac** (`netstation_remote_mac_enabled`): people control their own
  Mac from any client with RustDesk, on the client's keyboard, mouse and
  screens. The new `rustdesk_server` role runs a RustDesk ID and relay server
  (`rustdesk-hbbs`, `rustdesk-hbbr`) from the pinned release packages, with
  the key enforced, reachable only from `netstation_remote_mac_networks`. The
  image gets the RustDesk client (its own service masked: clients only
  connect out) and a "Mac" icon that asks for the Mac's RustDesk ID, puts
  each Mac screen on its own client screen, hides the XFCE panels and swaps
  Alt and the Windows key (keyd) until the session ends.
  `netstation_remote_mac_sound: false` keeps the Mac's sound on the Mac. See
  "Remote Mac" in the runbook. Takes `server` and an image build.
- `netstation-image build` installs packages from outside the package
  sources that the build context brings along (`debs/`), with their
  dependencies.

## 0.7.2 (2026-11-12)

- The client's speakers and headphones start at 100% at every login, so the
  Remote Desktop session's volume control alone sets how loud it is. Before,
  they started at WirePlumber's 40%, and changes on the panel were lost at
  logout with the fresh home folder. `netstation_audio_volume` sets the
  percentage; empty keeps WirePlumber's default. Takes an image build.

## 0.7.1 (2026-11-12)

- The panel has a volume icon (`xfce4-pulseaudio-plugin`) with a slider and
  mute, and the keyboard's volume keys work. Its mixer, `pavucontrol`, picks
  the output device, such as the monitor's speakers or the headphone jack.
  The workspace switcher is gone from the panel. The panel layout is now
  also used with `netstation_panel_show_address: false`, without the address.
  Takes an image build.

## 0.7.0 (2026-11-12)

- A client in `netstation_clients` can have its own wake times (`wake`),
  e.g. earlier or also on Saturday. They replace `netstation_power_wake` for
  that client; `wake: []` keeps it out of the scheduled wake. Clients with the
  same times share a `netstation-power-wake-N` timer. Takes `server`.

## 0.6.3 (2026-11-05)

- A screen switched on after the client started is put next to the others,
  not mirrored on top of them. Before, xfreerdp3 then refused to connect
  (exit code 136, "pre-connect failed") until the client was restarted. The
  new `netstation-displays` puts all connected screens side by side, left to
  right in xrandr's output order with the first one primary, at login and on
  every hotplug, and XFCE no longer acts on new screens or shows its display
  dialog. `netstation_displays_side_by_side: false` turns it off. Takes an
  image build.

## 0.6.2 (2026-11-03)

- When xfreerdp3 stops on an X error, such as `BadLength` after copying a lot
  to the clipboard, the client reports it as a crash. Before, Xlib's exit
  code 1 looked like a normal disconnect, so no alert came. Takes an image
  build.

## 0.6.1 (2026-10-29)

- Remote Desktop alerts include the message the user saw in the dialog, and
  keep the start of long error output instead of the end.
## 0.6.0 (2026-10-29)

- `netstation_clients` takes `label`: who or what the client is for (a
  person, a room, a desk). It shows in the clients-still-on alert, the
  `clients` playbook and the new client alerts.
- When xfreerdp3 crashes or fails with an error, the client reports it to the
  server, which sends an alert naming the client
  (`netstation_alerts_client_reports`, at most one per client every
  `netstation_alerts_client_quiet_minutes`). The dialog on the client now
  says what went wrong (wrong password, account blocked, can't connect), and
  a normal disconnect no longer shows "The connection failed". xfreerdp3's
  output goes to the client's journal (`journalctl -t netstation-rdp`).
  `netstation_rdp_messages` sets the dialog texts, e.g. in another language.
  Takes `server` and an image build.
## 0.5.2 (2026-10-26)

- With the boot splash, LightDM takes over the splash on console 1 and X
  keeps it on screen, which halves the black screen before the desktop.
  Console 1 has no text login any more; Ctrl+Alt+F2 still does.

## 0.5.1 (2026-10-26)

- `netstation_initrd_gpu_drivers` keeps only the listed GPU drivers (and
  their firmware) in the initrd. With the boot splash, Plymouth adds them
  all, which grew the initrd from about 40 to 95 MB; `[i915]` brings it back
  to about 57 MB.

## 0.5.0 (2026-10-26)

- The clients show a boot splash instead of boot messages
  (`netstation_boot_splash`). `netstation_boot_splash_image` shows a picture
  of your own; without one it's the computer maker's logo or a spinner.
  Takes `server` for the kernel parameters and an image build for Plymouth.
- `netstation_kernel_parameters` adds kernel parameters for all clients.

## 0.4.2 (2026-10-19)

- `image_diff` stops with a clear message when the server's tools are older
  than the collection (run `server` first), instead of a tar error.
- `netstation-image diff` rejects unknown options.

## 0.4.1 (2026-10-19)

- `image_diff` copies both versions to `build/image-diff/` in your
  configuration repository, to open in a diff viewer.

## 0.4.0 (2026-10-19)

- The clients keep their clocks in sync (systemd-timesyncd);
  `netstation_ntp_servers` picks the time servers.
- A dnsmasq watchdog restarts a hanging dnsmasq and reports it
  (`netstation_dnsmasq_watchdog`).
- Desktop shortcuts that open a web page (`netstation_desktop_shortcuts`).
- RDP pass-through for sound, USB sticks and printers (`netstation_rdp_sound`,
  `netstation_rdp_usb_drives`, `netstation_rdp_printers`).
- A power schedule (`netstation_power_wake`, `netstation_power_off`,
  `netstation_power_report`).
- Clients check in with the server every 2 minutes, so `netstation-image
  clients`, status and the Veyon list also see idle clients.
- The idle logout leaves sessions alone that nobody has used.
- The `clients` playbook lists the clients that are on: IP, MAC and name.
- `netstation-image diff` is colored and paged in a terminal, and `--dir`
  writes both versions to folders for a diff viewer.

## 0.3.0 (2026-10-19)

- A firewall on the server (`netstation_firewall_enabled`, on by default):
  nftables, only SSH, network boot, NFS, and Cockpit and WireGuard when on.
  NFS mountd gets a fixed port (`netstation_nfs_mountd_port`) and dnsmasq
  answers TFTP from port 69 only.

## 0.2.0 (2026-10-19)

- Pushover alerts (`alerts` role, on when `netstation_pushover_app_token` and
  `netstation_pushover_user_key` are set): failed builds and services, a
  hanging dnsmasq, a full disk, failed security updates, a reboot needed for a
  new kernel, and a staging build ready to test.
- Automatic security updates on the server
  (`netstation_server_security_updates`, on by default; no automatic reboots).
- `netstation-image diff` and the `image_diff` playbook show what changed
  between production and staging; `image_build` shows it at the end.
- The idle logout now logs out at `netstation_idle_logout_minutes`, with the
  warning `netstation_idle_logout_warning_seconds` before it.

## 0.1.0 (2026-10-19)

First release.

- Server role: LTSP, dnsmasq in proxy or full DHCP mode, NFS and HTTP boot.
  Optional NAT (`netstation_nat_enabled`) for full-DHCP setups where the
  server connects the clients to the internet.
- `netstation_dnsmasq_version` and `netstation_dnsmasq_suite` pin dnsmasq,
  also from another Debian release; `netstation-dnsmasq-check` tells whether
  dnsmasq hangs.
- Fresh client image builds with staging, promote and rollback; scheduled
  weekly staging builds.
- An unattended Debian installer ISO for the server (`installer_iso`).
- Clients: portal in Firefox, `.rdp` files in FreeRDP, a fresh home folder
  for every session and automatic login again after a logout, an optional
  idle logout (`netstation_idle_logout_minutes`),
  the client's name and address in the panel, shut down and restart from the
  desktop, Firefox in its own language, no saved logins or form data.
- Wake-on-LAN (`wake`) and a Veyon computer list (`veyon_computers`).
- Optional WireGuard and Cockpit.

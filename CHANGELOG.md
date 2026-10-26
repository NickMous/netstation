# Changelog

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

# Changelog

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

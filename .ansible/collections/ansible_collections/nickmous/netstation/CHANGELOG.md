# Changelog

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
  for every session, an optional idle logout (`netstation_idle_logout_minutes`),
  the client's name and address in the panel, shut down and restart from the
  desktop, Firefox in its own language, no saved logins or form data.
- Wake-on-LAN (`wake`) and a Veyon computer list (`veyon_computers`).
- Optional WireGuard and Cockpit.

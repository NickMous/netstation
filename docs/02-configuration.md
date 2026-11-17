# Configuration

All settings are Ansible variables with the `netstation_` prefix. Set them in
your configuration repository, usually in
`inventory/group_vars/netstation_servers/main.yml`. Keep passwords and keys in
`vault.yml` and point to them from `main.yml`:

```yaml
# vault.yml (encrypted)
vault_admin_password: "..."
# main.yml
netstation_installer_admin_password: "{{ vault_admin_password }}"
```

Every variable is documented, with its default, in the collection's
`roles/*/defaults/main.yml`. This page covers the ones you're most likely to
change.

| Defaults file | What's in it |
|---|---|
| `roles/common/defaults/main.yml` | Image names, session user, Debian release, locale, Veyon, power schedule, Remote Mac on/off |
| `roles/server/defaults/main.yml` | DHCP mode, NAT, firewall, dnsmasq, known clients and test clients, extra `ltsp.conf` settings |
| `roles/image/defaults/main.yml` | Homepage, wallpaper, desktop shortcuts, idle logout, RDP options and pass-through, time servers, Firefox, packages, extra files, scheduled builds, RustDesk client |
| `roles/installer/defaults/main.yml` | Admin account and network settings of the installed server |
| `roles/alerts/defaults/main.yml` | Pushover keys and what to alert on |
| `roles/rustdesk_server/defaults/main.yml` | RustDesk server, the Mac handout and office mode |
| `roles/wireguard/defaults/main.yml` | The WireGuard VPN |
| `roles/cockpit/defaults/main.yml` | Extra Cockpit pages |

## Applying changes

| What | Where | Then run |
|---|---|---|
| Homepage, wallpaper, boot splash, desktop shortcuts, RDP options and pass-through, idle logout, packages, Firefox, power-off schedule | `group_vars` | `image_build`, test, `image_promote` |
| Files in the image | `files/rootfs/...` | `image_build`, test, `image_promote` |
| Known clients, test client, per-client settings | `netstation_clients` | `server` |
| DHCP, firewall, WireGuard peers, wake schedule, alerts, other server settings | `group_vars` | `server` |
| Remote Mac on or off | `netstation_remote_mac_enabled` | `server`, then `image_build`, test, `image_promote` |

Always preview server changes first:

```bash
ansible-playbook nickmous.netstation.server --check --diff
```

## The desktop

```yaml
netstation_homepage: https://portal.example.com     # opens in Firefox at login
netstation_wallpaper: "{{ inventory_dir }}/../files/wallpaper.jpg"
netstation_boot_splash_image: "{{ inventory_dir }}/../files/splash.png"
netstation_panel_show_address: true                 # name and IP next to the clock

netstation_desktop_shortcuts:
  - name: HD Cloud
    url: https://www.hdcloud.nl/
    icon: firefox-esr            # optional: an icon name or an icon file path
```

**Idle logout.** Log the session out after a while without keyboard or mouse
input, so the next person gets a fresh desktop. An RDP session only
disconnects and can be resumed. An open [Remote Mac](07-remote-mac.md)
session counts as input, so it isn't logged out.

```yaml
netstation_idle_logout_minutes: 30            # 0 = off (the default)
netstation_idle_logout_warning_seconds: 60
```

**Screens.** All connected screens are put side by side
(`netstation_displays_side_by_side`), also one switched on after the client
started. Turning this off lets XFCE mirror such a screen, which xfreerdp3
refuses (exit code 136).

## Language and region

```yaml
netstation_timezone: Europe/Amsterdam
netstation_locale: nl_NL.UTF-8
netstation_keyboard_layout: us
netstation_firefox_language: nl       # empty keeps Firefox in English
netstation_ntp_servers: [dc01.example.local]   # empty uses Debian's pool
```

The texts the clients show are settings too, so you can translate them:
`netstation_idle_logout_message`, `netstation_rdp_messages` and
`netstation_remote_mac_messages`. When you change one text of a group, set
all of them.

## Remote Desktop

Clients open downloaded `.rdp` files in xfreerdp3.

```yaml
netstation_rdp_options:       # extra xfreerdp3 options
  - /cert:tofu
netstation_rdp_sound: false        # the session's audio plays on the client
netstation_rdp_usb_drives: false   # USB sticks show up as drives
netstation_rdp_printers: false     # printers the client finds
netstation_audio_volume: 100       # volume at login, in percent
```

## Firefox

`netstation_firefox_policies` is merged over netstation's defaults. It takes
[Firefox enterprise policies](https://mozilla.github.io/policy-templates/):

```yaml
netstation_firefox_policies:
  DisableDeveloperTools: true
  Bookmarks:
    - Title: Intranet
      URL: https://intranet.example.com
      Placement: toolbar
```

## Packages and files in the image

```yaml
netstation_image_extra_packages: [vlc]
netstation_image_files_dir: "{{ inventory_dir }}/../files/rootfs"
```

Everything in `files/rootfs/` is copied over the image root last, so
`files/rootfs/etc/foo.conf` ends up as `/etc/foo.conf` on every client.

The image is Debian `netstation_debian_suite` (default `trixie`), without
recommended packages to keep it small. The base package lists
(`netstation_image_packages_*`) are in `roles/image/defaults/main.yml`.

## Network

```yaml
netstation_dhcp_mode: proxy          # or full, see Getting started
netstation_firewall_extra_rules:     # nftables rules for incoming traffic
  - tcp dport 8080 accept
```

The firewall only lets in SSH, network boot, NFS, and Cockpit, WireGuard and
RustDesk when they're on.

dnsmasq 2.91 to 2.93 (Debian 13 has 2.91) hang in proxy mode when the proxy
`dhcp-range` has a tag. netstation's configuration leaves the tag out, so
these versions work and no pin is needed. If a dnsmasq version gives trouble
anyway, pin one that works, for example the one from Debian 12:

```yaml
netstation_dnsmasq_version: "2.90-4~deb12u2"
netstation_dnsmasq_suite: bookworm
```

To go back to Debian's own version, remove both lines (or change to another
pinned version) and run `server`.

## Clients

Known clients, test clients, per-client `ltsp.conf` settings, Wake-on-LAN and
the power schedule are on [Managing clients](04-clients.md). Settings for all
clients:

```yaml
netstation_kernel_parameters: [nomodeset]
netstation_ltsp_clients_settings:     # extra [clients] lines in ltsp.conf
  KEY: value
netstation_initrd_gpu_drivers: [i915, amdgpu]   # smaller initrd; empty keeps all
```

## Server

```yaml
netstation_server_security_updates: true    # automatic, no automatic reboots
netstation_server_headless: true
netstation_releases_keep: 3                 # production images kept for rollback
```

The server manages `/etc/ltsp/ltsp.conf` and
`/etc/dnsmasq.d/ltsp-dnsmasq.conf`. Don't run `ltsp dnsmasq` by hand; it would
overwrite the HTTP-boot setup.

The session user (`netstation_session_user`, default `kiosk`) exists only on
the server, with an empty password. LTSP merges it into each client, where
its home folder lives in RAM.

## Optional features

| Feature | Turn on with | Page |
|---|---|---|
| Alerts | `netstation_pushover_app_token`, `netstation_pushover_user_key` | [Status and alerts](05-monitoring.md#alerts) |
| WireGuard | `netstation_wireguard_enabled: true` | [Remote management](06-remote-management.md#wireguard) |
| Cockpit | on by default; `netstation_cockpit_enabled: false` | [Remote management](06-remote-management.md#cockpit) |
| Veyon | `netstation_veyon_enabled: true` | [Remote management](06-remote-management.md#veyon) |
| Power schedule | `netstation_power_wake`, `netstation_power_off` | [Managing clients](04-clients.md#power-schedule) |
| Remote Mac | `netstation_remote_mac_enabled: true` | [Remote Mac](07-remote-mac.md) |

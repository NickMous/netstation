# Commands

## Playbooks

Run them from your configuration repository. Add `--ask-vault-pass` if you
haven't set up a vault password file, and `--check --diff` to preview
changes.

| Playbook | What it does |
|---|---|
| `nickmous.netstation.installer_iso` | Build an unattended Debian installer ISO for the server |
| `nickmous.netstation.server` | Install or repair the server (LTSP, dnsmasq, NFS, nginx, image tooling, security updates, alerts, WireGuard, Cockpit, RustDesk) |
| `nickmous.netstation.server --tags clients` | Only apply changes to `netstation_clients` |
| `nickmous.netstation.image_build` | Build a new staging image from scratch |
| `nickmous.netstation.image_diff` | Show what changed between production and staging |
| `nickmous.netstation.image_promote` | Make the staging image the production image |
| `nickmous.netstation.image_rollback` | Restore a previous production image |
| `nickmous.netstation.status` | Images, booted clients, services |
| `nickmous.netstation.clients` | The clients that are on: IP, MAC and name |
| `nickmous.netstation.wake` | Wake clients with Wake-on-LAN |
| `nickmous.netstation.veyon_computers` | Write the booted clients to a CSV file for Veyon Master |
| `nickmous.netstation.mac_handout` | Write the setup script and manual for Remote Mac users |

## On the server

| Command | What it does |
|---|---|
| `sudo netstation-image build` | Build a fresh staging image from the settings of the last playbook run (use the `image_build` playbook after changing settings) |
| `sudo netstation-image diff` | What changed between production and staging (`--summary`, or `--dir DIR` for a diff viewer) |
| `sudo netstation-image promote` | Make staging the production image |
| `sudo netstation-image rollback [BUILD]` | Restore a previous production image (default: the newest) |
| `sudo netstation-image status` | Images, booted clients and services |
| `sudo netstation-image clients` | Booted clients as `IP MAC`, one per line |
| `sudo netstation-health` | Run the health check by hand |
| `sudo netstation-dnsmasq-check` | Check whether dnsmasq hangs |
| `sudo netstation-power wake` | Wake the clients now (also `wake-N`, `learn`, `report`) |

## Logs

| On | Command | What |
|---|---|---|
| Server | `sudo journalctl -u dnsmasq -f` | Network boot requests |
| Server | `sudo journalctl -u netstation-staging-build.service` | The scheduled build |
| Server | `sudo journalctl -t netstation-alert` | Alerts sent |
| Server | `sudo journalctl -t netstation-client-report` | Problems reported by clients |
| Server | `sudo journalctl -u netstation-dnsmasq-watchdog` | dnsmasq watchdog |
| Server | `sudo journalctl -k --grep netstation-firewall` | Dropped by the firewall |
| Client | `sudo journalctl -t netstation-rdp` | Remote Desktop output |
| Client | `journalctl -t netstation-mac` | Remote Mac sessions |

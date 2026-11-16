# Status and alerts

## Status

```bash
ansible-playbook nickmous.netstation.status
```

It shows the production and staging builds, the rollback targets, the
scheduled build, which clients are booted right now, and whether dnsmasq, NFS
and nginx are running. On the server, `sudo netstation-image status` shows the
same.

Only the clients that are on, with IP and MAC address:

```bash
ansible-playbook nickmous.netstation.clients
```

For logs, services, updates and a terminal in the browser, use
[Cockpit](06-remote-management.md#cockpit).

## Alerts

netstation sends alerts to your phone with [Pushover](https://pushover.net).
Create an application there for the API token; your user key is on your
Pushover dashboard. Keep both in your vault:

```yaml
netstation_pushover_app_token: "{{ vault_pushover_app_token }}"
netstation_pushover_user_key: "{{ vault_pushover_user_key }}"
```

Alerts are on when both are set. Run the `server` playbook to apply.

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

What to alert on is configurable:

```yaml
netstation_alerts_name: school-west          # start of every alert title
netstation_alerts_services: [dnsmasq, nfs-server, nginx]
netstation_alerts_disk_percent: 85
netstation_alerts_check_interval: 10min
netstation_alerts_staging_ready: true
netstation_alerts_client_reports: true
```

To see the alerts the server sent, also when Pushover couldn't be reached:

```bash
sudo journalctl -t netstation-alert
```

Run a check by hand with `sudo netstation-health`.

## Server reboots

Security updates install by themselves, but the server never reboots by
itself: clients stop working while it restarts. Reboot it outside opening
hours when the reboot alert comes:

```bash
ssh <server> sudo systemctl reboot
```

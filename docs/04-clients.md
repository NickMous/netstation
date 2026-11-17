# Managing clients

## Adding a client

PXE boot must be enabled in the BIOS/UEFI and Secure Boot disabled. A new
client then works without any configuration.

Add it to `netstation_clients` only to give it a fixed name, label it, make
it a test client, give it its own settings or its own wake times:

```yaml
netstation_clients:
  - mac: "d8:9e:f3:37:53:a1"
    hostname: reception01       # optional
    label: Anna de Vries        # optional: who or what it's for, shown in alerts
    test: true                  # optional: boots staging, shows the boot menu
    settings: {}                # optional: extra ltsp.conf lines for this client
    wake: ["Mon..Fri 07:30"]    # optional: its own wake times
```

Then run the `server` playbook. With `--tags clients` it only applies the
changes to `netstation_clients` (`ltsp.conf`, the client names and the wake
times), which is quicker:

```bash
ansible-playbook nickmous.netstation.server --tags clients
```

The client picks it up at its next boot.

To find a client's MAC address, switch it on and run:

```bash
ansible-playbook nickmous.netstation.clients
```

## Test clients

A client with `test: true` boots the **staging** image by default, and shows
a boot menu for `netstation_test_menu_timeout` milliseconds (default 5000) to
pick production instead. Use it to try a new image before you
[promote](03-updating-clients.md) it. You can have more than one.

## Per-client settings

`settings` adds lines to that client's section in `ltsp.conf`. Useful ones
while troubleshooting:

```yaml
netstation_clients:
  - mac: "d8:9e:f3:37:53:a1"
    settings:
      DEBUG_SHELL: 1                 # a shell when the boot stops with an error
      KERNEL_PARAMETERS: nosplash    # show boot messages instead of the splash
```

## Waking clients

Clients listed in `netstation_clients` can be switched on from the server:

```bash
ansible-playbook nickmous.netstation.wake                                    # all of them
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

"Input" is the client's own keyboard and mouse, and an open
[Remote Mac](07-remote-mac.md) session: that keeps the client on, also when
someone works on the Mac with its own keyboard and mouse. A client with a Mac
session left open stays on until someone closes the RustDesk windows.

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
shows when each fires next.

Switching off is part of the image, so after changing `netstation_power_off`
run `image_build`, test, and `image_promote`. `netstation_power_wake` and
`netstation_power_report` only need `server`.

Try it by hand on the server:

```bash
sudo netstation-power wake      # or: wake-1 (a group), learn, report
```

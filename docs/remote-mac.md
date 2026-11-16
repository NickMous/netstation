# Remote Mac

With Remote Mac, people control their own Mac from any client: the client's
keyboard, mouse and screens work the Mac, as with Remote Desktop.

It uses [RustDesk](https://rustdesk.com). The server runs a RustDesk ID and
relay server (`rustdesk-hbbs`, `rustdesk-hbbr`), reachable only from
`netstation_remote_mac_networks`, and the image has the RustDesk client and a
desktop icon (`netstation_remote_mac_icon_name`, default "Mac").

## Using it

At the client: click the icon, type the Mac's RustDesk ID, then the password
RustDesk on the Mac shows.

- Each Mac screen opens full screen on its own client screen.
- The XFCE panels are gone during the session.
- Alt and the Windows key swap places, so the key next to the space bar is
  Cmd (`netstation_remote_mac_swap_alt_cmd`).
- The Mac's sound plays on the client (`netstation_remote_mac_sound`).

Close the RustDesk windows (the toolbar at the top of the screen) to get the
desktop back.

## Turning it on

1. Set `netstation_remote_mac_enabled: true` and run the `server` playbook.
   It prints the ID server and key for the Macs.
2. Build, test and promote an image.
3. The server makes a key on its first run. **Save it in the vault right
   away**, or a reinstalled server gets a new one and every Mac has to be set
   up again:

   ```bash
   ssh netadmin@<server> sudo cat /var/lib/rustdesk-server/id_ed25519
   ```

   ```yaml
   # vault.yml
   vault_rustdesk_private_key: "..."
   # main.yml
   netstation_rustdesk_private_key: "{{ vault_rustdesk_private_key }}"
   ```

## Setting up the Macs

Make the handout for the Mac users:

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
- adds an **office mode** app;
- shows the Mac's RustDesk ID.

Running the script again updates the apps and the settings. Make and hand out
a new handout after changing a version or the server's key.

### Office mode

Office mode (`netstation_mac_office_mode_name`) is an app on the Mac. Clicked
once, it adds virtual screens and keeps the Mac awake; clicked again, it
removes them. RustDesk shows as many Mac screens as the client has, starting
with the Mac's main screen, usually its own. It needs macOS 14 or newer.

```yaml
netstation_mac_office_displays:
  - { name: Desk left, width: 1920, height: 1080 }
  - { name: Desk right, width: 1920, height: 1080 }
netstation_mac_office_builtin_off: false   # true: only the virtual screens show
```

### Translating the handout

The manual is English. `netstation_mac_handout_manual_template` points at
your own (e.g. translated) template, and `netstation_mac_handout_messages`
holds what the script and office mode say. The dialogs on the clients are in
`netstation_remote_mac_messages`.

## Updating RustDesk

The versions are pinned. Change them deliberately, with the checksums from
the GitHub release pages:

| Where | Variables | Then |
|---|---|---|
| Clients | `netstation_rustdesk_client_version`, `netstation_rustdesk_client_checksum` | `image_build`, test, `image_promote` |
| Server | `netstation_rustdesk_server_version`, `netstation_rustdesk_server_checksums` | `server` |
| Macs | `netstation_mac_rustdesk_version`, `netstation_mac_rustdesk_checksums` | `mac_handout`, hand it out again |

Something not working? See
[Remote Mac doesn't work](troubleshooting.md#remote-mac-doesnt-work).

# Updating the clients

Clients never change by themselves. Every change, from a Debian security
update to a new homepage, goes into a new image that you build, test and then
promote.

Run the `ansible-playbook` commands from your configuration repository. Add
`--ask-vault-pass` if you haven't set up a vault password file.

## The routine

1. **Build** a staging image. This also happens automatically every week (see
   [Scheduled builds](#scheduled-builds)).

   ```bash
   ansible-playbook nickmous.netstation.image_build
   ```

2. **Check** what changed compared with production. `image_build` shows this
   at the end; after a scheduled build, run:

   ```bash
   ansible-playbook nickmous.netstation.image_diff
   ```

   It lists updated, added and removed packages, and the changes to the files
   netstation puts in the image (settings, Firefox policies, scripts). It also
   copies both versions to `build/image-diff/` in your configuration
   repository, to open in a diff viewer you install yourself:

   ```bash
   git diff --no-index build/image-diff/production build/image-diff/staging
   meld build/image-diff/production build/image-diff/staging
   ```

   On the server, `sudo netstation-image diff` shows the same, colored and
   paged like `git diff`.

3. **Test** it: reboot the test client. It boots staging by default; its boot
   menu also lets you pick production. (No test client yet? See
   [Test clients](clients.md#test-clients).)

4. **Promote** it. Clients switch to it on their next reboot.

   ```bash
   ansible-playbook nickmous.netstation.image_promote
   ```

## Rolling back

Something wrong after all? Roll back to the previous production image:

```bash
ansible-playbook nickmous.netstation.image_rollback
# or pick one from the status list:
ansible-playbook nickmous.netstation.image_rollback -e netstation_rollback_build=20261015T0300Z
```

Rolling back archives the current production image, so running it twice
brings you back to where you started. The server keeps the last
`netstation_releases_keep` (default 3) production images.

## Scheduled builds

The server builds a fresh staging image every Sunday at 03:00, to pick up
Debian security updates. Promoting it stays a manual decision. With
[alerts](monitoring.md#alerts) on, you get a message when it's ready, with
what changed.

```yaml
netstation_staging_build_enabled: true
netstation_staging_build_schedule: "Sun *-*-* 03:00:00"   # systemd OnCalendar
```

A failed build never touches production. See
[The scheduled staging build failed](troubleshooting.md#the-scheduled-staging-build-failed).

## Which changes need a new image

Settings that end up on the clients need a new image; settings for the
server only need the `server` playbook. [Configuration](configuration.md#applying-changes)
has the full table.

# Getting started

This page takes you from an empty machine to the first client booting from
the network.

## What you need

- **A machine to run Ansible from.** The commands below are for Debian or
  Ubuntu.
- **A server or VM for netstation.** Its first disk is erased during
  installation. It must be on the same network segment as the clients.
- **Clients that can network-boot.** PXE boot must be enabled in their
  BIOS/UEFI and Secure Boot disabled.

### DHCP: proxy or full

netstation needs to tell the clients where to boot from. How depends on who
hands out IP addresses on the client network (`netstation_dhcp_mode`):

- **`proxy`** (the default): a router or Windows server already hands out IP
  addresses. netstation's dnsmasq only adds the network-boot information. Make
  sure that DHCP server doesn't set options 60, 66 or 67 itself.
- **`full`**: netstation is the DHCP server for the client network. Set
  `netstation_dhcp_range_start`, `netstation_dhcp_range_end` and
  `netstation_dhcp_dns_servers`. With two network cards (one to the clients,
  one to the internet), `netstation_nat_enabled: true` shares the server's
  internet connection with the clients.

## 1. Set up your configuration repository

Install the tools and start a private configuration repository from the
collection's `examples/` folder:

```bash
sudo apt install ansible git xorriso
git clone https://github.com/NickMous/netstation.git
cp -r netstation/examples ~/my-netstation && cd ~/my-netstation
git init
```

Set the collection version you want in `requirements.yml` (a release tag such
as `v0.9.2`), then install it:

```bash
ansible-galaxy collection install -r requirements.yml
```

The repository looks like this:

```
my-netstation/
├── ansible.cfg
├── requirements.yml
├── inventory/
│   ├── hosts.yml                          # the server
│   └── group_vars/netstation_servers/
│       ├── main.yml                       # your settings
│       └── vault.yml                      # passwords and keys (encrypted)
└── files/                                 # wallpaper, Veyon keys, extra image files
```

## 2. Settings

Edit `inventory/hosts.yml` (the server's name, address and admin user) and
`inventory/group_vars/netstation_servers/main.yml`. The example `main.yml`
lists the settings you usually change; [Configuration](02-configuration.md)
covers the rest.

Then create the vault with the admin password:

```bash
cp inventory/group_vars/netstation_servers/vault.yml.example inventory/group_vars/netstation_servers/vault.yml
nano inventory/group_vars/netstation_servers/vault.yml
ansible-vault encrypt inventory/group_vars/netstation_servers/vault.yml
```

Commit everything except the vault password. To avoid typing it each time,
put it in `.vault_pass` (never commit it) and uncomment
`vault_password_file` in `ansible.cfg`. Otherwise, add `--ask-vault-pass` to
every `ansible-playbook` command in these docs.

## 3. Build the installer

Build an ISO that installs Debian without any questions, with the admin
account, your SSH key and the network settings from step 2:

```bash
ansible-playbook nickmous.netstation.installer_iso --ask-vault-pass
```

The ISO ends up in `build/<server name>.iso`.

## 4. Install the server

Boot the server or VM from the ISO. It erases the first disk, installs Debian
and reboots, which takes about ten minutes. Remove the ISO afterwards.

For a Hyper-V VM: use Generation 2, with Secure Boot on the "Microsoft UEFI
Certificate Authority" template (or off).

## 5. Set up netstation and build the first image

```bash
ansible-playbook nickmous.netstation.server --ask-vault-pass
ansible-playbook nickmous.netstation.image_build --ask-vault-pass
ansible-playbook nickmous.netstation.image_promote --ask-vault-pass
```

`server` sets up LTSP, dnsmasq, NFS, nginx, the firewall and everything else
on the server. `image_build` builds the first client image as staging, and
`image_promote` makes it production.

## 6. Boot a client

Boot a client from the network. It gets the LTSP image, logs in
automatically and opens `netstation_homepage` in Firefox.

## Next steps

- Mark one client as a **test client**, so new images can be tested before
  everyone gets them: see [Managing clients](04-clients.md).
- Learn the [update routine](03-updating-clients.md).
- Set up [alerts](05-monitoring.md#alerts) on your phone.

# Remote management

## Cockpit

Cockpit is a web console for the server: logs, services, updates and a
terminal. It's on by default, at `https://<server>:9090`. Log in with your
server account.

```yaml
netstation_cockpit_enabled: true
netstation_cockpit_extra_packages: [cockpit-storaged]   # extra pages
```

## WireGuard

A WireGuard VPN lets you manage the server, and reach the clients, from
outside the network.

1. Make a key pair for the server, and one for each of your devices:

   ```bash
   wg genkey | tee server.key | wg pubkey > server.pub
   ```

2. Put the server's private key in your vault and set up the peers:

   ```yaml
   netstation_wireguard_enabled: true
   netstation_wireguard_private_key: "{{ vault_wireguard_private_key }}"
   netstation_wireguard_peers:
     - name: admin-laptop
       public_key: "..."
       address: 10.99.0.2/32
   ```

3. Run the `server` playbook.

4. On your device, use the server's public key, `<public address>:51820` as
   the endpoint, and the LAN (and `10.99.0.0/24`) as allowed IPs. Forward UDP
   port 51820 on your router to the server.

The VPN is `10.99.0.0/24` with the server at `10.99.0.1`
(`netstation_wireguard_address`, `netstation_wireguard_network`). With
`netstation_wireguard_masquerade` (on by default), traffic from the VPN to
the LAN looks like it comes from the server.

## SSH into a client

Every client accepts SSH logins with your server account:

```bash
ssh <your-server-account>@<client-ip>
```

Clients get new SSH host keys on every boot, so expect a host key warning.
`ansible-playbook nickmous.netstation.clients` lists the IP addresses of the
clients that are on.

## Veyon

[Veyon](https://veyon.io) lets you watch and control the clients from Veyon
Master, as in a classroom.

```yaml
netstation_veyon_enabled: true
netstation_veyon_public_keys:
  - name: admin
    key: "{{ lookup('file', inventory_dir ~ '/../files/veyon/admin-public.pem') }}"
```

Export the public key from Veyon Configurator on the Veyon Master computer.
Changing these settings needs a new image: `image_build`, test,
`image_promote`.

Veyon Master needs a list of the clients. Generate it from the clients that
are booted right now, then import it with the two commands the playbook
prints:

```bash
ansible-playbook nickmous.netstation.veyon_computers
```

Client IP addresses can change, so run it again when Veyon shows clients as
offline that are switched on. DHCP reservations for the clients avoid that.

# specsnl.specsops.netbird_client

NetBird client for Ubuntu 26.04. Every NetBird peer runs it: hosts that route a private
network into NetBird as well as hosts that are only reachable through it. The role adds
the NetBird apt repository, installs the `netbird` package at a pinned version and turns
lazy connections off. With a setup key it enrols the peer; without one it only installs
the client, so the same role prepares an image and enrols the VMs built from it.

## Variables

| Variable                          | Default                                     | Description                                            |
|-----------------------------------|---------------------------------------------|--------------------------------------------------------|
| `netbird_client_version`          | —                                           | Package version to install and pin, e.g. 0.81.5        |
| `netbird_client_repo_key_url`     | `https://pkgs.netbird.io/debian/public.key` | Repository signing key URL                             |
| `netbird_client_management_url`   | `""`                                        | The NetBird server, e.g. `https://netbird.example.com` |
| `netbird_client_setup_key`        | —                                           | Setup key to enrol with; unset only installs           |
| `netbird_client_hostname`         | `inventory_hostname`                        | Peer name, and so its NetBird DNS name                 |
| `netbird_client_wireguard_port`   | `51820`                                     | UDP port the client listens on                         |
| `netbird_client_lazy_connections` | `false`                                     | Leave lazy connections to the account setting          |
| `netbird_client_manage_firewall`  | `true`                                      | Open the WireGuard port in ufw                         |

`netbird_client_version` is required, and `netbird_client_management_url` is required
with a setup key. The setup key is only used in a task that runs with `no_log`.

## The version pin

`/etc/apt/preferences.d/netbird` pins the package at `netbird_client_version` with
priority 1001, so neither `apt upgrade` nor unattended-upgrades moves the client ahead of
the server. Raise the version here after the server has been upgraded. A lower version
downgrades the client.

## Without a setup key

The package's post-install script starts the client, which writes a WireGuard private key
to `/var/lib/netbird/default.json`. Two VMs built from an image that kept it get the same
key and show up as one peer. Without a setup key the role therefore stops and disables
`netbird.service` and deletes `/var/lib/netbird`. The `cleanup` role deletes it as well,
as a second guard.

Do not drop the setup key from an enrolled host's inventory: the next run would delete
its state, and the peer would have to enrol again as a new one.

## With a setup key

The role enables and starts `netbird.service`, and runs

```bash
netbird up --management-url <url> --setup-key <key> --hostname <hostname> --wireguard-port <port>
```

only when `netbird status --json` reports the client as neither `Connected` nor
`Connecting`. A peer that is already registered logs in with its WireGuard key, so this
does not use up a one-off setup key. Since `up` does not run on a connected peer, a new
hostname or port only takes effect after `netbird down` on the host and another run.

## Lazy connections

With `netbird_client_lazy_connections: false` the role adds
`/etc/systemd/system/netbird.service.d/lazy-connections.conf`:

```ini
[Service]
Environment=NB_LAZY_CONN=off
```

The client's local setting wins over the account setting, so a toggle in the dashboard
cannot make this peer lazy. Set the variable to `true` to remove the drop-in and follow the
account setting. A change restarts an enrolled client, which briefly interrupts sessions
over the tunnel, including the one running the play when it goes through NetBird.

## Firewall

Like `wireguard`, the role does not depend on the `firewall` role but manages its own ufw
rule through `community.general.ufw`. With `netbird_client_manage_firewall` on it adds
`allow <netbird_client_wireguard_port>/udp`, so peers connect directly instead of through
a relay. The port must stay free: when it is taken, the client silently listens on a
random one, which this rule does not pass.

The rule carries the ufw comment `specsnl.specsops.netbird_client`, and a rule for an old
port is removed on the next run. Rules without the tag are left alone, and nothing is
removed when `netbird_client_manage_firewall` is turned off. When ufw is not installed the
rule is skipped with a warning. Run `specsnl.specsops.firewall` first.

### Traffic on `wt0`

The role adds no ufw rules for the tunnel interface, and none are needed. The client
inserts `iifname wt0 accept` rules into `INPUT` and `FORWARD` above ufw's chains and
enforces the account's access policies itself, so NetBird's policies are the firewall for
traffic that arrives through the tunnel. Checked with 0.81.5 next to ufw and netavark: the
rules survive `ufw reload`, rule changes, `ufw disable` and `ufw enable`, and Podman network
changes.

This also means ufw's rules do not restrict tunnel traffic. Limit who can reach a peer, and
on which ports, with NetBird policies.

## Containers

The role installs the client, writes the pin and the drop-in and handles the service in a
container too, but does not touch ufw.

## Example

```yaml
# Image build: install only.
- hosts: all
  become: true
  roles:
    - specsnl.specsops.firewall
    - role: specsnl.specsops.netbird_client
      vars:
        netbird_client_version: 0.81.5
    - specsnl.specsops.cleanup

# Each VM: enrol.
- hosts: app
  become: true
  roles:
    - role: specsnl.specsops.netbird_client
      vars:
        netbird_client_version: 0.81.5
        netbird_client_management_url: https://netbird.example.com
        netbird_client_setup_key: "{{ vault_netbird_setup_key }}"
```

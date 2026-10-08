# specsnl.specsops.wireguard

WireGuard admin hub for Ubuntu 26.04. Admin access (SSH, and `psql` from a laptop) goes
through a tunnel that ends on this host, which forwards it to the private network
behind it. The role installs `wireguard-tools`, templates `/etc/wireguard/<interface>.conf`
with the key and peers you pass in, and enables `wg-quick@<interface>`. Optionally it
turns on IPv4 forwarding, masquerades the tunnel towards the private networks, and opens
the port and route rules in ufw.

## Variables

| Variable                    | Default | Description                                  |
|-----------------------------|---------|----------------------------------------------|
| `wireguard_interface`       | `wg0`   | Interface name                               |
| `wireguard_address`         | —       | Hub's tunnel address, e.g. `10.21.0.1/24`    |
| `wireguard_listen_port`     | `51820` | UDP listen port                              |
| `wireguard_private_key`     | —       | Hub's private key (required, see below)      |
| `wireguard_peers`           | `[]`    | `name`, `public_key`, `allowed_ips` per peer |
| `wireguard_forward_cidrs`   | `[]`    | Networks behind the hub that peers may reach |
| `wireguard_masquerade`      | `true`  | Masquerade forwarded traffic as the hub      |
| `wireguard_manage_firewall` | `true`  | Open the port and route rules in ufw         |

`wireguard_address` and `wireguard_private_key` are required. Only IPv4 is supported.

## The hub's key

The private key is passed in, not generated on the host. A replacement VM must keep the
same identity, or every laptop config that points at the old public key stops working.
Generate the pair once and keep the private key in your secret store:

```bash
wg genkey | tee hub.key | wg pubkey > hub.pub
```

The config is written as `0600` root in a `0700` directory, and the tasks that handle it
run with `no_log`.

## Peers

Each peer gets a `[Peer]` section with its name as a comment. `allowed_ips` is the
peer's own tunnel address: the hub drops traffic from the peer with any other source.

```yaml
wireguard_peers:
  - name: alice-laptop
    public_key: WWfFdZVUeKCzhWJJvzYlS/FNpbuysv6MuJr2yQdNSaw=
    allowed_ips:
      - 10.21.0.2/32
```

When only the peers change, the config is applied with
`wg syncconf <interface> <(wg-quick strip <interface>)`, which adds, updates and removes
peers without taking the interface down, so the admin running the play over the tunnel
keeps their session. Any change in the `[Interface]` section (address, port, key, the
masquerade rules) restarts `wg-quick@<interface>` instead, since those only take effect
when the interface comes up.

Renaming the interface leaves the old one running. Stop and disable
`wg-quick@<old name>` by hand.

## Forwarding and masquerade

With `wireguard_forward_cidrs` set, the role enables `net.ipv4.ip_forward` in
`/etc/sysctl.d/99-wireguard.conf`. With `wireguard_masquerade` also on, it adds a
`PostUp`/`PostDown` pair per network:

```text
PostUp = iptables -t nat -A POSTROUTING -s 10.21.0.1/24 -d 10.20.0.0/24 -j MASQUERADE
PostDown = iptables -t nat -D POSTROUTING -s 10.21.0.1/24 -d 10.20.0.0/24 -j MASQUERADE
```

The targets then see the hub's address, so they need no route back to the tunnel network.
With `wireguard_masquerade: false` they see the peer's tunnel address and do need that
route.

Emptying `wireguard_forward_cidrs` removes the drop-in, but leaves the live
`net.ipv4.ip_forward` value alone until the next boot, as something else on the host may
rely on it.

## Firewall

Like `postgresql`, the role does not depend on the `firewall` role but manages its own
ufw rules through `community.general.ufw`. With `wireguard_manage_firewall` on it adds:

- `allow <wireguard_listen_port>/udp`;
- `route allow in on <wireguard_interface> to <cidr>` for each forward network, since ufw
  drops forwarded traffic by default.

The rules carry the ufw comment `specsnl.specsops.wireguard`. A rule for an old port or
for a network dropped from `wireguard_forward_cidrs` is removed on the next run. Rules
without the tag are left alone, and nothing is removed when `wireguard_manage_firewall`
is turned off. Give the forward networks as network addresses: ufw normalises
`10.20.0.17/24` to `10.20.0.0/24`, and the role would then remove and re-add the rule on
every run.

When ufw is not installed the rules are skipped with a warning. Run
`specsnl.specsops.firewall` first.

SSH to the hub itself arrives on the tunnel interface, not through a route rule. To close
public SSH, limit the `firewall` role's SSH rule to the tunnel, and delete the open rule
an earlier run added. Check that the tunnel works first, and run that play over it:

```yaml
firewall_rules:
  - rule: allow
    port: "22"
    proto: tcp
    interface_in: wg0
  - rule: allow
    port: "22"
    proto: tcp
    delete: true
```

## Laptop peer config

Each admin generates their own key pair, sends the public key to be added to
`wireguard_peers`, and keeps a config like this as `/etc/wireguard/specsops.conf`, or
imports it into the WireGuard app:

```ini
[Interface]
# The laptop's own private key: wg genkey
PrivateKey = <laptop private key>
# The peer's allowed_ips on the hub
Address = 10.21.0.2/32

[Peer]
# The hub: wg pubkey < hub.key
PublicKey = <hub public key>
Endpoint = <hub public IP>:51820
# The tunnel network plus every wireguard_forward_cidrs entry
AllowedIPs = 10.21.0.0/24, 10.20.0.0/24
PersistentKeepalive = 25
```

Then `wg-quick up specsops` and connect to the private addresses, e.g. `ssh 10.21.0.1`
for the hub and `psql -h 10.20.0.5` for a database behind it. Since the hub's key is
passed in, the same config keeps working after the hub VM is replaced.

## Containers

A container cannot create the interface. There the role writes the config and the sysctl
drop-in and enables the unit, but does not start it, apply the sysctl, reload the peers or
touch ufw.

## Example

```yaml
- hosts: app
  become: true
  roles:
    - specsnl.specsops.firewall
    - role: specsnl.specsops.wireguard
      vars:
        wireguard_address: 10.21.0.1/24
        wireguard_private_key: "{{ vault_wireguard_private_key }}"
        wireguard_peers:
          - name: alice-laptop
            public_key: WWfFdZVUeKCzhWJJvzYlS/FNpbuysv6MuJr2yQdNSaw=
            allowed_ips:
              - 10.21.0.2/32
        wireguard_forward_cidrs:
          - 10.20.0.0/24
```

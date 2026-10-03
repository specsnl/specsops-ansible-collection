# specsnl.specsops.firewall

ufw firewall setup: default deny incoming / allow outgoing, allow SSH, container-safe
enable. Uses `firewall_extra_rules` as a composition seam for other roles or playbooks
to add their own port rules without duplicating firewall logic.

## Variables

| Variable                    | Default                                   | Description                         |
|-----------------------------|-------------------------------------------|-------------------------------------|
| `firewall_default_incoming` | `deny`                                    | Default incoming policy             |
| `firewall_default_outgoing` | `allow`                                   | Default outgoing policy             |
| `firewall_rules`            | `[{rule: allow, port: "22", proto: tcp}]` | Baseline rules                      |
| `firewall_extra_rules`      | `[]`                                      | Additional rules (composition seam) |

## Rule keys

Each entry of `firewall_rules` and `firewall_extra_rules` is passed to
[`community.general.ufw`](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html).
Only `rule` is required; any other key left out is not passed on.

| Key             | Description                                                             |
|-----------------|-------------------------------------------------------------------------|
| `rule`          | `allow`, `deny`, `limit` or `reject`                                    |
| `direction`     | `in` or `out`; required with `interface`                                |
| `interface`     | Interface the rule applies to, in `direction`                           |
| `interface_in`  | Input interface; combine with `interface_out` for route rules           |
| `interface_out` | Output interface; combine with `interface_in` for route rules           |
| `from_ip`       | Source address or network (default `any`)                               |
| `from_port`     | Source port or range                                                    |
| `to_ip`         | Destination address or network (default `any`)                          |
| `to_port`       | Destination port or range                                               |
| `port`          | Alias of `to_port`; `to_port` wins when both are set                    |
| `proto`         | Protocol, e.g. `tcp`, `udp`, `any`                                      |
| `route`         | `true` to match routed/forwarded traffic instead of traffic to the host |
| `comment`       | Comment stored with the rule                                            |
| `delete`        | `true` to remove a previously applied rule matching the other keys      |

Removing an entry from the list does **not** remove its rule from ufw: the role only
adds rules. To close a rule that was applied earlier, keep the entry and set
`delete: true`.

## Example

```yaml
- hosts: all
  become: true
  vars:
    firewall_extra_rules:
      - rule: allow
        port: "443"
        proto: tcp
  roles:
    - specsnl.specsops.firewall
```

### Source-limited rule

PostgreSQL only from the app range, on the private interface:

```yaml
firewall_extra_rules:
  - rule: allow
    direction: in
    interface: eth1
    from_ip: 10.20.0.16/28
    to_port: "5432"
    proto: tcp
    comment: postgres from app range
```

### Interface rule

SSH only on the WireGuard interface:

```yaml
firewall_rules:
  - rule: allow
    direction: in
    interface: wg0
    port: "22"
    proto: tcp
```

### Route rule

Forward traffic from WireGuard to the private network:

```yaml
firewall_extra_rules:
  - rule: allow
    route: true
    interface_in: wg0
    interface_out: eth1
    to_ip: 10.20.0.0/24
```

Route rules only take effect when the host forwards packets (`net.ipv4.ip_forward`).

### Delete a rule

Close the bootstrap public-SSH rule after the interface rule above has replaced it:

```yaml
firewall_rules:
  - rule: allow
    port: "22"
    proto: tcp
    delete: true
  - rule: allow
    direction: in
    interface: wg0
    port: "22"
    proto: tcp
```

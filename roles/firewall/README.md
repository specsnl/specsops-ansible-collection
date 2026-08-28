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

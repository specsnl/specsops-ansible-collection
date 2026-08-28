# specsnl.specsops.hardening

OS hardening for Ubuntu 24.04: deploys an sshd configuration drop-in that disables
root login and password authentication, and installs/enables fail2ban.

## Variables

| Variable                                | Default                                    | Description                  |
|-----------------------------------------|--------------------------------------------|------------------------------|
| `hardening_ssh_dropin_path`             | `/etc/ssh/sshd_config.d/99-hardening.conf` | Path to sshd drop-in         |
| `hardening_ssh_permit_root_login`       | `"no"`                                     | PermitRootLogin value        |
| `hardening_ssh_password_authentication` | `"no"`                                     | PasswordAuthentication value |
| `hardening_ssh_max_auth_tries`          | `3`                                        | MaxAuthTries value           |
| `hardening_fail2ban_enabled`            | `true`                                     | Install and enable fail2ban  |

## Example

```yaml
- hosts: all
  become: true
  roles:
    - specsnl.specsops.hardening
```

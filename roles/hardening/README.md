# specsnl.specsops.hardening

OS hardening for Ubuntu 24.04: installs openssh-server, deploys an sshd configuration
drop-in that disables root login and password authentication, and installs/enables
fail2ban.

## sshd drop-in ordering

The drop-in is deliberately numbered `00-`. OpenSSH's `Include` uses the **first**
value it obtains for a keyword, and Ubuntu cloud images ship
`/etc/ssh/sshd_config.d/50-cloud-init.conf` containing `PasswordAuthentication yes`.
A higher-numbered drop-in is parsed afterwards and silently ignored. If you override
`hardening_ssh_dropin_path`, keep the prefix below `50`.

The role also removes a leftover `99-hardening.conf` from hosts provisioned before
this collection's 0.2.0 release.

## Variables

| Variable                                       | Default                                    | Description                        |
|------------------------------------------------|--------------------------------------------|------------------------------------|
| `hardening_ssh_dropin_path`                    | `/etc/ssh/sshd_config.d/00-hardening.conf` | Path to sshd drop-in               |
| `hardening_ssh_permit_root_login`              | `"no"`                                     | PermitRootLogin value              |
| `hardening_ssh_password_authentication`        | `"no"`                                     | PasswordAuthentication value       |
| `hardening_ssh_kbd_interactive_authentication` | `"no"`                                     | KbdInteractiveAuthentication value |
| `hardening_ssh_max_auth_tries`                 | `3`                                        | MaxAuthTries value                 |
| `hardening_fail2ban_enabled`                   | `true`                                     | Install and enable fail2ban        |
| `hardening_ssh_manage_package`                 | `true`                                     | Install openssh-server             |

## Example

```yaml
- hosts: all
  become: true
  roles:
    - specsnl.specsops.hardening
```

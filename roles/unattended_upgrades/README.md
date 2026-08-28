# specsnl.specsops.unattended_upgrades

Automatic security updates for Ubuntu 24.04: installs `unattended-upgrades` and
`chrony`, deploys `20auto-upgrades` and `50unattended-upgrades` apt config drop-ins.

## Variables

| Variable                                  | Default                                        | Description                         |
|-------------------------------------------|------------------------------------------------|-------------------------------------|
| `unattended_upgrades_packages`            | `[unattended-upgrades, chrony]`                | Packages to install                 |
| `unattended_upgrades_periodic_update`     | `"1"`                                          | APT::Periodic::Update-Package-Lists |
| `unattended_upgrades_periodic_unattended` | `"1"`                                          | APT::Periodic::Unattended-Upgrade   |
| `unattended_upgrades_allowed_origins`     | `["${distro_id}:${distro_codename}-security"]` | Allowed upgrade origins             |
| `unattended_upgrades_autofix_dpkg`        | `"true"`                                       | AutoFixInterruptedDpkg              |
| `unattended_upgrades_minimal_steps`       | `"true"`                                       | MinimalSteps                        |
| `unattended_upgrades_remove_unused`       | `"true"`                                       | Remove-Unused-Dependencies          |
| `chrony_enabled`                          | `true`                                         | Enable and start chrony             |

## Example

```yaml
- hosts: all
  become: true
  roles:
    - specsnl.specsops.unattended_upgrades
```

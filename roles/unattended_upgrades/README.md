# specsnl.specsops.unattended_upgrades

Automatic security updates for Ubuntu 24.04: installs `unattended-upgrades` and
`chrony`, deploys `20auto-upgrades` and `50unattended-upgrades` apt config drop-ins.

## Variables

| Variable                                  | Default                                        | Description                         |
|-------------------------------------------|------------------------------------------------|-------------------------------------|
| `unattended_upgrades_packages`            | `[unattended-upgrades, chrony]`                | Packages to install                 |
| `unattended_upgrades_periodic_update`     | `"1"`                                          | APT::Periodic::Update-Package-Lists |
| `unattended_upgrades_periodic_download`   | `"1"`                                          | APT::Periodic::Download-Upgradeable |
| `unattended_upgrades_periodic_unattended` | `"1"`                                          | APT::Periodic::Unattended-Upgrade   |
| `unattended_upgrades_autoclean_interval`  | `"7"`                                          | APT::Periodic::AutocleanInterval    |
| `unattended_upgrades_allowed_origins`     | security + ESM origins (see below)             | Allowed upgrade origins             |
| `unattended_upgrades_autofix_dpkg`        | `"true"`                                       | AutoFixInterruptedDpkg              |
| `unattended_upgrades_minimal_steps`       | `"true"`                                       | MinimalSteps                        |
| `unattended_upgrades_remove_unused`       | `"true"`                                       | Remove-Unused-Dependencies          |
| `chrony_enabled`                          | `true`                                         | Enable and start chrony             |

## Allowed origins

The role writes `50unattended-upgrades` wholesale rather than patching it, so
`unattended_upgrades_allowed_origins` has to carry everything the distribution
shipped. It defaults to:

```yaml
unattended_upgrades_allowed_origins:
  - "${distro_id}:${distro_codename}-security"
  - "${distro_id}ESMApps:${distro_codename}-apps-security"
  - "${distro_id}ESM:${distro_codename}-infra-security"
```

The two ESM entries are Ubuntu's Expanded Security Maintenance origins. Dropping them
silently stops ESM security updates on Ubuntu Pro hosts — the play still reports
success. Keep them when you override this variable; they never match on non-Ubuntu
and are inert there.

## Example

```yaml
- hosts: all
  become: true
  roles:
    - specsnl.specsops.unattended_upgrades
```

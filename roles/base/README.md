# specsnl.specsops.base

Base system setup for Ubuntu 24.04: apt update/upgrade, core package install, locale
generation, timezone configuration, and sysctl tuning.

## Unattended-run safety

Three settings are applied *before* the apt upgrade, because without them an upgrade
can hang or fail on a host nobody is watching:

- `needrestart` is switched to automatic mode. Ubuntu 22.04+ defaults it to interactive,
  which blocks apt on a whiptail service-restart prompt.
- `DPkg::Lock::Timeout` is set, so apt waits for `unattended-upgrades` to release the
  dpkg lock instead of failing outright — common on freshly booted cloud instances.
- `force-confdef,force-confold` is passed to dpkg, so a modified conffile does not
  trigger a prompt.

## Variables

| Variable                     | Default                                                                     | Description                        |
|------------------------------|-----------------------------------------------------------------------------|------------------------------------|
| `base_packages`              | `[curl, git, jq, gnupg, ca-certificates, locales, acl, unzip]`              | Packages to install                |
| `base_apt_upgrade`           | `dist`                                                                      | apt upgrade type                   |
| `base_apt_cache_valid_time`  | `3600`                                                                      | Cache validity (seconds)           |
| `base_apt_dpkg_options`      | `force-confdef,force-confold`                                               | dpkg options for the upgrade       |
| `base_apt_lock_timeout`      | `300`                                                                       | Seconds apt waits for a held lock  |
| `base_needrestart_automatic` | `true`                                                                      | Restart services without prompting |
| `base_prefer_ipv4`           | `false`                                                                     | Prefer IPv4 in `/etc/gai.conf`     |
| `base_locale`                | `en_US.UTF-8`                                                               | Locale to generate                 |
| `base_lang`                  | `en_US.UTF-8`                                                               | LANG value in /etc/default/locale  |
| `base_timezone`              | `UTC`                                                                       | System timezone                    |
| `base_sysctl_file`           | `/etc/sysctl.d/99-base.conf`                                                | sysctl drop-in path                |
| `base_sysctl`                | `{vm.swappiness: "10", vm.vfs_cache_pressure: "50", fs.file-max: "200000"}` | sysctl settings                    |

`acl` is not decorative — Ansible needs it to `become_user` an unprivileged account
without falling back to world-readable temp files.

If you also use the `swap` role, note that it sets `vm.swappiness` too, in its own
drop-in. Set it in one role or the other, not both.

## Example

```yaml
- hosts: all
  become: true
  roles:
    - specsnl.specsops.base
```

# specsnl.specsops.base

Base system setup for Ubuntu 24.04: apt update/upgrade, core package install, locale
generation, timezone configuration, and sysctl tuning.

## Variables

| Variable                    | Default                                            | Description                       |
|-----------------------------|----------------------------------------------------|-----------------------------------|
| `base_packages`             | `[curl, git, jq, gnupg, ca-certificates, locales]` | Packages to install               |
| `base_apt_upgrade`          | `dist`                                             | apt upgrade type                  |
| `base_apt_cache_valid_time` | `3600`                                             | Cache validity (seconds)          |
| `base_locale`               | `en_US.UTF-8`                                      | Locale to generate                |
| `base_lang`                 | `en_US.UTF-8`                                      | LANG value in /etc/default/locale |
| `base_timezone`             | `UTC`                                              | System timezone                   |
| `base_sysctl_file`          | `/etc/sysctl.d/99-base.conf`                       | sysctl drop-in path               |
| `base_sysctl`               | `{vm.swappiness: "10", fs.file-max: "200000"}`     | sysctl settings                   |

## Example

```yaml
- hosts: all
  become: true
  roles:
    - specsnl.specsops.base
```

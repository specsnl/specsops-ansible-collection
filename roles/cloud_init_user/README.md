# specsnl.specsops.cloud_init_user

Bakes a cloud-init `default_user` into an image, so the first boot of every VM cloned
from it creates a non-root sudo user and installs the metadata SSH keys on that user.
Root stays locked.

Without it, cloud-init installs the keys given at server creation on the image's
default user, which on UpCloud's Ubuntu 26.04 template is `root`, and `hardening` sets
`PermitRootLogin no`. The VM is then unreachable.

## How cloud-init picks this up

The stock `/etc/cloud/cloud.cfg` has `users: [default]`, so `system_info.default_user`
decides who is created. `cc_ssh` writes the metadata `public_keys` to that user, and
`disable_root: true` gives root a "log in as `specsops`" stub.

These modules run once per instance-id, so the image must end with `cloud-init clean`.

`ssh_import_id` is written at the top level, not under `default_user`: `cc_ssh_import_id`
reads the default user's import sources from the top level only. Each import runs once
per instance and needs HTTPS access to the key server at first boot.

## Drop-in ordering

cloud-init merges `/etc/cloud/cloud.cfg.d/*.cfg` in byte order, and the last file wins
on a conflicting key. UpCloud's Ubuntu 26.04 template ships these drop-ins:

| File             | Sets                                                               |
|------------------|--------------------------------------------------------------------|
| `05_logging.cfg` | logging                                                            |
| `90_dpkg.cfg`    | `datasource_list`                                                  |
| `91-upcloud.cfg` | `datasource_list`                                                  |
| `92-upcloud.cfg` | `default_user.name: root`, `disable_root: false`, the module lists |

The drop-in is numbered `99-zz-` to sort after `92-upcloud.cfg`, and after subiquity's
`99-installer.cfg`, which carries an installer's own users. `-` sorts before `_` and
before letters, so a `99_*.cfg` or a letter-named drop-in would still win. If you
override `cloud_init_user_dropin_path`, keep it last.

Keys merge one by one, so the drop-in sets every `default_user` key it relies on rather
than inheriting `groups` or `gecos` from `cloud.cfg`.

## Validation

The drop-in is validated with `cloud-init schema -c` when `/usr/bin/cloud-init` exists.
Docker build images may ship without cloud-init, and there the check is skipped. The
schema reports `system_info` as deprecated for user-data and vendor-data. In
`cloud.cfg.d` it is the intended place, and the notice does not fail the check.

## Variables

| Variable                         | Default                                                  | Description                                |
|----------------------------------|----------------------------------------------------------|--------------------------------------------|
| `cloud_init_user_name`           | `specsops`                                               | Default user created at first boot         |
| `cloud_init_user_gecos`          | `SpecsOps admin`                                         | Full-name (GECOS) field                    |
| `cloud_init_user_groups`         | `[adm, sudo]`                                            | Supplementary groups                       |
| `cloud_init_user_sudo`           | `"ALL=(ALL) NOPASSWD:ALL"`                               | Sudoers rule; the password is locked       |
| `cloud_init_user_shell`          | `/bin/bash`                                              | Login shell                                |
| `cloud_init_user_ssh_import_ids` | `[]`                                                     | ssh-import-id sources, e.g. `gh:<user>`    |
| `cloud_init_user_dropin_path`    | `/etc/cloud/cloud.cfg.d/99-zz-specsops-default-user.cfg` | Path to the drop-in; must sort after peers |

## Example

```yaml
# postgres/playbook.yml (Packer)
- hosts: all
  become: true
  vars:
    cloud_init_user_ssh_import_ids: [gh:octocat]
  roles:
    - specsnl.specsops.base
    - specsnl.specsops.hardening
    - specsnl.specsops.cloud_init_user   # the login user PermitRootLogin no leaves missing
    - specsnl.specsops.cleanup
```

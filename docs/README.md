# Role Index

| Role                | FQCN                                   | Description                                                                                                                                    |
|---------------------|----------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| base                | `specsnl.specsops.base`                | apt update/upgrade, core packages (curl, git, jq, gnupg, ca-certificates, locales, acl, unzip), locale-gen, timezone UTC, sysctl tuning        |
| hardening           | `specsnl.specsops.hardening`           | sshd drop-in (`00-hardening.conf`): PermitRootLogin no, PasswordAuthentication no, MaxAuthTries 3; fail2ban install + enable                   |
| firewall            | `specsnl.specsops.firewall`            | ufw: default deny incoming / allow outgoing, allow SSH 22, container-safe enable, `firewall_extra_rules` composition seam                      |
| unattended_upgrades | `specsnl.specsops.unattended_upgrades` | chrony + unattended-upgrades; `20auto-upgrades` and `50unattended-upgrades` apt config drop-ins                                                |
| postgresql          | `specsnl.specsops.postgresql`          | PGDG apt repo + GPG key, `postgresql-{{ postgresql_version }}` install, tuning.conf template, `pg_hba` entries, systemd enable, ufw allow port |
| swap                | `specsnl.specsops.swap`                | non-sparse swap file create/format, `/etc/fstab` persist, activate, `vm.swappiness` + `vm.vfs_cache_pressure` drop-in                          |
| logrotate           | `specsnl.specsops.logrotate`           | global `maxsize` + `compress` in `/etc/logrotate.conf`, plus per-file overrides for jobs that set their own                                    |
| cleanup             | `specsnl.specsops.cleanup`             | apt autoremove + clean, wipe `/tmp`, `/var/tmp`, `/var/lib/apt/lists` — build-time only                                                        |

Each role's own README documents its variables in full.

## Architecture

These roles are consumed by two repos:

```text
specsops-golden-images        ─requires→  specsnl.specsops  ←requires─  specsops-ansible
  Packer build-time                        (this collection)               ansible-pull runtime
```

The golden-image build installs software via these roles. The ansible-pull runtime
enforces the same roles on live servers. Because both use the same collection version,
the system that runs is the system that was tested.

## Container safety

Roles that touch systemd services, swap or the firewall detect container environments and
skip the steps that are not container-compatible (`ufw enable`, `systemctl start`,
`swapon`, `sysctl reload`). Detection stats `/.dockerenv` and `/run/.containerenv` on the
**target** (a lookup would probe the controller, which is a container itself here, and
mark every host as one). Each such role derives its own `<role>_in_container` fact and
only when it is not already defined, so a playbook or Molecule inventory can override it —
for example `base_in_container: true`. This lets the same roles run inside Docker/Podman
containers during Packer builds and Molecule tests without error.

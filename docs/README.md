# Role Index

| Role                | FQCN                                   | Description                                                                                                                  |
|---------------------|----------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| base                | `specsnl.specsops.base`                | apt update/upgrade, core packages (curl, git, jq, gnupg, ca-certificates, locales), locale-gen, timezone UTC, sysctl tuning  |
| hardening           | `specsnl.specsops.hardening`           | sshd drop-in (`99-hardening.conf`): PermitRootLogin no, PasswordAuthentication no, MaxAuthTries 3; fail2ban install + enable |
| firewall            | `specsnl.specsops.firewall`            | ufw: default deny incoming / allow outgoing, allow SSH 22, container-safe enable, `firewall_extra_rules` composition seam    |
| unattended_upgrades | `specsnl.specsops.unattended_upgrades` | chrony + unattended-upgrades; `20auto-upgrades` and `50unattended-upgrades` apt config drop-ins                              |
| postgresql          | `specsnl.specsops.postgresql`          | PGDG apt repo + GPG key, `postgresql-{{ postgresql_version }}` install, tuning.conf template, systemd enable, ufw allow port |
| cleanup             | `specsnl.specsops.cleanup`             | apt autoremove + clean, wipe `/tmp`, `/var/tmp`, `/var/lib/apt/lists` — build-time only                                      |

## Architecture

These roles are consumed by two repos:

```text
packer-golden-images          ─requires→  specsnl.specsops  ←requires─  specsops-ansible
  Packer build-time                        (this collection)               ansible-pull runtime
```

The golden-image build installs software via these roles. The ansible-pull runtime
enforces the same roles on live servers. Because both use the same collection version,
the system that runs is the system that was tested.

## Container safety

Roles that touch systemd services or the firewall detect container environments via the
`in_container` fact and skip steps that are not container-compatible (`ufw enable`,
`systemctl start`, `sysctl reload`). This allows the same roles to run inside Docker/Podman
containers during Packer builds and Molecule tests without error.

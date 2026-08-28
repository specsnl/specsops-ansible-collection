# SpecsOps Ansible Collection

Ansible collection `specsnl.specsops` — reusable roles for provisioning and hardening
Ubuntu 24.04 (Noble) hosts at Specs.

This collection is the single source of truth consumed by both:

- **[specsops-golden-images](https://github.com/specsnl/specsops-golden-images)** —
  Packer builds golden images using these roles at build time
- **[specsops-ansible](https://github.com/specsnl/specsops-ansible)** —
  `ansible-pull` enforces baseline policy at runtime using the same roles

Same roles, same version — no drift between the image that was built and the system
that is enforced.

## Roles

| Role                                   | Description                                                 |
|----------------------------------------|-------------------------------------------------------------|
| `specsnl.specsops.base`                | apt update/upgrade, core packages, locale, timezone, sysctl |
| `specsnl.specsops.hardening`           | sshd drop-in hardening + fail2ban                           |
| `specsnl.specsops.firewall`            | ufw baseline + parameterized extra rules                    |
| `specsnl.specsops.unattended_upgrades` | chrony + unattended-upgrades + apt config                   |
| `specsnl.specsops.postgresql`          | PGDG repo, PostgreSQL install, tuning, access, enable       |
| `specsnl.specsops.swap`                | swap file create/format/persist/activate + sysctl           |
| `specsnl.specsops.logrotate`           | global logrotate maxsize + compression                      |
| `specsnl.specsops.cleanup`             | apt autoremove/clean, wipe temp dirs (build-time)           |

## Requirements

- Ansible Core >= 2.16
- Ubuntu 24.04 LTS (Noble Numbat)
- Collections: `community.general`, `ansible.posix` (installed automatically via Galaxy)

## Installation

```yaml
# requirements.yml
collections:
  - name: git+https://github.com/specsnl/specsops-ansible-collection.git
    type: git
    version: main
```

```bash
ansible-galaxy collection install -r requirements.yml
```

## Usage

```yaml
- hosts: all
  become: true
  roles:
    - specsnl.specsops.base
    - specsnl.specsops.hardening
    - specsnl.specsops.firewall
    - specsnl.specsops.unattended_upgrades
    - specsnl.specsops.swap
    - specsnl.specsops.logrotate
```

If you use both `base` and `swap`, note that they each manage `vm.swappiness` in their
own `/etc/sysctl.d/` drop-in. Set it in one of them, not both.

## Development

```bash
# Lint
task ansible:lint

# Test a single role (full molecule create → converge → idempotence → verify → destroy)
task ansible:test:base

# Test all roles
task ansible:test

# Build collection tarball
task galaxy:build
```

### Local testing with Podman

The Molecule driver is `docker`. If you use Podman locally, expose the Podman socket
so the Docker driver can reach it:

```bash
podman system service --time=0 &
export DOCKER_HOST=unix://${XDG_RUNTIME_DIR}/podman/podman.sock
task ansible:test:base
```

## Release

1. Add a changelog fragment under `changelogs/fragments/`
2. Bump `version:` in `galaxy.yml`
3. Run `task changelog:release:<version>`
4. Commit, tag `v<version>`, push — CI publishes to Ansible Galaxy automatically

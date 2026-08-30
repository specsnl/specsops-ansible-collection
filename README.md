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

| Role                                   | Description                                                 | Docs                                                             |
|----------------------------------------|-------------------------------------------------------------|------------------------------------------------------------------|
| `specsnl.specsops.base`                | apt update/upgrade, core packages, locale, timezone, sysctl | [roles/base](roles/base/README.md)                               |
| `specsnl.specsops.hardening`           | sshd drop-in hardening + fail2ban                           | [roles/hardening](roles/hardening/README.md)                     |
| `specsnl.specsops.firewall`            | ufw baseline + parameterized extra rules                    | [roles/firewall](roles/firewall/README.md)                       |
| `specsnl.specsops.unattended_upgrades` | chrony + unattended-upgrades + apt config                   | [roles/unattended_upgrades](roles/unattended_upgrades/README.md) |
| `specsnl.specsops.postgresql`          | PGDG repo, PostgreSQL install, tuning, `pg_hba`, ufw port   | [roles/postgresql](roles/postgresql/README.md)                   |
| `specsnl.specsops.swap`                | swap file create/format/persist/activate + sysctl           | [roles/swap](roles/swap/README.md)                               |
| `specsnl.specsops.logrotate`           | global logrotate maxsize + compression                      | [roles/logrotate](roles/logrotate/README.md)                     |
| `specsnl.specsops.cleanup`             | apt autoremove/clean, wipe temp dirs (build-time)           | [roles/cleanup](roles/cleanup/README.md)                         |

Every role documents its variables in its own README; [docs/README.md](docs/README.md)
carries the condensed index and the notes on container safety.

All roles detect container environments and skip the steps that cannot work there
(`ufw enable`, `systemctl start`, `swapon`, `sysctl reload`), so the same roles run
unchanged in a Packer build container, in Molecule, and on a live VM.

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

Everything runs in containers via [Task](https://taskfile.dev) and Docker Compose —
you need Docker and `task` on your machine, but no local Ansible, Python or Molecule.
The `ansible` service (`ghcr.io/specsnl/ansible`) mounts the repo at `/workspace` and
the Docker socket, so Molecule can start test containers from inside it.

```bash
# Lint (yamllint + ansible-lint)
task ansible:lint
task ansible:lint:ansible:fix   # auto-fix what ansible-lint can

# Test a single role (destroy → syntax → create → converge → idempotence → verify → destroy)
task ansible:test:base

# Test all roles
task ansible:test

# Iterate on one role without tearing the container down
task ansible:converge:base
task ansible:verify:base
task ansible:destroy:base

# Markdown
task md:checkstyle
task md:fixstyle

# Interactive shell in the Ansible container
task shell

# Build / install the collection tarball into ./dist
task galaxy:build
task galaxy:install:local
```

`task --list` shows every task.

Molecule tests run against `geerlingguy/docker-ubuntu2404-ansible`, pinned by digest,
with systemd as PID 1.

### Local testing with Podman

The Molecule driver is `docker`, and `compose.yml` bind-mounts `/var/run/docker.sock`
into the `ansible` service. With Podman you need a Docker-API-compatible socket at that
path, so run Compose against the Podman socket and expose it:

```bash
podman system service --time=0 &
export DOCKER_HOST=unix://${XDG_RUNTIME_DIR}/podman/podman.sock
task ansible:test:base
```

If the socket lives elsewhere, override the bind mount in a `compose.override.yml`
rather than editing `compose.yml`.

## Release

1. Add a changelog fragment under `changelogs/fragments/`
2. Bump `version:` in `galaxy.yml`
3. Run `task changelog:release:<version>` — regenerates `CHANGELOG.rst` and consumes
   the fragments
4. Commit, tag `v<version>`, push

The tag workflow asserts the tag matches `version:` in `galaxy.yml`, then builds and
publishes to Ansible Galaxy using the `GALAXY_API_KEY` secret.

## CI

| Workflow                               | Trigger         | Runs                                               |
|----------------------------------------|-----------------|----------------------------------------------------|
| [pr.yml](.github/workflows/pr.yml)     | pull request    | Lint + Molecule, only for the roles the PR touches |
| [main.yml](.github/workflows/main.yml) | push to `main`  | Lint + Molecule for all roles                      |
| [md.yml](.github/workflows/md.yml)     | `**.md` changes | markdownlint                                       |
| [tag.yml](.github/workflows/tag.yml)   | `v*` tag        | Build + publish to Ansible Galaxy                  |

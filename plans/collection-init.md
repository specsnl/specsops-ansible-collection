# Collection Init — specsnl.specsops

## Goal

Stand up the `specsnl.specsops` Ansible Galaxy collection from its init skeleton:
scaffold all foundation files, convert the three Packer shell scripts into six
idempotent Ansible roles, add Molecule tests per role, lint/CI, and a tag-triggered
publish to public Ansible Galaxy.

This collection is the **single source of truth** for all Specs infrastructure roles,
consumed by both build-time (Packer in `specsops-golden-images`) and runtime
(ansible-pull in `specsops-ansible`). Same roles, same version — no drift.

## Context

The current golden images are provisioned by shell scripts (`base/install.sh`,
`base/cleanup.sh`, `postgres/install.sh`). Those scripts are the source of truth for
authoring the roles — keep them until the roles are verified, then delete them as part
of the companion plan in `specsops-golden-images/plans/postgres-ansible.md`.

## Collection Identity

| Field        | Value                                 |
|--------------|---------------------------------------|
| Namespace    | `specsnl`                             |
| Name         | `specsops`                            |
| FQCN pattern | `specsnl.specsops.<role>`             |
| Distribution | Public Ansible Galaxy (tag-triggered) |
| `galaxy.yml` | Repo root (one repo = one collection) |

## Role Mapping

Each shell script concern maps to one role so roles compose independently at both
Packer build time and ansible-pull runtime:

| Shell script          | Concern                                                     | Role                  |
|-----------------------|-------------------------------------------------------------|-----------------------|
| `base/install.sh`     | apt update/upgrade, core packages, locale, timezone, sysctl | `base`                |
| `base/install.sh`     | sshd drop-in 99-hardening.conf + fail2ban                   | `hardening`           |
| `base/install.sh`     | ufw baseline + container-safe enable + extra-rules seam     | `firewall`            |
| `base/install.sh`     | chrony + unattended-upgrades + apt config drop-ins          | `unattended_upgrades` |
| `postgres/install.sh` | PGDG repo+key, postgresql-18, tuning.conf, enable, ufw 5432 | `postgresql`          |
| `base/cleanup.sh`     | apt autoremove/clean, wipe temp dirs (build-only)           | `cleanup`             |

## Target Structure

```text
specsops-ansible-collection/
├── galaxy.yml                 # EDIT: description, license→[MIT], tags, deps, build_ignore
├── meta/runtime.yml           # EDIT: requires_ansible: ">=2.16.0"
├── README.md                  # EDIT: purpose, role index, usage, local Podman note
├── LICENSE                    # keep (MIT, Copyright 2026 Specs BV)
├── ansible.cfg                # NEW
├── .gitignore                 # NEW
├── .yamllint                  # NEW
├── .ansible-lint              # NEW
├── requirements.txt           # NEW (pinned Python dev deps)
├── requirements.yml           # NEW (community.general + ansible.posix)
├── Taskfile.dist.yml          # NEW (ansible:* / galaxy:* / changelog:* tasks)
├── changelogs/config.yaml     # NEW (antsibull-changelog) + fragments/.gitkeep
├── docs/README.md             # NEW (role index)
├── .github/
│   ├── dependabot.yml         # NEW (github-actions + pip, weekly)
│   └── workflows/
│       ├── pr.yml             # NEW (lint + molecule per-role, paths-filter gated)
│       ├── main.yml           # NEW (lint + all roles unconditionally)
│       └── tag.yml            # NEW (build + publish to Ansible Galaxy)
└── roles/<role>/
    ├── defaults/main.yml
    ├── tasks/main.yml
    ├── handlers/main.yml
    ├── templates/*.j2
    ├── meta/main.yml
    ├── meta/argument_specs.yml
    ├── README.md
    └── molecule/default/
        ├── molecule.yml
        ├── converge.yml
        └── verify.yml
```

## Foundation Files

### `galaxy.yml` edits

- `description:` — meaningful one-liner
- `license_file: LICENSE` → **`license: [MIT]`** (SPDX; the two keys are mutually exclusive; the skeleton's stray `MIT-0` comment is wrong — the LICENSE text is MIT)
- `tags:` → `[specsnl, specsops, ubuntu, noble, hardening, postgresql, firewall, system]`
- `dependencies:` →

  ```yaml
  community.general: ">=8.0.0,<12.0.0"   # locale_gen, timezone, ufw
  ansible.posix:     ">=1.5.0,<3.0.0"    # sysctl
  ```

- `build_ignore:` — `.github`, `.gitignore`, `.editorconfig`, `.yamllint`,
  `.ansible-lint`, `Taskfile*.yml`, `requirements.txt`, `requirements.yml`,
  `ansible.cfg`, `roles/*/molecule`, `changelogs/fragments`, `dist`, `*.tar.gz`

### `meta/runtime.yml`

Set `requires_ansible: ">=2.16.0"` — must match the `ansible-core` lower bound in
`requirements.txt` and the CI matrix.

## Role Design

### Container detection (shared by every role)

The same `/.dockerenv` guard from the shell scripts, as an Ansible fact:

```yaml
- name: Detect container environment
  ansible.builtin.set_fact:
    in_container: >-
      {{ (ansible_virtualization_type | default('') in
          ['docker','container','containerd','podman','lxc'])
         or (lookup('ansible.builtin.fileglob', '/.dockerenv') | length > 0) }}
```

Gate `ufw enable`, `systemctl state: started`, and `sysctl reload` with
`when: not in_container`. Molecule sets `in_container: true` via host_vars.

### Module FQCNs

| Action                                      | Module                                                            | Collection dep    |
|---------------------------------------------|-------------------------------------------------------------------|-------------------|
| apt update/upgrade/install/autoremove/clean | `ansible.builtin.apt`                                             | builtin           |
| locale gen                                  | `community.general.locale_gen`                                    | community.general |
| timezone                                    | `community.general.timezone`                                      | community.general |
| sysctl                                      | `ansible.posix.sysctl`                                            | ansible.posix     |
| ufw rules/default/enable                    | `community.general.ufw`                                           | community.general |
| sshd drop-in / apt configs / pg tuning      | `ansible.builtin.template`                                        | builtin           |
| service enable/start                        | `ansible.builtin.systemd_service`                                 | builtin           |
| PGDG repo+key                               | `ansible.builtin.deb822_repository` (requires ansible-core ≥2.15) | builtin           |

Do **not** use the deprecated `ansible.builtin.apt_key`. Use `deb822_repository`
for the PGDG keyring — it handles the GPG key and sources list in one idempotent task.

### Key defaults (values match the shell scripts)

- **base:** `base_packages` (curl, git, jq, gnupg, ca-certificates, locales),
  `base_locale/base_lang: en_US.UTF-8`, `base_timezone: UTC`,
  `base_sysctl: {vm.swappiness: "10", fs.file-max: "200000"}`,
  `base_sysctl_file: /etc/sysctl.d/99-base.conf`
- **hardening:** `hardening_ssh_permit_root_login: "no"`,
  `hardening_ssh_password_authentication: "no"`, `hardening_ssh_max_auth_tries: 3`,
  drop-in path `/etc/ssh/sshd_config.d/99-hardening.conf`, `hardening_fail2ban_enabled: true`
- **firewall:** `firewall_default_incoming: deny`, `firewall_default_outgoing: allow`,
  `firewall_rules: [{rule: allow, port: "22", proto: tcp}]`, `firewall_extra_rules: []`
- **unattended_upgrades:** packages `[unattended-upgrades, chrony]`, periodic `"1"`,
  allowed origins `${distro_id}:${distro_codename}-security`,
  autofix/minimal-steps/remove-unused `"true"`
- **postgresql:** `postgresql_version: 18`, `postgresql_port: 5432`,
  `postgresql_manage_firewall: true`,
  key URL `https://www.postgresql.org/media/keys/ACCC4CF8.asc`,
  keyring `/etc/apt/keyrings/postgresql.gpg`,
  `postgresql_tuning: {shared_buffers: 256MB, effective_cache_size: 1GB, work_mem: 4MB,
  maintenance_work_mem: 64MB, max_connections: 100, logging_collector: "on"}`;
  `vars/main.yml` derives `postgresql_conf_d: /etc/postgresql/{{ postgresql_version }}/main/conf.d`
- **cleanup:** `cleanup_paths: [/tmp, /var/tmp, /var/lib/apt/lists]`

### Cross-role firewall pattern

No `meta` dependencies, no cross-role handlers. `postgresql` opens 5432 idempotently
via `community.general.ufw` guarded by `when: postgresql_manage_firewall and not in_container`.
The `firewall` role owns the host-wide baseline plus `firewall_extra_rules` for playbook
composition. Each role stays self-contained whether invoked by Packer or ansible-pull.

## Testing — Molecule

- **Driver:** `docker` (via `molecule-plugins[docker]`). CI runners ship Docker;
  geerlingguy images are Docker-oriented. Podman users set `DOCKER_HOST` to the Podman
  socket — same `molecule.yml` works, documented in README.
- **Scenario:** per-role `molecule/default/` — keeps paths-filter CI matrix clean.
- **Platform:** `geerlingguy/docker-ubuntu2404-ansible` (pin a dated tag; `latest`
  is not reproducible), `command: /usr/lib/systemd/systemd`, `privileged: true`,
  `cgroupns_mode: host`, mount `/sys/fs/cgroup:rw`, tmpfs `/run /tmp`, cap `SYS_ADMIN`.
- **converge.yml:** `become: true` → `include_role: specsnl.specsops.<role>`.
  Before running, install the collection via `ansible-galaxy collection install . --force`
  so the FQCN resolves (Taskfile handles this; CI does it before `molecule test`).
- **verify.yml:** `verifier: ansible` using `ansible.builtin.assert` — slurp sshd
  drop-in and assert contents; `package_facts` for `postgresql-18`; stat `tuning.conf`
  and sysctl file. Idempotence is covered automatically by `molecule test`.

### Lint

- `.ansible-lint`: `profile: production`; exclude `changelogs/`, `.github/`,
  `**/molecule/`. Commands using `shell`/`command` (`gpg --dearmor`, cleanup `find`)
  need `changed_when:`/`creates:` guards.
- `.yamllint`: `extends: default`, `line-length max 160`, `indentation spaces 2`.

## CI

Mirror `specsops-docker-images` conventions: `concurrency` cancel-in-progress,
`actions/checkout@v6`, `actions/setup-python@v5`, `dorny/paths-filter@v4`, final `check`
aggregation job. The `specsnl/github-actions` reusable workflows are Docker-only — Ansible
CI stays inline here; adding reusable Ansible workflows there is a follow-up.

- **`pr.yml`:** `changes` → `lint` (`yamllint` + `ansible-lint`) → `molecule` (matrix
  per role, each step gated by paths-filter output) → `check`
- **`main.yml`:** same lint + molecule without paths-filter (all roles) to catch
  cross-role regressions
- **`tag.yml`** (on `push: tags: ['v*']`): assert tag == `v$(galaxy.yml version)` →
  `ansible-galaxy collection build --output-path ./dist` →
  `ansible-galaxy collection publish ... --api-key ${{ secrets.GALAXY_API_KEY }}`
- **`dependabot.yml`:** `github-actions` + `pip`, weekly

### Release flow

`antsibull-changelog`: add a fragment per change; release = bump `galaxy.yml version` →
`antsibull-changelog release --version X.Y.Z` → commit → `git tag vX.Y.Z` → push →
`tag.yml` publishes.

## Dev Tooling

- **`requirements.txt`:** `ansible-core>=2.16,<2.19`, `molecule>=6.0`,
  `molecule-plugins[docker]>=23.5`, `ansible-lint>=24.0`, `yamllint>=1.35`,
  `antsibull-changelog>=0.27` (pin exact `==` in CI; dependabot keeps them current)
- **`requirements.yml`:** `community.general` + `ansible.posix` (same ranges as
  `galaxy.yml dependencies`)
- **`Taskfile.dist.yml`:** `ansible:deps`, `ansible:lint`, `ansible:format`,
  `ansible:test` (all), `ansible:test:*` / `ansible:converge:*` / `ansible:verify:*`
  (per role), `galaxy:build`, `galaxy:install:local`, `galaxy:publish`,
  `changelog:release:*`

## Implementation Order

1. Edit `galaxy.yml` + `meta/runtime.yml`; add `ansible.cfg`, `.gitignore`,
   `.yamllint`, `.ansible-lint`, `requirements.txt`, `requirements.yml`, `Taskfile.dist.yml`;
   expand `README.md` + `docs/README.md`
2. Confirm `yamllint .` + `ansible-lint` pass on the empty skeleton
3. Roles — each green under `molecule test` before moving on:
   `base` → `firewall` → `hardening` → `unattended_upgrades` → `postgresql` → `cleanup`
4. Add `.github/workflows/{pr,main,tag}.yml` + `dependabot.yml`; validate on a PR
5. `antsibull-changelog init`, add `changelogs/config.yaml` + first fragment
6. Tag `v0.1.0` and verify Galaxy publish (see prerequisite below)

## Verification

- `task ansible:deps` → `task ansible:lint` → `task ansible:test:<role>` per role →
  `task ansible:test` (all)
- `task galaxy:build` → tarball in `./dist`; `task galaxy:install:local` → smoke-test
  `include_role: specsnl.specsops.<role>` in a throwaway play
- Confirm the roles reproduce the shell scripts: sshd drop-in, sysctl values, UFW rules
  incl. 5432, `postgresql-18` installed + tuning.conf, unattended-upgrades config
- Open a PR → `pr.yml` lint + molecule matrix green; cut `v0.1.0` → `tag.yml` publishes

## Key Decisions

| Decision                             | Choice                                      | Rationale                                                                                                   |
|--------------------------------------|---------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Fine-grained roles                   | 6 roles from 3 shell scripts                | Each role tests independently; composes cleanly for both Packer (build) and ansible-pull (runtime)          |
| No `meta` dependencies between roles | self-contained firewall calls in postgresql | Avoids ordering surprises; roles run in isolation from Packer with no orchestration play                    |
| Molecule driver                      | `docker`                                    | CI runners ship Docker; geerlingguy images are Docker-oriented; Podman users set `DOCKER_HOST` — one config |
| PGDG key                             | `deb822_repository`                         | Handles key + sources atomically; avoids deprecated `apt_key`                                               |
| `license: [MIT]` not `license_file`  | SPDX identifier                             | Galaxy importer prefers SPDX; the two keys are mutually exclusive                                           |
| CI inline (not reusable)             | Inline workflows                            | `specsnl/github-actions` has no Ansible workflows yet; add as follow-up                                     |

## External Prerequisites

- The `specsnl` namespace must be claimed/owned on galaxy.ansible.com and a
  `GALAXY_API_KEY` repo secret added before `tag.yml` can publish.
- Keep `requires_ansible`, CI matrix `ansible-core` version, and `requirements.txt`
  lower bound in sync.

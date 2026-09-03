# App Roles — `podman`, `caddy`, `deploy_agent`, `ansible_pull`

## Context

The `golden-app` image in `specsops-golden-images` needs four roles that do not exist in
this collection yet. Its plan
([`plans/app-image.md`](https://github.com/specsnl/specsops-golden-images/blob/main/plans/app-image.md))
covers the playbook, the Packer sources and the CI wiring; the role specifications live
here, alongside the specs for the roles that already shipped.

The authoritative description of what the app VM is and does is
[`docs/architecture.md`](https://github.com/specsnl/specsops-golden-images/blob/main/docs/architecture.md)
in that repo — sections 3, 4, 9 and 13. Where this plan and the architecture disagree,
the architecture wins.

Role names use underscores, matching `unattended_upgrades`. `deploy-agent` is not a
valid Ansible role name.

## Scope boundary

These roles install **software and static scaffolding only**. Per-host configuration —
the Caddyfile, Quadlet unit definitions, `config.json`, SSH keys — is laid down at
runtime by ansible-pull from `specsnl/specsops-ansible`, and must not be baked.

That boundary is what makes these roles reusable in both places: the same role that
Packer runs at build time is the role ansible-pull enforces at runtime.

## Conventions to follow

Every existing role in this collection establishes patterns these four must match:

- **Container detection** via `ansible.builtin.stat` on `/.dockerenv` and
  `/run/.containerenv` plus `ansible_facts['virtualization_type']`, guarded by
  `when: <role>_in_container is not defined` so a caller can override it. Never a
  `fileglob` lookup — lookups run on the controller, which is itself a container here,
  and would misclassify real VMs. Copy the block from
  [roles/firewall/tasks/main.yml](../roles/firewall/tasks/main.yml).
- **No `meta` inter-role dependencies.** Roles that need a firewall rule call
  `community.general.ufw` themselves, behind a `<role>_manage_firewall` flag and a
  `stat` check that ufw is actually installed — see the tail of
  [roles/postgresql/tasks/main.yml](../roles/postgresql/tasks/main.yml), which warns
  rather than failing when it is not.
- **Full skeleton per role**: `defaults/main.yml`, `tasks/main.yml`, `handlers/main.yml`
  where needed, `templates/*.j2`, `meta/main.yml`, `meta/argument_specs.yml`, a
  `README.md` with the variable table, and `molecule/default/`. Every new variable goes
  in **both** `argument_specs.yml` and the README table — ansible-lint checks the former.
- **Molecule** against `geerlingguy/docker-ubuntu2404-ansible` pinned by digest with
  systemd as PID 1; copy the platform block from
  [roles/base/molecule/default/molecule.yml](../roles/base/molecule/default/molecule.yml).
  Add each role to `ROLES` in `Taskfile.dist.yml`.

---

## 1. `podman`

The simplest of the four.

- Install `podman` from Noble `universe`. 4.9.x bundles the Quadlet generator, so no
  extra repositories are needed.
- Do **not** ship Quadlet unit files — those are per-environment and belong to
  ansible-pull.

Variables: `podman_packages` (default `[podman]`) so a caller can add `podman-compose`
or `slirp4netns` without editing the role.

**Verify:** `podman --version` reports 4.x, and
`/usr/lib/systemd/system-generators/podman-system-generator` exists — the latter is what
actually proves Quadlet support, which a version string does not.

## 2. `caddy`

- Add the official Caddy apt repository via `ansible.builtin.deb822_repository` with
  `signed_by` pointing at the Cloudsmith GPG key. Use `deb822_repository`, not the
  deprecated `apt_key`, matching the PGDG handling in `postgresql`.
- Install `caddy`.
- Enable `caddy.service`, started only when not in a container.
- Open 80/tcp and 443/tcp via `community.general.ufw`, behind `caddy_manage_firewall`
  (default `true`) and the same "is ufw installed" guard `postgresql` uses.
- Do **not** ship a Caddyfile. The package's default one is left in place; ansible-pull
  replaces it.

Variables: `caddy_manage_firewall`, `caddy_http_port` (80), `caddy_https_port` (443),
`caddy_repo_key_url`.

**Verify:** `caddy version` reports v2.x, the service is enabled, and the ufw rules
exist. In a container, assert the ufw tasks were skipped rather than that the rules are
present.

## 3. `deploy_agent`

The scaffolding for the deploy-agent webhook receiver
(`docs/architecture.md` §9). Note what is deliberately **absent**.

- Create system user `deploy_agent_user` (default `deploy-agent`) with no shell and no
  home directory.
- Create `/etc/deploy-agent/`, mode `0750`, owned by that user — left empty; the
  `config.json` is ansible-pull's job.
- Install `/etc/sudoers.d/deploy-agent`, mode `0440`, **validated with
  `validate: visudo -cf %s`** so a malformed file can never be written:

  ```text
  deploy-agent ALL=(root) NOPASSWD: /usr/bin/podman pull *
  deploy-agent ALL=(root) NOPASSWD: /usr/bin/systemctl restart app-*.service
  ```

- Install `deploy-agent.service` from a template: runs as the deploy-agent user, listens
  on `127.0.0.1:9000`, with `ProtectHome=true` and `PrivateTmp=true`.
- Enable the unit.

### The binary is not installed

The unit **must** carry `ConditionPathExists=/usr/local/bin/deploy-agent`. The
architecture is explicit that the image bakes the user, sudoers rule and unit but not
the Go binary; the condition is what lets the unit be enabled in the image yet stay
dormant instead of crash-looping.

This decouples the role from `specsnl/deploy-agent`, which is planned but not written —
the role can be authored, tested and shipped before the CLI exists. How the binary is
delivered later (ansible-pull, or a release-fetch step) is a separate decision.

Do not add a `deploy_agent_version` variable or a `get_url` task. An earlier draft of
the golden-image plan had both; they contradicted the architecture and were dropped.

Variables: `deploy_agent_user`, `deploy_agent_listen` (`127.0.0.1:9000`),
`deploy_agent_binary_path`, `deploy_agent_config_dir`.

**Verify:** the user exists with `/usr/sbin/nologin`; `visudo -cf` parses the sudoers
file; the unit is enabled; `/usr/local/bin/deploy-agent` does **not** exist; and the
service is inactive with its start condition unmet — which is the point of the whole
design, so assert it rather than just asserting the file is missing.

## 4. `ansible_pull`

- Install `ansible` (the full package, not just `ansible-core` — ansible-pull playbooks
  will want the batteries).
- Create `/etc/ansible-pull/`.
- Install `/etc/ansible-pull/env` with `ANSIBLE_PULL_REPO=` and
  `ANSIBLE_PULL_BRANCH=main` placeholders.
- Install `ansible-pull.service` (oneshot, `EnvironmentFile`, and a guard that makes it
  a no-op when `ANSIBLE_PULL_REPO` is empty) and `ansible-pull.timer`
  (`OnCalendar` every 30 min, `Persistent=true`).
- Leave both units **disabled**. `specsnl/specsops-ansible` does not exist yet; enabling
  a timer that pulls from nowhere buys nothing.

Variables: `ansible_pull_repo` (default `""`), `ansible_pull_branch` (`main`),
`ansible_pull_interval` (`30min`), `ansible_pull_enabled` (default `false`, gating
whether the units get enabled).

**Verify:** both unit files exist, the timer is **not** enabled, and the env file
contains the empty-repo placeholder. Also assert the service is a no-op with an empty
repo — start it and confirm success rather than failure.

---

## Implementation order

Each role green under `molecule test` before the next:

1. `podman` — smallest, establishes the pattern
2. `caddy` — adds the deb822 repo + ufw composition
3. `deploy_agent` — the most detail, and the one with a real correctness trap in the
   `ConditionPathExists` guard
4. `ansible_pull`
5. Collection wiring: `Taskfile.dist.yml` `ROLES`, `galaxy.yml` tags + version bump,
   root `README.md` role index, `docs/README.md`, changelog fragment

## Verification

```bash
task ansible:lint
task ansible:test:podman
task ansible:test:caddy
task ansible:test:deploy_agent
task ansible:test:ansible_pull
task ansible:test          # all roles, catches cross-role regressions
task md:fixstyle
```

`molecule test` includes an idempotence step, which is what will catch an unguarded
`command` task — the likely offenders here are anything shelling out to `caddy` or
`systemctl`.

Beyond molecule, the honest end-to-end check is the `golden-app` image itself: build it
and boot a VM per the verification section of the golden-image plan.

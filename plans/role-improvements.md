# Role Improvements — Provisioning Robustness

## Context

The roles in this collection were authored from the older Packer shell scripts per
`plans/collection-init.md`. Reviewing them against a known-good production provisioning
setup for the same workloads surfaced one latent correctness bug — the sshd drop-in never
takes effect on Ubuntu cloud images — plus several unattended-run robustness gaps in
`base` and a handful of small config completions.

It also surfaced material with no home in the current role set. This plan takes the two
self-contained pieces (`swap`, `logrotate`) and leaves the rest as a documented backlog.

## Scope decisions

- Improve the six existing roles; add two new roles (`swap`, `logrotate`).
- `postgresql` gains `listen_addresses` + `pg_hba` management, both defaulting to
  localhost-only. Role/database creation stays out of scope.
- The sshd drop-in moves to `00-hardening.conf`; cloud-init's file is left in place.

## Explicitly rejected

Record these in the PR description so they aren't "rediscovered" later:

- `fs.protected_regular = 0` — a security downgrade that disables kernel protection
  against symlink/hardlink attacks in world-writable dirs. Contradicts `hardening`.
- `ssl_protocols TLSv1 TLSv1.1` — obsolete, insecure.
- A `logrotate.timer` override with `OnCalendar=*:0/1` — runs logrotate every minute.
- `pg_hba: host all all 0.0.0.0/0 md5` — we default to local-only.
- `apt-key add` for the PGDG key — deprecated; our `deb822_repository` + `signed_by` in
  [roles/postgresql/tasks/main.yml](../roles/postgresql/tasks/main.yml) is already correct.
- `--force-yes` — removed from modern apt.

---

## 1. `hardening` — sshd drop-in ordering (the actual bug)

OpenSSH's `Include` is **first-obtained-value-wins**. Ubuntu cloud images ship
`/etc/ssh/sshd_config.d/50-cloud-init.conf` containing `PasswordAuthentication yes`. At
`99-`, our drop-in is parsed *after* it and silently ignored on exactly the images this
collection targets.

**Files:** [roles/hardening/defaults/main.yml](../roles/hardening/defaults/main.yml),
[roles/hardening/tasks/main.yml](../roles/hardening/tasks/main.yml),
`meta/argument_specs.yml`, `README.md`, molecule `prepare.yml` + `verify.yml`.
Rename the template `sshd-99-hardening.conf.j2` → `sshd-hardening.conf.j2`.

- `hardening_ssh_dropin_path` default → `/etc/ssh/sshd_config.d/00-hardening.conf`.
- Add a task removing `/etc/ssh/sshd_config.d/99-hardening.conf` so already-provisioned
  hosts converge. Guard it with `when: hardening_ssh_dropin_path != <old path>` so a user
  who pins the old path isn't fighting the role.
- Extend the template with `KbdInteractiveAuthentication no` — without it,
  `PasswordAuthentication no` can still be bypassed via PAM keyboard-interactive on some
  configs. New var `hardening_ssh_kbd_interactive_authentication`, default `"no"`.

**Test (this is the part that matters):** the previous `verify.yml` slurped the file and
grepped its contents, which passes even when the setting is inert. Replace with an
*effective config* assertion:

- `prepare.yml`: after installing openssh-server, write
  `/etc/ssh/sshd_config.d/50-cloud-init.conf` containing `PasswordAuthentication yes`,
  reproducing the cloud image.
- `verify.yml`: run `sshd -T` (`changed_when: false`) and assert the output contains
  `passwordauthentication no`, `permitrootlogin no`, `maxauthtries 3`,
  `kbdinteractiveauthentication no`. Also assert the old `99-` file is absent.

This test fails against the unfixed role and passes after the rename — that is the
acceptance criterion for this section.

## 2. `base` — unattended-run robustness

All of the following must run **before** the existing "Update apt cache and upgrade
packages" task in [roles/base/tasks/main.yml](../roles/base/tasks/main.yml).

- **needrestart.** On Ubuntu 22.04/24.04 `$nrconf{restart}='i'` makes apt block on an
  interactive whiptail service-restart prompt, hanging the play. Write
  `/etc/needrestart/conf.d/99-ansible.conf` with `$nrconf{restart} = 'a';` — a drop-in,
  not a `sed` on the main conf. Var `base_needrestart_automatic` (default `true`).
- **apt lock timeout.** Write `/etc/apt/apt.conf.d/90lock-timeout` with
  `DPkg::Lock::Timeout` and `APT::Get::Lock::Timeout`. This removes the whole class of
  races against `unattended-upgrades` on freshly booted cloud instances — directly relevant
  since this collection also *installs* unattended-upgrades. Var `base_apt_lock_timeout`
  (default `300`). Makes an `apt_wait`/fuser polling loop unnecessary.
- **dpkg conffile options.** Add `dpkg_options: force-confdef,force-confold` to the upgrade
  task; without it a dist-upgrade can prompt on modified conffiles. Var `base_apt_dpkg_options`.
- **`acl` in `base_packages`.** Required for Ansible `become_user` to an unprivileged user
  to work without world-readable temp files. Also add `unzip`.
- **`vm.vfs_cache_pressure: "50"`** in `base_sysctl`, alongside the existing `vm.swappiness`.
- **IPv4 precedence in `/etc/gai.conf`** — uncomment `precedence ::ffff:0:0/96  100`. Only
  helps hosts with broken/absent IPv6. Add as `base_prefer_ipv4` defaulting to **`false`**
  (opt-in); it is a behaviour change for dual-stack hosts.

**Verify additions:** assert both drop-in files exist and `acl` is installed.

## 3. `postgresql`

**Files:** `defaults/main.yml`, `tasks/main.yml`, `templates/tuning.conf.j2`,
`vars/main.yml`, `meta/argument_specs.yml`, `README.md`, `verify.yml`.

- **`timezone` and `log_timezone`** — rendered from a dedicated `postgresql_timezone` var,
  replacing a fragile `sed "s/localtime/UTC/"` on `postgresql.conf`.
- ~~**`postgresql-contrib-<version>`**~~ — **dropped during implementation.**
  It turned out to be a pure virtual package with no candidate: versioned contrib packages
  stopped at 9.6, and since PostgreSQL 10 the modules ship inside `postgresql-<version>`.
  apt satisfies the name via `Provides` and installs nothing. The molecule verify asserts
  `pg_stat_statements.control` is present instead, which guards the decision.
- **`listen_addresses`** — dedicated var defaulting to `localhost`. Previously the role
  opened ufw 5432 against a server that only listens on the loopback, which is misleading.
- **`pg_hba` entries** — new var `postgresql_hba_entries`, defaulting to the empty list so
  nothing changes out of the box. Rendered into `pg_hba.conf` via an Ansible-managed
  `blockinfile` marker rather than replacing the whole file, so the distribution's default
  local/peer rules stay intact. Notifies the existing `Restart postgresql` handler.
- **Tighten the ufw rule:** gate it on `postgresql_listen_addresses` not being loopback, so
  the port is only opened when the server actually listens off-loopback.

**Verify additions:** assert `listen_addresses = 'localhost'` and `timezone = 'UTC'` land in
`tuning.conf`, and that the blockinfile markers are absent with default (empty) entries.

## 4. `unattended_upgrades`

[20auto-upgrades.j2](../roles/unattended_upgrades/templates/20auto-upgrades.j2) set two of
the four periodic keys. Add:

- `APT::Periodic::Download-Upgradeable-Packages` (var, default `"1"`)
- `APT::Periodic::AutocleanInterval` (var, default `"7"`) — without it `/var/cache/apt`
  grows unbounded on long-lived hosts.

**Verify addition:** assert all four keys are present in the rendered file.

## 5. New role: `swap`

`base` sets `vm.swappiness` but nothing ever creates swap, so on most cloud VMs that
tunable is inert.

- Vars: `swap_enabled` (`true`), `swap_file_path` (`/swapfile`), `swap_size` (`1G`),
  `swap_swappiness` (`"30"`), `swap_vfs_cache_pressure` (`"50"`).
- Tasks: the container-detection `set_fact` pattern used by every other role in this
  collection; then `community.general.filesize` (non-sparse) → `mkswap` → the `mount`
  module with `fstype: swap`, `state: present` for fstab → `swapon`.
- The swap tasks must be a no-op when `swap_in_container` — `swapon` cannot work in a
  Docker container. Also skip when the host already has active swap and our file does not
  exist, so a VM image with vendor swap isn't given a second swap area.
- Sysctl values are set here via `ansible.posix.sysctl` into `/etc/sysctl.d/99-swap.conf`,
  **not** merged into `base_sysctl` — keeps the roles independently composable. Note the
  overlap: `base_sysctl.vm.swappiness` is `"10"` and this role defaults to `"30"`; whichever
  drop-in sorts last wins. Document that it should be set in one role or the other.

**Molecule:** honest but limited — converge in a container, assert the swap tasks skipped
cleanly, that `/swapfile` and the fstab entry were not created, and that the sysctl drop-in
still rendered. Real verification is manual on a VM.

## 6. New role: `logrotate`

The common approach is to `sed` `maxsize` into five vendor logrotate configs. That is
fragile and gets clobbered on package upgrade. Do it once, globally:

- Var `logrotate_maxsize` (default `100M`), `logrotate_compress` (default `true`).
- Manage the global directives in `/etc/logrotate.conf` with `lineinfile`, which apply to
  every job that doesn't override them — covering nginx, fail2ban, rsyslog, ufw and redis
  without touching any package-owned file.
- Optional escape hatch `logrotate_file_overrides` (default `{}`) for per-service files
  that do set their own `maxsize`, applied with `lineinfile` on that specific file guarded
  by a `stat` so absent services are skipped.

**Molecule:** converge, assert `maxsize 100M` in `/etc/logrotate.conf`, assert the per-file
override applied, and that `logrotate --debug /etc/logrotate.conf` reports no errors.

## 7. Collection-level wiring

Each new role needs the full skeleton every existing role has: `defaults/main.yml`,
`tasks/main.yml`, `meta/main.yml`, `meta/argument_specs.yml`, `README.md` with the variable
table, and `molecule/default/{molecule,converge,verify}.yml`. Copy the structure from
[roles/cleanup/](../roles/cleanup/) (smallest) and the molecule platform block from
[roles/base/molecule/default/molecule.yml](../roles/base/molecule/default/molecule.yml).

- `Taskfile.dist.yml`: add `swap` and `logrotate` to the `ROLES` var.
- `galaxy.yml`: extend `description` and `tags`; bump `version` `0.1.0` → `0.2.0`.
- Root `README.md`: add both roles to the role index.
- `changelogs/fragments/`: create the directory and add fragments — a `bugfixes` entry for
  the sshd drop-in ordering, a `security_fixes` entry for `KbdInteractiveAuthentication`,
  `breaking_changes` for the changed defaults, `minor_changes` for the rest.
- Every new variable must be added to the role's `meta/argument_specs.yml` **and** its
  `README.md` table — both are part of the existing convention and ansible-lint checks the
  former.

## Commit sequence

Each of these should be independently green:

1. `hardening`: drop-in rename + `sshd -T` test (the bug fix — land first)
2. `base`: needrestart + apt lock timeout + dpkg options
3. `base`: packages, sysctl, gai.conf
4. `postgresql`: timezone, listen_addresses, pg_hba, ufw gating
5. `unattended_upgrades`: periodic keys
6. new role `swap`
7. new role `logrotate`
8. collection wiring: Taskfile, galaxy.yml, README, changelog fragments

## Verification

Per role, via the existing Taskfile:

```bash
task ansible:lint                  # yamllint + ansible-lint
task ansible:test:hardening        # includes idempotence
task ansible:test:base
task ansible:test:postgresql
task ansible:test:unattended_upgrades
task ansible:test:swap
task ansible:test:logrotate
task ansible:test                  # all roles
task md:fixstyle                   # markdown lint + table alignment
```

The molecule `test_sequence` already includes `idempotence`, which will catch any of the
new `lineinfile`/`blockinfile`/`command` tasks that aren't properly guarded — the swap
`filesize`/`mkswap` steps and the logrotate `lineinfile` calls are the likely offenders.

The single most important check: `task ansible:test:hardening` must **fail** before the
rename once the new `prepare.yml`/`verify.yml` are in place, and pass after. Run it in that
order to confirm the test actually has teeth.

Beyond molecule, `swap` cannot be meaningfully verified in a container. Before tagging
`0.2.0`, run the full role set once against a real Ubuntu 24.04 cloud VM and confirm:
`swapon --show` reports the swapfile, `sshd -T | grep -E 'passwordauth|permitrootlogin'`
shows `no`, and `sudo -u postgres psql -c 'show timezone'` returns `UTC`.

# specsnl.specsops.ansible_pull

Installs the ansible-pull scaffolding on Ubuntu 26.04: the full `ansible` package,
`/etc/ansible-pull/env`, and an `ansible-pull.service` / `ansible-pull.timer` pair.
Both ship **disabled and inert** — the service is a no-op until a repository is
configured, and the timer is not enabled unless you ask for it.

## Variables

| Variable                    | Default                 | Description                                      |
|-----------------------------|-------------------------|--------------------------------------------------|
| `ansible_pull_repo`         | `""`                    | Repo to pull from; empty means no-op             |
| `ansible_pull_branch`       | `main`                  | Branch to pull                                   |
| `ansible_pull_interval`     | `*:0/30`                | Timer `OnCalendar` expression (every 30 minutes) |
| `ansible_pull_enabled`      | `false`                 | Enable and start the timer                       |
| `ansible_pull_checkout_dir` | `/var/lib/ansible-pull` | Directory the repo is cloned into                |
| `ansible_pull_playbook`     | `local.yml`             | Playbook run from the checkout                   |
| `ansible_pull_packages`     | `[ansible, git]`        | Packages to install                              |

## The empty-repo no-op

`ANSIBLE_PULL_REPO=` in the env file is the shipping state. The service's `ExecStart`
checks it first and exits 0 with a log line rather than running `ansible-pull`, so a
host that is enabled before its repository exists reports success instead of a failed
unit every half hour.

Point a host at a repository by rewriting the env file — through this role's variables,
or by hand:

```ini
ANSIBLE_PULL_REPO=https://github.com/specsnl/specsops-ansible.git
ANSIBLE_PULL_BRANCH=main
```

## The interval is a calendar expression

`OnCalendar` takes a systemd *calendar event*, not a timespan: `30min` is a valid
`OnUnitActiveSec` value but not a valid `OnCalendar` one, and systemd would leave the
timer parked forever. Every 30 minutes is `*:0/30`. The role runs
`systemd-analyze calendar` on the value and fails with that explanation rather than
writing a timer that never fires.

`Persistent=true` makes the timer catch up on a missed run after downtime, and a
`RandomizedDelaySec=5min` keeps a fleet from hitting the git host in lockstep.

## Unit states

| Unit                   | Shipped state | With `ansible_pull_enabled: true` |
|------------------------|---------------|-----------------------------------|
| `ansible-pull.timer`   | `disabled`    | `enabled`, started                |
| `ansible-pull.service` | `static`      | `static`                          |

The service has no `[Install]` section on purpose — it is triggered by the timer, so
`systemctl is-enabled` reports `static` rather than `disabled`. Nothing runs it on its
own; `systemctl start ansible-pull.service` remains the way to force a pull.

## Out of scope

The role ships no playbook, no inventory and no SSH keys. What ansible-pull applies
lives in the pulled repository, which is what keeps the same role usable at image-build
time and at runtime.

## Example

```yaml
- hosts: app
  become: true
  roles:
    - specsnl.specsops.base
    - specsnl.specsops.ansible_pull
```

Enabled, for a host that has somewhere to pull from:

```yaml
- hosts: app
  become: true
  roles:
    - role: specsnl.specsops.ansible_pull
      vars:
        ansible_pull_repo: https://github.com/specsnl/specsops-ansible.git
        ansible_pull_enabled: true
```

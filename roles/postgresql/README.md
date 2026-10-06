# specsnl.specsops.postgresql

PostgreSQL setup for Ubuntu 26.04: adds the official PGDG apt repository, installs
the specified version, optionally moves the cluster to a data directory of your
choosing, deploys a tuning configuration, manages connection access,
enables the service, and optionally opens the port in ufw.

## Variables

| Variable                      | Default                                              | Description                       |
|-------------------------------|------------------------------------------------------|-----------------------------------|
| `postgresql_version`          | `18`                                                 | PostgreSQL major version          |
| `postgresql_port`             | `5432`                                               | Port                              |
| `postgresql_pgdg_key_url`     | `https://www.postgresql.org/media/keys/ACCC4CF8.asc` | PGDG signing key URL              |
| `postgresql_listen_addresses` | `localhost`                                          | `listen_addresses` value          |
| `postgresql_timezone`         | `UTC`                                                | `timezone` and `log_timezone`     |
| `postgresql_data_directory`   | `""`                                                 | Data directory (see below)        |
| `postgresql_manage_firewall`  | `true`                                               | Open port in ufw (see below)      |
| `postgresql_firewall_sources` | `[]`                                                 | Sources the port is open to       |
| `postgresql_hba_entries`      | `[]`                                                 | Extra `pg_hba.conf` entries       |
| `postgresql_tuning`           | see below                                            | Map of postgresql.conf parameters |

## Connection access

The server listens on loopback only by default. The ufw rule is gated on
`postgresql_listen_addresses`, so setting `postgresql_manage_firewall: true` on a
loopback-only server does not open a port to something nothing can reach — you must
also widen `listen_addresses`.

The role does not depend on the `firewall` role, so ufw may not be installed. When it
is missing the rule is skipped with a warning rather than failing the play, which
would otherwise abort with PostgreSQL already listening off-loopback. Run
`specsnl.specsops.firewall` first if you want the port opened.

By default the port is open to every source, and the role warns about that when
`postgresql_listen_addresses` is `'*'`. List the networks that need access in
`postgresql_firewall_sources` to narrow it down. Each entry becomes its own allow rule,
limited to a source address or network (`from_ip`), an incoming interface
(`interface`), or both. Give `from_ip` as a network address: ufw normalises
`10.20.0.17/28` to `10.20.0.16/28`, and the role would then remove and re-add the rule
on every run.

```yaml
postgresql_listen_addresses: "*"
postgresql_firewall_sources:
  - from_ip: 10.20.0.16/28
    interface: eth1
```

The role tags its rules with the ufw comment `specsnl.specsops.postgresql`. A source
dropped from the list has its rule removed on the next run, and a non-empty list also
removes the rule open to every source, including the untagged one that earlier releases
added. Rules without the tag, such as those from the `firewall` role, are left alone.
Nothing is removed when the role stops managing the firewall, either because
`postgresql_manage_firewall` is false or because the server went back to listening on
loopback only.

`postgresql_hba_entries` are appended to `pg_hba.conf` inside an Ansible-managed
marker block, leaving the distribution's default `local`/`peer` rules intact. The
block is removed when the list is empty.

```yaml
postgresql_listen_addresses: "*"
postgresql_hba_entries:
  - type: host
    database: app
    user: app
    address: 10.0.0.0/8
    method: scram-sha-256
```

## Data directory

`postgresql_data_directory` keeps the cluster's data somewhere other than the Debian
default `/var/lib/postgresql/<version>/main` — typically a block volume, so the VM
can be rebuilt from an image and the data reattached. Empty (the default) leaves the
cluster where it is.

This role does not mount anything. Mount the volume first, for example with the
`block_volume` role. The parent of the target must exist, or the role fails, so that
a volume that failed to mount does not get a fresh cluster written to the OS disk.

What happens depends on what the target holds:

- **Nothing, or it does not exist yet** — first use. The server is stopped and the
  freshly initialised default cluster is copied in with `rsync -a`, owned by
  `postgres` with mode `0700`.
- **A `PG_VERSION` matching `postgresql_version`** — a rebuild. The cluster is adopted
  as-is and never re-initialised.
- **A `PG_VERSION` for another major version** — the role fails. Upgrade it with
  `pg_upgradecluster` first.
- **Anything else** — the role fails rather than delete or overwrite it. A fresh
  filesystem holds `lost+found`, so point the variable at a subdirectory of the mount
  point, not at the mount point itself.

`data_directory` is then set in `postgresql.conf`, which is where `pg_ctlcluster`
reads it from, and the server is restarted from the new location. The old default
directory is left in place. Don't set `data_directory` in `conf.d` as well: it is
included last and would win.

```yaml
postgresql_data_directory: /mnt/pgdata/main
```

## Contrib extensions

No separate contrib package is installed. `postgresql-contrib-<version>` is a pure
virtual package with no candidate — versioned contrib packages stopped at 9.6, and
since PostgreSQL 10 the modules (`pg_stat_statements`, `pgcrypto`, …) ship inside
`postgresql-<version>`. Just `CREATE EXTENSION`.

## Out of scope

Creating roles and databases is out of scope — do that in the consuming playbook with
`community.postgresql`.

## Tuning defaults

```yaml
postgresql_tuning:
  shared_buffers: 256MB
  effective_cache_size: 1GB
  work_mem: 4MB
  maintenance_work_mem: 64MB
  max_connections: 100
  logging_collector: "on"
```

`listen_addresses`, `port`, `timezone` and `log_timezone` have dedicated variables and
are rendered separately, so they do not belong in this map.

## Example

```yaml
- hosts: db
  become: true
  roles:
    - specsnl.specsops.base
    - specsnl.specsops.firewall
    - specsnl.specsops.postgresql
```

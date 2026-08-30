# specsnl.specsops.postgresql

PostgreSQL setup for Ubuntu 24.04: adds the official PGDG apt repository, installs
the specified version, deploys a tuning configuration, manages connection access,
enables the service, and optionally opens the port in ufw.

## Variables

| Variable                      | Default                                              | Description                       |
|-------------------------------|------------------------------------------------------|-----------------------------------|
| `postgresql_version`          | `18`                                                 | PostgreSQL major version          |
| `postgresql_port`             | `5432`                                               | Port                              |
| `postgresql_pgdg_key_url`     | `https://www.postgresql.org/media/keys/ACCC4CF8.asc` | PGDG signing key URL              |
| `postgresql_listen_addresses` | `localhost`                                          | `listen_addresses` value          |
| `postgresql_timezone`         | `UTC`                                                | `timezone` and `log_timezone`     |
| `postgresql_manage_firewall`  | `true`                                               | Open port in ufw (see below)      |
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

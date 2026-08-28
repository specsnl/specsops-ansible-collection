# specsnl.specsops.postgresql

PostgreSQL setup for Ubuntu 24.04: adds the official PGDG apt repository, installs
the specified version, deploys a tuning configuration, enables the service, and
optionally opens the port in ufw.

## Variables

| Variable                     | Default                                              | Description                              |
|------------------------------|------------------------------------------------------|------------------------------------------|
| `postgresql_version`         | `18`                                                 | PostgreSQL major version                 |
| `postgresql_port`            | `5432`                                               | Port (used for ufw rule)                 |
| `postgresql_pgdg_key_url`    | `https://www.postgresql.org/media/keys/ACCC4CF8.asc` | PGDG signing key URL                     |
| `postgresql_manage_firewall` | `true`                                               | Open port in ufw (skipped in containers) |
| `postgresql_tuning`          | see defaults                                         | Map of postgresql.conf parameters        |

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

## Example

```yaml
- hosts: db
  become: true
  roles:
    - specsnl.specsops.base
    - specsnl.specsops.firewall
    - specsnl.specsops.postgresql
```

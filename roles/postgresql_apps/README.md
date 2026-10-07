# specsnl.specsops.postgresql_apps

Gives every app on a shared PostgreSQL server the same setup:

- its own login role
- its own database, owned by that role
- `CONNECT` on that database revoked from `PUBLIC`, so other apps' roles cannot connect
- a `pg_hba.conf` line that lets only that role into only that database, from only the
  app network

The role does not depend on `postgresql` and only assumes a server running on the same
host. It connects as `postgres` over the local Unix socket, using `community.postgresql`
and `python3-psycopg2`, which it installs.

## Variables

| Variable               | Default | Description                       |
|------------------------|---------|-----------------------------------|
| `postgresql_apps`      | `[]`    | The apps (see below)              |
| `postgresql_apps_port` | `5432`  | Port of the local server's socket |

Each entry of `postgresql_apps` takes these keys:

| Key            | Default         | Description                                            |
|----------------|-----------------|--------------------------------------------------------|
| `name`         | —               | Login role, and owner of the database                  |
| `password`     | —               | Password of the role. Required when `present`          |
| `database`     | `name`          | Database of the app                                    |
| `allowed_cidr` | —               | Network the app connects from. Required when `present` |
| `auth_method`  | `scram-sha-256` | Authentication method of the `pg_hba.conf` line        |
| `state`        | `present`       | `present` or `absent` (see below)                      |

`name` and `database` must be lowercase identifiers (`^[a-z_][a-z0-9_]*$`), so they
need no quoting in SQL or in `pg_hba.conf`. They may not start with `pg_`, and may not
be a `pg_hba.conf` keyword (`all`, `sameuser`, `samerole`, `samegroup`, `replication`),
`postgres`, `public` or a template database. Every entry needs its own name and its own
database.

The password is passed in. The role does no secret lookups, so read it from your
secret store in the playbook. The task that sets it runs with `no_log`.

## pg_hba.conf

The role asks the server for its `hba_file` and writes one `host` line per present app
inside its own marker block, `specsnl.specsops.postgresql_apps`. The `postgresql`
role's block and the distribution's default rules are left as they are.

The lines are appended to the end of the file, and PostgreSQL uses the first line that
matches. Debian's default rules allow every role from `127.0.0.1` and `::1`, so on
loopback the revoked `CONNECT` is what keeps one app out of another's database. From
the app network, nothing but these lines matches.

After a change, the role checks `pg_hba_file_rules` for errors, then reloads the server.
It does not restart it. PostgreSQL keeps its old rules when a reload finds an error, so
the role fails the play instead of reloading a file that does not parse.

## Removing an app

`state: absent` removes the app's `pg_hba.conf` line and sets `NOLOGIN` on its role. It
**never** drops the role or the database. Drop those by hand once the data is no longer
needed. Connections that are already open are not terminated. An absent app whose role
does not exist is skipped.

## Requirements

Tasks run as `postgres` through `become_user`, which needs `acl` on the target (the
`base` role installs it) or pipelining.

## Example

```yaml
- hosts: db
  become: true
  roles:
    - specsnl.specsops.postgresql
    - role: specsnl.specsops.postgresql_apps
      vars:
        postgresql_apps:
          - name: poc_app
            password: "{{ lookup('community.general.onepassword', 'poc_app db', field='password') }}"
            allowed_cidr: 10.20.0.16/28
          - name: old_app
            state: absent
```

The server must listen on the app network for these lines to matter. With the
`postgresql` role, set `postgresql_listen_addresses` and, if ufw is in use,
`postgresql_firewall_sources`.

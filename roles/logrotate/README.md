# specsnl.specsops.logrotate

Caps rotated log size and enables compression for Ubuntu 24.04.

## Why global rather than per-file

The obvious approach is to `sed` a `maxsize` line into each of
`/etc/logrotate.d/{nginx,fail2ban,rsyslog,ufw,redis-server}`. Those files are
package-owned, so the edits are lost on the next package upgrade, and the approach only
covers services you thought to enumerate.

`maxsize` and `compress` are global directives. Setting them once in
`/etc/logrotate.conf` covers every job that does not explicitly override them, survives
package upgrades, and needs no list of services. `logrotate_file_overrides` exists for
the handful of jobs that do set their own.

## Variables

| Variable                   | Default | Description                               |
|----------------------------|---------|-------------------------------------------|
| `logrotate_maxsize`        | `100M`  | Global `maxsize` directive                |
| `logrotate_compress`       | `true`  | Global `compress` directive               |
| `logrotate_file_overrides` | `{}`    | Per-file `maxsize`, for jobs that set own |

```yaml
logrotate_file_overrides:
  /etc/logrotate.d/nginx: 50M
```

## Example

```yaml
- hosts: all
  become: true
  roles:
    - specsnl.specsops.logrotate
```

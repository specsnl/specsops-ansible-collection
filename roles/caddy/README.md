# specsnl.specsops.caddy

Caddy setup for Ubuntu 26.04: adds the official Cloudsmith apt repository, installs
Caddy, enables the service, and optionally opens 80/443 in ufw.

## Variables

| Variable                | Default                                                | Description                |
|-------------------------|--------------------------------------------------------|----------------------------|
| `caddy_repo_key_url`    | `https://dl.cloudsmith.io/public/caddy/stable/gpg.key` | Repository signing key URL |
| `caddy_http_port`       | `80`                                                   | HTTP port opened in ufw    |
| `caddy_https_port`      | `443`                                                  | HTTPS port opened in ufw   |
| `caddy_manage_firewall` | `true`                                                 | Open ports in ufw          |

## Firewall

The role does not depend on the `firewall` role, so ufw may not be installed. When it
is missing the rules are skipped with a warning rather than failing the play, which
would otherwise abort with Caddy already listening. Run `specsnl.specsops.firewall`
first if you want the ports opened.

The rules are also skipped in containers, where `ufw` cannot manage the host's
netfilter tables.

## Out of scope

The role ships no Caddyfile. The package's default one is left in place; ansible-pull
replaces it at runtime, so the same role is safe to run at image-build time.

## Example

```yaml
- hosts: app
  become: true
  roles:
    - specsnl.specsops.base
    - specsnl.specsops.firewall
    - specsnl.specsops.caddy
```

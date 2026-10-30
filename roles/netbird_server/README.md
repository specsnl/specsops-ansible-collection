# specsnl.specsops.netbird_server

The self-hosted NetBird control plane as Podman Quadlets behind the host's Caddy. Since
v0.65.0 NetBird runs management, signal, relay and STUN, with an embedded identity
provider, in one container, `netbirdio/netbird-server`, next to `netbirdio/dashboard`.
NetBird only documents Docker Compose; this role runs both images as Quadlets published on
`127.0.0.1`, writes the server's `config.yaml`, adds a Caddy site after NetBird's own
Caddy setup, and opens the STUN port in ufw.

Checked with server 0.81.5 and dashboard v2.96.0 under Podman: the API, the OIDC
discovery, the dashboard and gRPC answer through Caddy, STUN listens, and the server
restarts cleanly with its data. Idle, the server uses about 100 MiB and the dashboard
25 MiB.

## Variables

| Variable                                | Default                              | Description                                   |
|-----------------------------------------|--------------------------------------|-----------------------------------------------|
| `netbird_server_version`                | —                                    | Server image tag, e.g. `0.81.5`               |
| `netbird_server_dashboard_version`      | —                                    | Dashboard image tag, e.g. `v2.96.0`           |
| `netbird_server_image`                  | `docker.io/netbirdio/netbird-server` | Server image, without the tag                 |
| `netbird_server_dashboard_image`        | `docker.io/netbirdio/dashboard`      | Dashboard image, without the tag              |
| `netbird_server_domain`                 | —                                    | Host name, e.g. `netbird.example.com`         |
| `netbird_server_auth_secret`            | —                                    | Relay authentication secret                   |
| `netbird_server_store_encryption_key`   | —                                    | Store encryption key; must never change       |
| `netbird_server_session_cookie_key`     | —                                    | Embedded IdP session cookie key               |
| `netbird_server_data_dir`               | `/srv/netbird`                       | Data directory on the data disk               |
| `netbird_server_data_dir_require_mount` | `true`                               | Fail when the data would land on `/`          |
| `netbird_server_port`                   | `8081`                               | Server host port, on `127.0.0.1` only         |
| `netbird_server_dashboard_port`         | `8080`                               | Dashboard host port, on `127.0.0.1` only      |
| `netbird_server_stun_port`              | `3478`                               | STUN UDP port, on every address               |
| `netbird_server_trusted_proxies`        | `[10.88.0.1/32]`                     | Where Caddy's traffic reaches the server from |
| `netbird_server_setup_pat_enabled`      | `false`                              | Let `POST /api/setup` return a PAT            |
| `netbird_server_manage_firewall`        | `true`                               | Open the STUN port in ufw                     |
| `netbird_server_caddy_sites_dir`        | `/etc/caddy/sites`                   | Where the Caddy site file goes                |

The versions, the domain and the three secrets are required. Pin the two images
separately: the dashboard has its own version line. Generate each secret once and keep it
in your secret store:

```bash
openssl rand -base64 32
```

`netbird_server_store_encryption_key` must never change, like `pocket_id_encryption_key`:
the store is unreadable without the key it was written with.

## Requirements

The role installs neither Podman nor Caddy. Run `specsnl.specsops.podman` and
`specsnl.specsops.caddy` first. As with `pocket_id`, the Caddyfile you deploy has to
import the site file:

```caddyfile
import /etc/caddy/sites/*
```

## The server config

`/etc/netbird-server/config.yaml`, `0600` root, is mounted into the server as
`/etc/netbird/config.yaml`. It sets:

- `listenAddress: ":80"` and `exposedAddress: https://<domain>:443`, since Caddy
  terminates TLS;
- `stunPorts`, `authSecret` and `dataDir: /var/lib/netbird`;
- the embedded IdP: `issuer: https://<domain>/oauth2`, the dashboard's `/nb-auth` and
  `/nb-silent-auth` and the CLI's `http://localhost:53000/` redirect URIs, and
  `sessionCookieEncryptionKey`;
- `reverseProxy.trustedHTTPProxies` and `reverseProxy.trustedPeers`, both
  `netbird_server_trusted_proxies`;
- the SQLite store with `encryptionKey`;
- `disableAnonymousMetrics: true`.

Without `trustedPeers` the server trusts `X-Forwarded-For` from every client, and logs a
warning about it. Caddy connects to `127.0.0.1:<port>`, which Podman masquerades to its
default network's gateway, so the server sees `10.88.0.1` as the proxy. Change
`netbird_server_trusted_proxies` when the containers run on another network.

The task that writes the config runs with `no_log`. The dashboard's environment,
pointing it at the server and the embedded IdP, is in `/etc/netbird-server/dashboard.env`.

## The data directory

`netbird_server_data_dir` is mounted as `/var/lib/netbird` and holds:

- `store.db`: accounts, peers, groups, policies, setup keys;
- `idp.db`: the embedded IdP's users and sessions;
- `events.db`: the activity log;
- the GeoLite files, which the server downloads again when they are missing.

Back up the three databases, and keep the store key with, but not in, the backups. The
server exits when it cannot open its database, so the role creates the directory, `0700`
root. As with `pocket_id`, the role fails when it would land on the root filesystem; set
`netbird_server_data_dir_require_mount: false` to allow that on purpose.

## The containers

- `netbird-server.container`: `PublishPort=127.0.0.1:<netbird_server_port>:80`,
  `PublishPort=<stun>:<stun>/udp`, the data directory and the config, and
  `NB_SETUP_PAT_ENABLED=true` with `netbird_server_setup_pat_enabled`.
- `netbird-dashboard.container`: `PublishPort=127.0.0.1:<netbird_server_dashboard_port>:80` and
  `dashboard.env`.

Quadlet units are generated, so they cannot be enabled; their `[Install]` sections start
them at boot. A change to a Quadlet, the config or the dashboard environment restarts the
container it belongs to. Logs are in `journalctl -u netbird-server` and
`journalctl -u netbird-dashboard`.

## The Caddy site

`/etc/caddy/sites/netbird.caddy` follows NetBird's documented Caddy setup:

```caddyfile
netbird.example.com {
    @grpc header Content-Type application/grpc*
    reverse_proxy @grpc h2c://127.0.0.1:8081

    @backend path /relay* /ws-proxy/* /api/* /oauth2/*
    reverse_proxy @backend 127.0.0.1:8081

    reverse_proxy /* 127.0.0.1:8080
}
```

gRPC, for management and signal, goes to the server over HTTP/2 cleartext; the relay, the
WebSocket proxy, the REST API and the IdP go to the server; everything else goes to the
dashboard.

## Firewall

With `netbird_server_manage_firewall` on, the role adds two ufw rules, tagged
`specsnl.specsops.netbird_server`:

- `allow <stun>/udp`;
- `route allow <stun>/udp`: the published port is DNATed into the container, so the
  packets are forwarded, and ufw drops forwarded traffic by default.

Rules for an old STUN port are removed on the next run. When ufw is not installed the
rules are skipped with a warning. Confirm on the host that STUN answers from outside; if
the forwarded traffic is still dropped, `Network=host` on the server is the fallback.

The HTTPS port is Caddy's: the `caddy` role opens 80 and 443.

## Bootstrap

A fresh server has no users. `GET /api/instance` reports `{"setup_required":true}`, and
`POST /api/setup` creates the owner, a local user of the embedded IdP. With
`netbird_server_setup_pat_enabled`, `create_pat` also returns a personal access token for
the rest of the bootstrap through the API:

```bash
curl -X POST https://netbird.example.com/api/setup \
  -H 'Content-Type: application/json' \
  -d '{"email": "admin@example.com", "password": "…", "name": "Admin", "create_pat": true}'
```

The token is valid for a day unless `pat_expire_in` sets more days. Setup only works once:
afterwards the endpoint answers `412 setup already completed`. Turn
`netbird_server_setup_pat_enabled` off again once the bootstrap is done.

## Break-glass

When no one can sign in, e.g. because the external identity provider is down, change the
local owner's password from the host, e.g. over the UpCloud console:

```bash
sudo podman exec -i netbird-server /go/bin/netbird-server admin user change-password \
  --email admin@example.com --password-file - --config /etc/netbird/config.yaml
```

It reads the new password from standard input; the password needs at least one uppercase
letter. `admin user reset-mfa` clears the user's MFA enrolment the same way.

## Containers

In a container the role writes the directories, config, Quadlets and site, but does not
start NetBird, reload Caddy or touch ufw.

## Example

```yaml
- hosts: netbird
  become: true
  roles:
    - specsnl.specsops.firewall
    - role: specsnl.specsops.block_volume
      vars:
        block_volume_devices:
          - storage_uuid: "{{ data_volume_uuid }}"
            label: data
            mount_point: /srv/data
    - specsnl.specsops.podman
    - specsnl.specsops.caddy
    - role: specsnl.specsops.netbird_server
      vars:
        netbird_server_version: 0.81.5
        netbird_server_dashboard_version: v2.96.0
        netbird_server_domain: netbird.example.com
        netbird_server_data_dir: /srv/data/netbird
        netbird_server_auth_secret: "{{ vault_netbird_auth_secret }}"
        netbird_server_store_encryption_key: "{{ vault_netbird_store_encryption_key }}"
        netbird_server_session_cookie_key: "{{ vault_netbird_session_cookie_key }}"
```

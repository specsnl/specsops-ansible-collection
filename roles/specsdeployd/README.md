# specsnl.specsops.specsdeployd

Scaffolding for [`specsdeployd`](https://github.com/specsnl/specsdeployd), the deploy
webhook receiver of the Specs golden images: a system user, an empty config directory,
a two-command sudoers drop-in and a systemd unit. The binary is installed only when you
ask for it — the default lays down everything *around* it and leaves the image
binary-free.

## Variables

| Variable                        | Default                                                     | Description                                    |
|---------------------------------|-------------------------------------------------------------|------------------------------------------------|
| `specsdeployd_user`             | `specsdeployd`                                              | System user the unit runs as                   |
| `specsdeployd_listen`           | `127.0.0.1:9000`                                            | Listen address baked into the unit             |
| `specsdeployd_binary_path`      | `/usr/bin/specsdeployd`                                     | Target of `ConditionPathExists`                |
| `specsdeployd_config_dir`       | `/etc/specsdeployd`                                         | Config directory, created empty                |
| `specsdeployd_version`          | `""`                                                        | Release to install; empty means do not install |
| `specsdeployd_release_base_url` | `https://github.com/specsnl/specsdeployd/releases/download` | Override for a mirror or an air-gapped build   |

## The unit ships enabled and dormant

The golden image bakes the user, the sudoers rule and the unit, but not the binary.
`ConditionPathExists={{ specsdeployd_binary_path }}` is what makes that safe: systemd
skips the start rather than failing it, so an image can carry an enabled unit without
collecting a failed one on every boot.

| State                   | `specsdeployd.service` | Starting it                                   |
|-------------------------|------------------------|-----------------------------------------------|
| No binary (the default) | `enabled`, inactive    | no-op, `ConditionResult=no`, `Result=success` |
| Binary installed        | `enabled`, inactive    | runs the receiver                             |

The role **enables** the unit; it never starts it. A host that has the binary brings the
receiver up at the next boot, or with an explicit `systemctl start`.

## Installing the binary

`specsdeployd_version` is empty by default, which is the golden-image behaviour: a Packer
run bakes no binary. Set it — ansible-pull does, at runtime — and the role fetches that
release's `.deb` and installs it with `apt`:

```yaml
- role: specsnl.specsops.specsdeployd
  vars:
    specsdeployd_version: "0.1.0"
```

The version carries no leading `v`. The `.deb` name is derived from it and from the
host's architecture (`x86_64` → `amd64`, `aarch64` → `arm64`), and the download is
checked against the release's `checksums.txt` — `get_url` matches that file on the
downloaded basename, which is why the package keeps the release asset's filename. dpkg
then owns the binary, so a re-run reports a change only when the version actually
differs.

The package ships **only** `/usr/bin/specsdeployd`. The user, sudoers rule, config
directory and unit belong to this role — if the package also shipped a unit, the two
would fight and the package would quietly win on upgrade.

`specsdeployd_binary_path` and the package have to keep naming the same path. If they
drift, the install succeeds, `ConditionPathExists` never fires, and the unit sits dormant
— indistinguishable from the intended uninstalled state. The role's molecule scenario
asserts the path on a host that installed the release, so the drift fails a test instead
of a deployment.

## Privileges

The receiver runs as an unprivileged user with `ProtectHome=true` and `PrivateTmp=true`,
and escalates through `/etc/sudoers.d/specsdeployd` for exactly two commands:

```sudoers
specsdeployd ALL=(root) NOPASSWD: /usr/bin/podman pull *
specsdeployd ALL=(root) NOPASSWD: /usr/bin/systemctl restart app-*.service
```

The file is written through `visudo -cf`, so a malformed drop-in can never land and take
`sudo` down with it. The unit deliberately does **not** set `NoNewPrivileges`, which
would block both escalations.

## Out of scope

No `config.json`, no webhook secrets, no Caddy route. Per-host configuration is laid down
at runtime by ansible-pull from `specsnl/specsops-ansible`; this role only creates the
directory it goes in.

## Example

The golden-image default — scaffolding, no binary:

```yaml
- hosts: app
  become: true
  roles:
    - specsnl.specsops.base
    - specsnl.specsops.podman
    - specsnl.specsops.specsdeployd
```

A host that should actually run the receiver:

```yaml
- hosts: app
  become: true
  roles:
    - role: specsnl.specsops.specsdeployd
      vars:
        specsdeployd_version: "0.1.0"
```

# specsnl.specsops.podman

Installs Podman on Ubuntu 26.04 from the distribution's `universe` repository.

## Variables

| Variable          | Default    | Description         |
|-------------------|------------|---------------------|
| `podman_packages` | `[podman]` | Packages to install |

Resolute ships Podman 5.7.x, which already bundles the Quadlet generator, so no extra
apt repository is needed. Add `podman-compose` or `slirp4netns` to
`podman_packages` rather than editing the role.

## Out of scope

The role ships no Quadlet unit files. Unit definitions are per-environment and are
laid down at runtime by ansible-pull, not baked into an image.

The role also does not enable `podman.socket` or `podman-auto-update.timer`.
Quadlet-managed containers are plain systemd units and need neither.

## Example

```yaml
- hosts: app
  become: true
  roles:
    - specsnl.specsops.base
    - specsnl.specsops.podman
```

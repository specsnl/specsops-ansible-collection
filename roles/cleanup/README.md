# specsnl.specsops.cleanup

Build-time image cleanup: `apt autoremove`, `apt clean`, and wipes the contents of
temp directories. Intended as the final role in a Packer image build to reduce image
size. Not intended for runtime ansible-pull.

## Mounts that never reach the image are skipped

An entry of `cleanup_paths` is left alone when it is a mount whose contents do not end
up in the image:

- a **tmpfs**, anywhere — a VM gets a fresh one on boot;
- **any mount in a container** — `docker commit` captures neither tmpfs nor volumes.
  The systemd-capable build images declare `/tmp` a volume, so there it is not a
  tmpfs at all.

A path on a VM that sits on its own disk partition is still wiped, since that
partition is part of the template.

Wiping these saves nothing, and it breaks the run when Ansible executes on the target
itself (Packer's `ansible-local`, or `connection: local`): Python keeps its
multiprocessing scratch directory in `/tmp`, and deleting it mid-run makes the
interpreter's exit handler traceback. Set `cleanup_skip_ephemeral: false` to wipe them
anyway.

## Variables

| Variable                 | Default                                | Description                                   |
|--------------------------|----------------------------------------|-----------------------------------------------|
| `cleanup_paths`          | `[/tmp, /var/tmp, /var/lib/apt/lists]` | Directories to wipe                           |
| `cleanup_skip_ephemeral` | `true`                                 | Leave mounts that never reach the image alone |

## Example

```yaml
# postgres/playbook.yml (Packer)
- hosts: all
  become: true
  roles:
    - specsnl.specsops.base
    - specsnl.specsops.postgresql
    - specsnl.specsops.cleanup   # always last
```

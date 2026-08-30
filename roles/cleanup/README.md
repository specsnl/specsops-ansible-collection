# specsnl.specsops.cleanup

Build-time image cleanup: `apt autoremove`, `apt clean`, and wipes the contents of
temp directories. Intended as the final role in a Packer image build to reduce image
size. Not intended for runtime ansible-pull.

## Variables

| Variable        | Default                                | Description         |
|-----------------|----------------------------------------|---------------------|
| `cleanup_paths` | `[/tmp, /var/tmp, /var/lib/apt/lists]` | Directories to wipe |

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

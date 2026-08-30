# specsnl.specsops.swap

Swap file provisioning for Ubuntu 24.04: creates a non-sparse swap file, formats it,
activates it, persists it in `/etc/fstab`, and applies the related sysctl tuning.

The `fstab` entry is written last, on purpose: a file that failed to format or
activate must never be left referenced there, or `swap.target` fails on every boot.

The `base` role sets `vm.swappiness` but nothing there ever creates swap, so on a
cloud VM without a swap area that tunable does nothing. This role is what makes it
meaningful.

## When the role does nothing

- In a container — `swapon` cannot work there.
- When the host already has active swap and `swap_file_path` does not exist. A VM
  image that ships its own swap partition is left alone rather than given a second
  swap area on top.

The sysctl drop-in is still written in both cases, gated only on `swap_enabled`.

## Variables

| Variable                  | Default                      | Description             |
|---------------------------|------------------------------|-------------------------|
| `swap_enabled`            | `true`                       | Manage swap at all      |
| `swap_file_path`          | `/swapfile`                  | Path to the swap file   |
| `swap_size`               | `1G`                         | Swap file size          |
| `swap_sysctl_file`        | `/etc/sysctl.d/99-swap.conf` | sysctl drop-in path     |
| `swap_swappiness`         | `"30"`                       | `vm.swappiness`         |
| `swap_vfs_cache_pressure` | `"50"`                       | `vm.vfs_cache_pressure` |

`swap_swappiness` overlaps with `base_sysctl.vm.swappiness` (default `"10"`). Both
land in `/etc/sysctl.d/`, and the later-sorting file wins. Set it in one role or the
other — if you adopt this role, drop `vm.swappiness` from `base_sysctl`.

## Example

```yaml
- hosts: all
  become: true
  roles:
    - specsnl.specsops.base
    - specsnl.specsops.swap
```

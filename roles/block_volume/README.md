# specsnl.specsops.block_volume

Mounts UpCloud block storage volumes for stateful hosts. A volume outlives the server it
is attached to, so a VM rebuilt from an image mounts the same data again. For each volume
the role:

1. Finds the disk by its UpCloud storage UUID under `/dev/disk/by-id/`, and fails when
   it is not there.
2. Creates a filesystem on it only when it has none. A disk that already holds one is
   never reformatted.
3. Sets the filesystem label if it is missing, and mounts the volume as `LABEL=<label>`.
4. Adds a `RequiresMountsFor=<mount_point>` drop-in to each unit in `required_by`, so
   those units only start with the volume mounted.

Device names like `/dev/vdb` depend on attach order, so the role uses neither them nor
the filesystem UUID in `/etc/fstab`.

## Finding the disk

UpCloud passes the storage UUID to the guest as the virtio-blk serial. A virtio-blk
serial holds at most 20 bytes, so udev links the disk as
`/dev/disk/by-id/virtio-<first 20 characters of the UUID without dashes>`:

| Storage UUID                           | by-id link                                    |
|----------------------------------------|-----------------------------------------------|
| `01456089-b1c4-4d5e-9708-4e1c9e0dbb39` | `/dev/disk/by-id/virtio-01456089b1c44d5e9708` |

That pair was checked on an Ubuntu 26.04 VM in `nl-ams1`: the storage at `virtio:0`
showed up with that link, and `/sys/block/vda/serial` held the same 20 characters.
UpCloud's CSI driver finds volumes the same way (`volumeIDToDiskID` in
[UpCloudLtd/upcloud-csi](https://github.com/UpCloudLtd/upcloud-csi)). The link only
exists for a disk on the `virtio` bus, which is UpCloud's default. When the link is
missing the role fails and lists what `/dev/disk/by-id/` holds instead.

To check a host by hand:

```bash
ls -l /dev/disk/by-id/
cat /sys/block/vdb/serial
```

## When the role fails

The role stops rather than guess when:

- the disk is not attached, or not on the virtio bus;
- the disk holds a partition table. Only a whole, unpartitioned disk is formatted;
- the disk holds a filesystem of another type than `fstype`;
- the filesystem carries another label than `label`, which suggests the wrong disk;
- another disk carries the same label, so `LABEL=<label>` would not name one disk.

## No `nofail`

The fstab entry has no `nofail`, on purpose. If the volume is missing, the dependent
service must not start and write to the OS disk underneath the empty mount point.

The catch is that a missing volume also stops the boot. systemd gives up on the mount
after its device timeout of 90 seconds and drops to emergency mode, so the host is not
reachable over SSH until the volume is attached again. Use the UpCloud console to get
in. Setting `mount_opts` with `nofail` trades this for a host that boots without its
data; the `required_by` units still stay down.

## Containers

A container cannot attach or mount a block device. There the role skips finding,
formatting and mounting. It only writes the fstab entry and the drop-ins, which is what
the Molecule scenario verifies. Formatting and mounting have to be verified on a VM.
Like the other roles, it detects the container itself unless `block_volume_in_container`
is set.

## Variables

| Variable                  | Default            | Description                                     |
|---------------------------|--------------------|-------------------------------------------------|
| `block_volume_devices`    | `[]`               | Volumes to set up, see below                    |
| `block_volume_fstype`     | `ext4`             | Filesystem for entries that don't set `fstype`  |
| `block_volume_mount_opts` | `defaults,noatime` | Options for entries that don't set `mount_opts` |

Each `block_volume_devices` entry:

| Key            | Required | Description                                                                        |
|----------------|----------|------------------------------------------------------------------------------------|
| `storage_uuid` | yes      | UpCloud storage UUID of the attached disk                                          |
| `label`        | yes      | Filesystem label: letters, digits, `-` and `_`, at most 16 characters (12 for xfs) |
| `mount_point`  | yes      | Absolute path to mount on, created when missing                                    |
| `fstype`       | no       | `ext4`, `ext3`, `ext2` or `xfs`                                                    |
| `mount_opts`   | no       | Mount options                                                                      |
| `owner`        | no       | Owner of the mounted filesystem's root, left alone when unset                      |
| `group`        | no       | Group of the mounted filesystem's root, left alone when unset                      |
| `mode`         | no       | Mode of the mounted filesystem's root, left alone when unset                       |
| `required_by`  | no       | systemd units that require the mount. A name without a unit type is a service      |

Drop-ins are written to `/etc/systemd/system/<unit>.d/block-volume-<label>.conf`. Removing
a unit from `required_by` does not delete its drop-in.

## Example

A PostgreSQL host with its cluster on a volume. `block_volume` runs before `postgresql`,
and the data directory is a subdirectory of the mount point, because a fresh ext4
filesystem holds `lost+found`.

```yaml
- hosts: db
  become: true
  roles:
    - role: specsnl.specsops.block_volume
      vars:
        block_volume_devices:
          - storage_uuid: 01d4fcd4-e446-433b-8a9c-551a1284952e
            label: pgdata
            mount_point: /srv/pgdata
            required_by:
              - postgresql@18-main
    - role: specsnl.specsops.postgresql
      vars:
        postgresql_data_directory: /srv/pgdata/main
```

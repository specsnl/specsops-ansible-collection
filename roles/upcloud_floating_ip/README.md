# specsnl.specsops.upcloud_floating_ip

Holds UpCloud floating IPs on the server they are assigned to. UpCloud routes a floating
IP to the MAC address of a server's public interface, but the OS still has to configure
the address itself. With this role on every node, moving the IP to a replacement server
needs no DNS change: re-run Ansible on both, the new server takes the address and the old
one drops it.

The role:

1. Reads the UpCloud metadata service at `http://169.254.169.254/metadata/v1.json`. The
   server needs Metadata enabled.
2. Takes the public interface that has an IPv4 address from the metadata, and finds it on
   the host by its MAC address. Interface names depend on the image and are not used.
3. Writes a netplan drop-in that adds each floating IP to that interface as a `/32`, and
   runs `netplan generate`, then `netplan apply`.
4. Removes the drop-in and applies again when no floating IP is assigned. This is what the
   old server does after its IP moved.

## The metadata

The floating IPs are the entries with `"floating": true` among the `ip_addresses` of the
public interface:

```json
{
  "network": {
    "interfaces": [
      {
        "index": 1,
        "type": "public",
        "mac": "ee:1b:db:ca:61:ee",
        "ip_addresses": [
          { "address": "94.237.105.53", "family": "IPv4", "floating": false, "dhcp": true },
          { "address": "94.237.105.50", "family": "IPv4", "floating": true, "dhcp": false }
        ]
      }
    ]
  }
}
```

This shape comes from the fixture of cloud-init's UpCloud datasource and has not been
checked on a VM yet. To check a host by hand:

```bash
curl -s http://169.254.169.254/metadata/v1.json | jq '.network.interfaces'
```

UpCloud gives public IPv4 and public IPv6 their own interface by default, so the role
picks the public interface with an IPv4 address and fails when there is not exactly one.
Only IPv4 floating IPs are handled.

## The drop-in

`/etc/netplan/60-upcloud-floating-ip.yaml`, mode `0600`:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      match:
        macaddress: "ee:1b:db:ca:61:ee"
      addresses:
        - "94.237.105.50/32"
```

The drop-in only adds addresses. DHCP, routes and DNS for the interface come from the file
that already configures it, such as cloud-init's `50-cloud-init.yaml`. The
drop-in is keyed by the interface name, as cloud-init keys its own definition, so netplan
merges the addresses into it rather than seeing two definitions of one interface.

The handlers run `netplan generate` before `netplan apply`, so a drop-in netplan rejects
fails the play before the live network is touched. They run at the end of the role, so
roles after it can bind to the floating IP.

## Containers

The role does not run netplan in a container. It still reads the metadata and writes or
removes the drop-in, which is what the Molecule scenario verifies against fixture
metadata. That netplan applies and drops the address has to be verified on a VM. Like the
other roles, it detects the container itself unless `upcloud_floating_ip_in_container` is
set.

## Variables

| Variable                           | Default                                    | Description                                         |
|------------------------------------|--------------------------------------------|-----------------------------------------------------|
| `upcloud_floating_ip_addresses`    | `[]`                                       | IPv4 floating IPs; empty takes them from metadata   |
| `upcloud_floating_ip_netplan_path` | `/etc/netplan/60-upcloud-floating-ip.yaml` | Netplan drop-in to write or remove                  |
| `upcloud_floating_ip_metadata_url` | `http://169.254.169.254/metadata/v1.json`  | Metadata service, only overridden to test a fixture |

Leave `upcloud_floating_ip_addresses` empty so the metadata decides which server holds the
IP. Setting it pins the addresses on every host it applies to, whatever UpCloud routes
there. The metadata is read either way, for the interface's MAC address.

## Example

On every app node, so whichever one the IP is assigned to holds it:

```yaml
- hosts: app
  become: true
  roles:
    - role: specsnl.specsops.upcloud_floating_ip
```

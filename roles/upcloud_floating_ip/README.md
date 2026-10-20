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

Under `--check` the role still reads the metadata, so it reports the drop-in it would
write or remove without touching it.

## The metadata

The floating IPs are the entries with `"floating": true` among the `ip_addresses` of the
public interface. This is the public interface of an UpCloud VM with a floating IP:

```json
{
  "network": {
    "interfaces": [
      {
        "index": 1,
        "ip_addresses": [
          { "address": "94.237.109.121", "dhcp": true, "dns": ["94.237.127.9", "94.237.40.9"],
            "family": "IPv4", "floating": false, "gateway": "94.237.108.1", "network": "94.237.108.0/22" },
          { "address": "5.22.211.179", "dhcp": false, "family": "IPv4", "floating": true }
        ],
        "mac": "e6:42:09:32:5e:9b",
        "network_id": "03d8bd26-3862-42cf-8a86-88bec05df938",
        "type": "public"
      }
    ]
  }
}
```

The second entry is abridged: only its `address` and `floating` were captured. The role
uses the interface's `type` and `mac`, and the `family`, `floating` and `address` of its
entries; `dhcp`, `dns`, `gateway`, `network` and `network_id` are not used. To check a
host by hand:

```bash
curl -s http://169.254.169.254/metadata/v1.json | jq '.network.interfaces'
```

UpCloud gives public IPv4 and public IPv6 their own interface by default, so the role
picks the public interface with an IPv4 address and fails when there is not exactly one.
Only IPv4 floating IPs are handled.

## Stale metadata

UpCloud does not update a server's metadata when a floating IP is assigned to it or moved
away from it. Until Metadata is turned off and on again, the role sees the old
assignment: the new server does not take the IP, and the old one keeps it.

After moving the IP, refresh the metadata on both the new and the old server, then run
the role:

```bash
upctl server modify <uuid> --disable-metadata
upctl server modify <uuid> --enable-metadata
```

Through the API, that is `PUT /1.3/server/{uuid}` with `"metadata": "no"`, then
`"metadata": "yes"`. UpCloud describes this in
[its tutorial on Terraform redeployments with a floating IP](https://upcloud.com/resources/tutorials/reduce-downtime-terraform-redeployments).

## The drop-in

`/etc/netplan/60-upcloud-floating-ip.yaml`, mode `0600`:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      match:
        macaddress: "e6:42:09:32:5e:9b"
      addresses:
        - "5.22.211.179/32"
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

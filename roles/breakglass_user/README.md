# specsnl.specsops.breakglass_user

A console-only login for when no network path into the host is left. Once NetBird is the
only admin door, a broken NetBird locks everyone out: reopening port 22 in OpenTofu does not
help, because ufw closes it too, and changing ufw needs SSH. The way in is then the
provider's web console, e.g. UpCloud's, which needs a user that logs in with a password.
The regular login user is key-only.

The role creates that user, in `sudo`, with a password given as a hash, no
`authorized_keys`, and an sshd drop-in that denies it over SSH, so the password only works
on the console.

## Variables

| Variable                           | Default                                     | Description                        |
|------------------------------------|---------------------------------------------|------------------------------------|
| `breakglass_user_name`             | `breakglass`                                | Name of the user                   |
| `breakglass_user_password_hash`    | —                                           | SHA-512 crypt hash of the password |
| `breakglass_user_sshd_dropin_path` | `/etc/ssh/sshd_config.d/00-breakglass.conf` | sshd drop-in with `DenyUsers`      |

`breakglass_user_password_hash` is required. Run the role on every host.

## The password

Only the hash reaches Ansible. Generate the password and its hash once, e.g.:

```bash
mkpasswd -m sha-512          # from the whois package
openssl passwd -6            # or this
```

Keep the plaintext encrypted to the break-glass recipient only (`age - break-glass`), which
the CI service account can read but not decrypt; the hash goes into the inventory's
secrets. The role fails on anything that is not a `$6$` hash, so a plaintext password never
lands in `/etc/shadow` as given, and the task that sets it runs with `no_log`.

Rotate the password after every use.

## Over SSH

The user has no `~/.ssh/authorized_keys` (or `authorized_keys2`); any that exist are
removed. `breakglass_user_sshd_dropin_path` holds `DenyUsers <name>`, so sshd refuses the
user even if password login is ever turned on, e.g. by cloud-init's `50-cloud-init.conf`.
Unlike `PasswordAuthentication`, `DenyUsers` adds to a list from every file instead of
taking the first value, so the drop-in's number does not matter.

The drop-in is validated with `sshd -t` and sshd is reloaded when sshd is installed. On a
host without it, the drop-in is written for later.

## On the console

`hardening` changes only sshd and fail2ban, whose jail watches sshd. A console login goes
through getty and the `login` PAM stack, which neither role touches, so the user can log
in there with its password. Molecule checks this by authenticating against the `login`
PAM service after running `hardening`.

The user is in `sudo` and gives the same password to `sudo`. A typical repair:

```bash
# On the UpCloud console, logged in as breakglass
sudo -i
systemctl status netbird
journalctl -u netbird --since -1h
```

## Containers

In a container the role creates the user and writes the drop-in, but does not reload sshd.

## Example

```yaml
- hosts: all
  become: true
  roles:
    - specsnl.specsops.hardening
    - role: specsnl.specsops.breakglass_user
      vars:
        breakglass_user_password_hash: "{{ vault_breakglass_password_hash }}"
```

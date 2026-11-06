# circuit-users

[![Conn3x grade](https://conn3x.org/circuits/users/grade.svg)](https://conn3x.org/circuits/users/grade)

A [Conn3x](https://conn3x.org) circuit that manages local accounts from one map: the users, the groups they belong to, and the SSH keys they log in with. A user can also get passwordless sudo, through a drop-in file in `/etc/sudoers.d`.

## Usage

```hcl
module "team" {
  source = "github.com/conn3xorg/circuit-users?ref=v0.1.3"

  hosts = {
    web1 = "203.0.113.10"
    web2 = "203.0.113.11"
  }

  ssh_private_key_file = "~/.ssh/id_ed25519"

  users = {
    alice = {
      groups = ["developers"]
      keys   = [file("${path.module}/keys/alice.pub")]
      sudo   = true
    }
    bob = {
      groups = ["developers"]
      keys = [
        "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIKcIIzM1rn1/TLenZvLoOhYMUDH09FQbF79aH8EBw8CM bob@laptop",
      ]
    }
    backup = {
      uid   = 990
      shell = "/usr/sbin/nologin"
    }
  }
}
```

Then run `conn3x init` and `conn3x apply`. A second `conn3x plan` says `No changes.` To try it on a throwaway container first, see [examples/local](examples/local).

### Hosts with pinned host keys

A value in `hosts` can also be a host object, as in a circuit's own [`hosts`](https://docs.conn3x.org/reference/circuits/hosts#host-object-fields). Use this to hand over servers created in the same plan, with the host key you gave them:

```hcl
module "team" {
  source = "github.com/conn3xorg/circuit-users?ref=v0.1.3"

  hosts = {
    for name, d in digitalocean_droplet.web : name => {
      address  = d.ipv4_address
      host_key = tls_private_key.host[name].public_key_openssh
    }
  }

  users = {
    alice = { keys = [file("${path.module}/keys/alice.pub")], sudo = true }
  }
}
```

A host object can set `address`, `user`, `port`, `private_key_file`, `host_key` and `jump_hosts`, and one map can mix addresses and host objects.

## What it does

On every host in `hosts`, in this order:

1. Creates every group that a user lists in `groups`. Groups that already exist, such as `wheel` or `sudo`, are left as they are.
2. Creates each user, or brings an existing one in line with the attributes you set.
3. Adds each key in `keys` to the user's `~/.ssh/authorized_keys`. Other keys in the file stay. A key is matched by its type and data, so the same key listed twice with different comments or options is added once, and dropping one copy leaves it.
4. For each user with `sudo = true`, installs sudo and writes `/etc/sudoers.d/NAME`, mode `0440`, with the line `NAME ALL=(ALL) NOPASSWD: ALL`.

Removing things from the map:

- **A key** is removed from `authorized_keys`, so a rotated key stops working.
- **`sudo = true`** turned off, or the user removed, deletes the user's sudoers file.
- **A user** is no longer managed, but the account stays on the host. Delete it by hand, or with a `user` step with `on_remove = "delete"`.
- **A group** that no user lists any more stays on the host.

Conn3x has no way yet to check a file with `visudo` before writing it. The sudoers line is built only from the user's name, and the `users` variable rejects names that aren't valid login names, so the file can't hold a syntax error.

## Inputs

| Name | Type | Default | Description |
| --- | --- | --- | --- |
| `hosts` | map | Required | The hosts to configure. Each key is a host's name. Each value is its address, such as `"203.0.113.10"`, or a host object such as `{ address = "203.0.113.10", host_key = "ssh-ed25519 AAAA..." }`. |
| `users` | map(object) | Required | The accounts to manage, keyed by login name. At least one. See [users](#users). |
| `ssh_user` | string | `"root"` | The user to log in as. |
| `ssh_port` | number | `22` | The SSH port. |
| `ssh_private_key_file` | string | `null` | Path to a private key file. When null, Conn3x uses the keys in the SSH agent. |
| `ssh_known_hosts_file` | string | `null` | The known_hosts file to check host keys against. When null, Conn3x uses `~/.ssh/known_hosts`. Ignored for hosts with a `host_key`. |
| `ssh_become` | bool | `false` | Run every command as root through sudo. Set this when `ssh_user` isn't root. |

### users

Each entry's key is the login name. Every field is optional, so `{ carol = {} }` creates `carol` with a bash shell.

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `uid` | number | `null` | User ID. When null, the host picks one and Conn3x keeps it. |
| `shell` | string | `"/bin/bash"` | Login shell. |
| `home` | string | `null` | Home directory. When null, the host's default, usually `/home/NAME`. |
| `groups` | list(string) | `null` | Supplementary groups, as the complete list: the user is taken out of any group not in it. When null, the user's groups are left as they are. |
| `keys` | list(string) | `[]` | Public keys, in authorized_keys format, that can log in as the user. |
| `sudo` | bool | `false` | Let the user run any command as root through sudo, without a password. |

Login and group names start with a lowercase letter or `_`, followed by up to 31 lowercase letters, digits, `-` or `_`.

## Outputs

None.

## Requirements

- Conn3x v0.6.4 or later.
- Hosts running Rocky Linux, RHEL, AlmaLinux, Fedora, Debian or Ubuntu, with systemd.

## Tested on

Rocky Linux 10, Fedora 45, Ubuntu 24.04 and Debian 13. CI applies [examples/local](examples/local) to each on amd64, checks that the second plan has no changes, logs in as a user with the key and uses sudo, and checks a user with no key and no sudo can do neither.

It was also applied by hand, with six other circuits, to an Ubuntu 24.04 droplet and a Rocky Linux 10 droplet with SELinux enforcing. On both, `alice` was in `developers` and listed by `sudo -l`, and `bob` got `/bin/sh` as the login shell. After a reboot the plan had no changes and the audit log had no SELinux denials. The same checks passed on Fedora 44, with SELinux enforcing, and Debian 13 droplets, before and after a reboot.

## License

[MPL-2.0](LICENSE)

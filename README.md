# sleif.ssh_tunnel

This role creates a systemd based outgoing (forward, default) or reverse ssh tunnel.

**Security Note:** During the playbook run the involved ssh host keys of the remote and maybe the jump host are added without being asked. Please verify the keys in the known_hosts manually.

## Role Variables

### Required variables

- ssh_tunnel_name
- ssh_tunnel_remote_host # FQDN for outgoing end-point
- ssh_tunnel_local_port # locally listen port
- ssh_tunnel_remote_port # outgoing end-point

### Optional Variables

- ssh_tunnel_mode # -L (forward) or -R (reverse) - defaults to '-L'; '-R' for lower ports (<1024) only possible as user root
- ssh_tunnel_jump_host # FQDN of possibly jump host
- ssh_tunnel_local_interface # locally listen interface ip; only useable for host local ip - defaults to none (-> 0.0.0.0)
- ssh_tunnel_remote_host_ip # IP address of the end-point host in case of a jump host
- ssh_tunnel_remote_interface # outgoing end-point interface ip
- ssh_tunnel_target_host_ip # IP address of the target endpoint, if not local ip (= ssh_local_interface) - defaults to none
- ssh_tunnel_user_jump # restricted user - account will be created - defaults to ssh_tunnel_user_remote
- ssh_tunnel_user_local # restricted user - account will be created
- ssh_tunnel_user_remote # restricted user - account will be created
- ssh_tunnel_user_target # restricted user - account will be created (if set and target_host not local)
- storage_dir_base # local storage base

## Usage

### Forward tunnel from host local ip to remote

Use required variables as above. Jump host is taken care of connecting to remote, if required.
A listening port (ssh_tunnel_local_port) lower than 1024 requires ssh_tunnel_user_local to be root.

```
       -----------------------------------
       | [0.0.0.0]:ssh_tunnel_local_port |
       -----------------------------------

                       |
                       V

-------------------------------------------------
| ssh_tunnel_remote_host:ssh_tunnel_remote_port |
-------------------------------------------------
```

and

```
        -----------------------------------
        | [0.0.0.0]:ssh_tunnel_local_port |
        -----------------------------------

                        |
                        V

             ------------------------
             | ssh_tunnel_jump_host |
             ------------------------

                        |
                        V

-------------------------------------------------
| ssh_tunnel_remote_host:ssh_tunnel_remote_port |
-------------------------------------------------
```

With added users:

```
       ----------------------------------------------------------
       |  ssh_tunnel_user_local@[0.0.0.0]:ssh_tunnel_local_port |
       ----------------------------------------------------------

                                   |
                                   V

             ---------------------------------------------
             | ssh_tunnel_user_jump@ssh_tunnel_jump_host |
             ---------------------------------------------

                                   |
                                   V

--------------------------------------------------------------------------
|  ssh_tunnel_user_remote@ssh_tunnel_remote_host:ssh_tunnel_remote_port  |
--------------------------------------------------------------------------
```

Combine as and if needed, also for (ssh_tunnel_user_jump)


### Forward tunnel from target host to remote

Host local ssh process is intermediary, only, listening host is "target".

Set ssh_tunnel_target_host_ip as listening host. ssh_tunnel_local_interface is disabled and ssh_tunnel_local_port is the listening port on ssh_tunnel_target_host_ip. Using a different target host, replacing host local endpoint. If ssh_tunnel_target_host_ip is set ssh_tunnel_local_interface is being ignored. Further, ssh_tunnel_user_target is taken care of.

### Reverse tunnel from remote to host local

Listening port (ssh_tunnel_remote_port) on remote (ssh_tunnel_remote_host) is being forwarded to host local (ssh_tunnel_local_interface) port (ssh_tunnel_local_port). Jump host to connect to remote is taken care of, if defined (default: none). A listening port lower than 1024 requires ssh_tunnel_user_remote to be root. ssh_tunnel_mode needs to be set to '-R'

### Reverse tunnel from remote to target

Listening port on remote is forwarded to target (ssh_tunnel_target_host_ip) port (ssh_tunnel_target_port). Jump host to connect to remote is taken care of, if defined (default: none). A listening port lower than 1024 requires ssh_tunnel_user_remote to be root.


## Examples

### Without a jump host

```yml
- name: include_role sleif.ssh_tunnel
  ansible.builtin.include_role:
    name: sleif.ssh_tunnel
    apply:
      tags: "ssh_tunnel, ssh_tunnel_nsupdate_example_de"
  vars:
    ssh_tunnel_name: "nsupdate_example_de"
    ssh_tunnel_remote_host: "dns.example.de"
    ssh_tunnel_user_remote: "ssh-tunnel-user" # optional
    ssh_tunnel_local_port: "1053"
    ssh_tunnel_remote_port: "53"
    # ssh_tunnel_local_interface: "0.0.0.0" # local listen interface, default 0.0.0.0
    # ssh_tunnel_remote_interface: "localhost" # default localhost
  tags: "ssh_tunnel, ssh_tunnel_nsupdate_example_de"
```

### With a jump host

- To use this role with a jump host, it's necessary to configure the remote host connection parameter in the inventory:

```yml
  hosts:
    all-dns:
      ansible_host: 192.168.1.10
      ansible_ssh_common_args: '-o ProxyJump=root@jumphost.example.de'
```

```yml
- name: include_role sleif.ssh_tunnel
  ansible.builtin.include_role:
    name: sleif.ssh_tunnel
    apply:
      tags: "ssh_tunnel, ssh_tunnel_nsupdate_example_de"
  vars:
    ssh_tunnel_name: "nsupdate_example_de"
    ssh_tunnel_jump_host: "jumphost.example.de"
    ssh_tunnel_remote_host: "dns-behind-jumphost.example.de"
    ssh_tunnel_remote_host_ip: "192.168.1.10" # required in case of a jump host
    ssh_tunnel_user_local: "ssh-tunnel-user" # optional
    ssh_tunnel_user_remote: "ssh-tunnel-user" # optional
    ssh_tunnel_local_port: "1053"
    ssh_tunnel_remote_port: "53"
    # ssh_tunnel_local_interface: "0.0.0.0" # local listen interface, default 0.0.0.0
    # ssh_tunnel_remote_interface: "localhost" # default localhost
  tags: "ssh_tunnel, ssh_tunnel_nsupdate_example_de"
```

## License

MIT

## Author Information

Created in 2024 by Sebastian Berthold

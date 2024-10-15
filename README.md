# sleif.ssh_tunnel

This role creates a systemd based outgoing (forward, default) or reverse ssh tunnel.

**Security Note:** During the playbook run the involved ssh host keys of the remote and maybe the jump host are added without being asked. Please verify the keys in the known_hosts manually.

## Role Variables

### Required variables

- ssh_tunnel_name
- ssh_tunnel_remote_host # FQDN for outgoing end-point
- ssh_tunnel_local_port # locally listen port; lower ports (<1024) only possible as user root
- ssh_tunnel_jump_host # FQDN of possibly jump host
- ssh_tunnel_remote_port # outgoing end-point

### Optional Variables

- ssh_tunnel_mode # -L (forward) or -R (reverse) - defaults to '-L'
- ssh_tunnel_local_interface # locally listen interface ip; only useable for host local ip - defaults to 0.0.0.0 (all interfaces)
- ssh_tunnel_remote_host_ip # IP address of the end-point host in case of a jump host
- ssh_tunnel_remote_interface # outgoing end-point interface ip - defaults to localhost
- ssh_tunnel_target_host_ip # IP address of the target endpoint, if not local ip (= ssh_local_interface) - defaults to none
- ssh_tunnel_target_bind_ip # IP of target local endpoint interface - defaults to 0.0.0.0 (all interfaces)
- ssh_tunnel_user_jump # restricted user - account will be created - defaults to ssh_tunnel_user_remote
- ssh_tunnel_user_local # restricted user - account will be created - defaults to root
- ssh_tunnel_user_remote # restricted user - account will be created - defaults to root
- ssh_tunnel_user_target # restricted user - account will be created (if set and ssh_tunnel_target_host_ip is set) - defaults to root
- storage_dir_base # local storage base

## Usage

### Definition

- **Host Local** - the host executing the ssh command
- **Remote** - the host ssh connects to via ssh standard port 22
- **Jump** - intermediary host used to connect from Host Local to Remote
- **Target** - the host connected to from Remote

Not covered (yet?):

- ssh connect port - only standard port 22
- extra options to the ssh process

### Forward tunnel from host local ip to remote

Use required variables as above. Jump host is taken care of connecting to remote, if required.
A listening port (*ssh_tunnel_local_port*) lower than 1024 requires *ssh_tunnel_user_local* to be root.

```ascii
       -----------------------------------
       | [0.0.0.0]:ssh_tunnel_local_port |
       -----------------------------------
                       |
                       V
-------------------------------------------------
| ssh_tunnel_remote_host:ssh_tunnel_remote_port |
-------------------------------------------------
```

SSH command generated:

```code
ssh root@ssh_tunnel_remote_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -L 0.0.0.0:ssh_tunnel_local_port:localhost:ssh_tunnel_remote_port
```

With a jump host:

```ascii
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

SSH command generated:

```code
ssh root@ssh_tunnel_remote_host_ip -J root@ssh_tunnel_jump_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -L 0.0.0.0:ssh_tunnel_local_port:localhost:ssh_tunnel_remote_port
```

With added users:

```ascii
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

SSH command generated - run as user-systemd unit (-> ssh_tunnel_user_local user):

```code
ssh ssh_tunnel_user_remote@ssh_tunnel_remote_host_ip -J ssh_tunnel_user_jump@ssh_tunnel_jump_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -L 0.0.0.0:ssh_tunnel_local_port:localhost:ssh_tunnel_remote_port
```

Combine as and if needed, also for *ssh_tunnel_user_jump*

### Forward tunnel from target host to remote

Host local ssh process is intermediary, only, listening host is "target".

Set *ssh_tunnel_target_host_ip* as listening host. *ssh_tunnel_local_interface* is disabled and ssh_tunnel_local_port is the listening port on *ssh_tunnel_target_host_ip*. Using a different target host, replacing host local endpoint. If *ssh_tunnel_target_host_ip* is set, then *ssh_tunnel_local_interface* is being ignored. Further, *ssh_tunnel_user_target* is taken care of. If you use a FQDN instead of IP as *ssh_tunnel_target_host_ip*, this address needs to be resolveable and reachable by ssh.

```ascii
---------------------------------------------------
| ssh_tunnel_target_host_ip:ssh_tunnel_local_port |
---------------------------------------------------
                       |
                       V
 -------------------------------------------------
 | ssh_tunnel_remote_host:ssh_tunnel_remote_port |
 -------------------------------------------------
```

SSH command generated:

```code
ssh root@ssh_tunnel_remote_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -L ssh_tunnel_target_host_ip:ssh_tunnel_local_port:localhost:ssh_tunnel_remote_port
```

With a jump host:

```ascii
---------------------------------------------------
| ssh_tunnel_target_host_ip:ssh_tunnel_local_port |
---------------------------------------------------
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

SSH command generated:

```code
ssh root@ssh_tunnel_remote_host_ip -J root@ssh_tunnel_jump_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -L ssh_tunnel_target_host_ip:ssh_tunnel_local_port:localhost:ssh_tunnel_remote_port
```

With added users:

```ascii
---------------------------------------------------------------------------
|  ssh_tunnel_user_target@ssh_tunnel_target_host_ip:ssh_tunnel_local_port |
---------------------------------------------------------------------------
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

SSH command generated - run as user-systemd unit (-> ssh_tunnel_user_local user):

```code
ssh ssh_tunnel_user_remote@ssh_tunnel_remote_host_ip -J ssh_tunnel_user_jump@ssh_tunnel_jump_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -L ssh_tunnel_user_target@ssh_tunnel_target_host_ip:ssh_tunnel_local_port:localhost:ssh_tunnel_remote_port
```

### Reverse tunnel from remote to host local

Listening port (*ssh_tunnel_remote_port*) on remote (*ssh_tunnel_remote_host*) is being forwarded to host local (*ssh_tunnel_local_interface*) port (*ssh_tunnel_local_port*). Jump host to connect to remote is taken care of, if defined (default: none). A listening port lower than 1024 requires *ssh_tunnel_user_remote* to be root. ssh_tunnel_mode needs to be set to '-R'

```ascii
-------------------------------------------------
| ssh_tunnel_remote_host:ssh_tunnel_remote_port |
-------------------------------------------------
                       |
                       V
       -----------------------------------
       | [0.0.0.0]:ssh_tunnel_local_port |
       -----------------------------------
```

SSH command generated:

```code
ssh root@ssh_tunnel_remote_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -R 0.0.0.0:ssh_tunnel_local_port:localhost:ssh_tunnel_remote_port
```

With a jump host:

```ascii
-------------------------------------------------
| ssh_tunnel_remote_host:ssh_tunnel_remote_port |
-------------------------------------------------
                        |
                        V
             ------------------------
             | ssh_tunnel_jump_host |
             ------------------------
                        |
                        V
        -----------------------------------
        | [0.0.0.0]:ssh_tunnel_local_port |
        -----------------------------------
```

SSH command generated:

```code
ssh root@ssh_tunnel_remote_host_ip -J root@ssh_tunnel_jump_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -R 0.0.0.0:ssh_tunnel_local_port:localhost:ssh_tunnel_remote_port
```

With added users:

```ascii
--------------------------------------------------------------------------
|  ssh_tunnel_user_remote@ssh_tunnel_remote_host:ssh_tunnel_remote_port  |
--------------------------------------------------------------------------
                                   |
                                   V
             ---------------------------------------------
             | ssh_tunnel_user_jump@ssh_tunnel_jump_host |
             ---------------------------------------------
                                   |
                                   V
       ----------------------------------------------------------
       |  ssh_tunnel_user_local@[0.0.0.0]:ssh_tunnel_local_port |
       ----------------------------------------------------------
```

SSH command generated - run as user-systemd unit (-> ssh_tunnel_user_local user):

```code
ssh ssh_tunnel_user_remote@ssh_tunnel_remote_host_ip -J ssh_tunnel_user_jump@ssh_tunnel_jump_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -R 0.0.0.0:ssh_tunnel_local_port:localhost:ssh_tunnel_remote_port
```

Combine as and if needed, also for (ssh_tunnel_user_jump).

### Reverse tunnel from remote to (non-local) target

Listening port (*ssh_tunnel_remote_port*) on remote (*ssh_tunnel_remote_host*) is forwarded to target (*ssh_tunnel_target_host_ip*) port (*ssh_tunnel_target_port*). Jump host to connect to remote is taken care of, if defined (default: none). A listening port lower than 1024 requires *ssh_tunnel_user_remote* to be root.

```ascii
 -------------------------------------------------
 | ssh_tunnel_remote_host:ssh_tunnel_remote_port |
 -------------------------------------------------
                       |
                       V
---------------------------------------------------
| ssh_tunnel_target_host_ip:ssh_tunnel_local_port |
---------------------------------------------------
```

SSH command generated:

```code
ssh root@ssh_tunnel_remote_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -R 0.0.0.0:ssh_tunnel_local_port:ssh_tunnel_target_host_ip:ssh_tunnel_remote_port
```

With a jump host:

```ascii
 -------------------------------------------------
 | ssh_tunnel_remote_host:ssh_tunnel_remote_port |
 -------------------------------------------------
                         |
                         V
              ------------------------
              | ssh_tunnel_jump_host |
              ------------------------
                         |
                         V
---------------------------------------------------
| ssh_tunnel_target_host_ip:ssh_tunnel_local_port |
---------------------------------------------------
```

SSH command generated:

```code
ssh root@ssh_tunnel_remote_host_ip -J root@ssh_tunnel_jump_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -R 0.0.0.0:ssh_tunnel_local_port:ssh_tunnel_target_host_ip:ssh_tunnel_remote_port
```

With added users:

```ascii
--------------------------------------------------------------------------
|  ssh_tunnel_user_remote@ssh_tunnel_remote_host:ssh_tunnel_remote_port  |
--------------------------------------------------------------------------
                                   |
                                   V
             ---------------------------------------------
             | ssh_tunnel_user_jump@ssh_tunnel_jump_host |
             ---------------------------------------------
                                   |
                                   V
       ----------------------------------------------------------
       |  ssh_tunnel_user_target@[0.0.0.0]:ssh_tunnel_local_port |
       ----------------------------------------------------------
```

SSH command generated - run as user-systemd unit (-> ssh_tunnel_user_local user):

```code
ssh ssh_tunnel_user_remote@ssh_tunnel_remote_host_ip -J ssh_tunnel_user_jump@ssh_tunnel_jump_host -N -o ExitOnForwardFailure=yes -o "ServerAliveInterval 60" -o "ServerAliveCountMax 3" -R 0.0.0.0:ssh_tunnel_local_port:localhost:ssh_tunnel_remote_port
```

If you use a FQDN for *ssh_tunnel_target_host_ip* you need to ensure, that *ssh_tunnel_target_host_ip* is resolvable and reachable by *ssh_tunnel_remote_host* and/or *ssh_tunnel_jump_host*.

Combine as and if needed, also for (*ssh_tunnel_user_jump*).

Only if *ssh_tunnel_target_host_ip* is defined, *ssh_tunnel_user_target* is properly taken care of.

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
    # *ssh_tunnel_local_interface*: "0.0.0.0" # local listen interface, default 0.0.0.0
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
Contributors: Jens Gecius 2024

# sleif.ssh_tunnel

This role creates a systemd based outgoing (default) or reverse ssh tunnel.

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
- ssh_tunnel_local_interface # locally listen interface ip
- ssh_tunnel_remote_host_ip # IP address of the end-point host in case of a jump host
- ssh_tunnel_remote_interface # outgoing end-point interface ip
- ssh_tunnel_user_jump # restricted user - account will be created - defaults to ssh_tunnel_user_remote
- ssh_tunnel_user_local # restricted user - account will be created
- ssh_tunnel_user_remote # restricted user - account will be created
- storage_dir_base # local storage base

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

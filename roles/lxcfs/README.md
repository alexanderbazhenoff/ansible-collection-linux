lxcfs
=====
Installs and configures [lxc](https://linuxcontainers.org/lxc/introduction/) or
[lxcfs](https://linuxcontainers.org/lxcfs/introduction/) on linux system.

Requirements
------------
- Any Debian or Ubuntu (tested on LTS version, but perhaps work on any version), any RedHat and Alpine Linux 
distributive.
- jmespath installed: `python3 -m pip install jmespath`.

Role Variables
--------------
- `role_action` - role action to perform, e.g: `install` or `uninstall`.
- `clean_install` - perform clean installation of lxc/lxcfs.
- `lxc_version` - version of lxc/lxcfs, e.g: `3.0` or `2.0`. Affects only on RedHat repository URL. Please note: there
is no lxcfs for 2.0, only lxc.
- `lxc_technology` - container technology to install: `lxc` or `lxcfs`.
- `lxc_gpg_keyserver` - add LXC GPG keyserver (`DOWNLOAD_KEYSERVER`) to environment variables, e.g:
`keyserver.ubuntu.com`.
- `check_host_is_lxc_ready` - check host is lxc-ready (run `lxc-checkconfig` command when *True*). Version of lxc may
differ from distribution to distribution where lxc-checkconfig script could be 
[a bit outdated](https://github.com/lxc/lxc/issues/4070#issuecomment-1374883653).
- `append_lxc_config` - append config lines to default lxc config. Paste something for default container settings, e.g:
```
append_lxc_config: |
  lxc.net.0.type = veth
  lxc.net.0.link = lxcbr0
  lxc.net.0.name = eth0
  lxc.net.0.flags = up
  lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
  lxc.net.0.veth.pair = pariname-lxc
  lxc.net.0.ipv4.address = 10.0.0.200/24
  lxc.net.0.ipv4.gateway = 10.0.0.254
  lxc.start.auto = 1
  lxc.start.delay = 8
```
for LXC3.0 or:
```
  lxc.network.type = veth
  lxc.network.link = lxcbr0
  lxc.network.flags = up
  lxc.start.auto = 1
  lxc.start.delay = 8
```
for LXC2.0. So `append_lxc_config` lines will append to all new created containers configs.

## lxc-net settings

- `use_lxc_default_net` - use default lxc bridge-nat network (`lxc-net` daemon) bind to `lxc_default_net_bridge`
interface. Disable (set `False`) if you want to connect with host bridge, pass-through, etc.
- `lxc_default_net_bridge` - name of the interface for bridge-nat network (default: `xcbr0`).
- `lxc_default_net_addr` - IP address for `xcbr0`, e.g: `"10.0.3.1"`.
- `lxc_default_net_netmask` - subnet mask for bridge-nat network, e.g.: `"255.255.255.0"`.
- `lxc_default_net_network` - broadcast and CIDR prefix for bridge-nat network, e.g.: `"10.0.3.0/24"`.
- `lxc_default_net_dhcp_range` - DHCP range for bridge-nat network, e.g.: `"10.0.3.2,10.0.3.254"`.
- `lxc_default_net_dhcp_max` - max number of DHCP leases in `lxc_default_net_dhcp_range`, e.g.: `"253"`.

Example Playbook
----------------

Simple lxc role usage with defaults is:

    - hosts: all
      become: True
      become_method: sudo
      gather_facts: True

      tasks:
        - name: Include lxc(fs) role
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.lxcfs
          vars:
            lxc_technology: lxc

You can also switch to lxcfs with:

            lxc_technology: lxcfs

More complex lxcfs usage with disabled default lxc-net for br0 usage with some static addresses should look like:

    - hosts: all
      become: True
      become_method: sudo
      gather_facts: True

      tasks:

        # Define your br0 brdige here

        - name: Include lxc(fs) role
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.lxcfs
          vars:
            lxc_technology: lxcfs
            use_lxc_default_net: False
            lxc_append_default_config: |
              lxc.net.0.type = veth
              lxc.net.0.link = br0
              lxc.net.0.flags = up
              lxc.net.0.hwaddr = 00:16:3e:xx:xx:xx
              lxc.net.0.veth.pair = someprefix-lxc
              lxc.net.0.name = eth0
              lxc.net.0.link = br0
              lxc.net.0.ipv4.address = 10.0.0.200/24
              lxc.net.0.ipv4.gateway = 10.0.0.254
              lxc.start.auto = 1
              lxc.start.delay = 8

License
-------
MIT-0

lxcfs
=====
Installs [lxc](https://linuxcontainers.org/lxc/introduction/) or
[lxcfs](https://linuxcontainers.org/lxcfs/introduction/) on linux system.

Requirements
------------
- Ubuntu18.04, 20.04 or CentOS7 distro.
- jmespath installed: `python3 -m pip install jmespath`.
- For CentOS you should have already installed epel-release (for package 'debootstrap').
- If you wish bridged network you should create bridge before. This role does not include network bridge creation.

Role Variables
--------------
- `lxc_technology` - container technology to install: lxc or lxcfs.
- `use_lxc_default_net` - use lxc default network or not.
- `check_host_is_lxc_ready` - check host is lxc-ready (run *lxc-checkconfig*
command when True).
- `lxc_append_default_config` - append config lines to default lxc config.

Example Playbook
----------------

Simple lxc role usage is:

    - hosts: all
      become: True
      become_method: sudo
      roles:
        - role: lxcfs
          lxc_technology: lxc
          use_lxc_default_net: False

You can also switch to lxcfs with:

          lxc_technology: lxc

More complex lxcfs usage with epel-release installation. Disabled default lxc-net and custom br0 usage with some static
addresses should look like:

    - hosts: all
      become: True
      become_method: sudo
      tasks:

        - name: "default >>> prepare | Install epel_repo (CentOS)"
          ansible.builtin.yum:
            name: epel-repo
          when: ansible_distribution == "CentOS"
    
        - name: "default >>> prepare | Update yum cache (CentOS)"
          ansible.builtin.yum:
            update_cache: True
          when: ansible_distribution == "CentOS"

        # br0 creation tasks here
        # ...

        - name: "kvm >>> side effect | Include lxc role"
          ansible.builtin.include_role:
            name: "lxcfs"
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
              lxc.net.0.ipv4.address = 10.10.1.200/24
              lxc.net.0.ipv4.gateway = 10.10.1.1
              lxc.start.auto = 1
              lxc.start.delay = 8

In this case `lxc_append_default_config` will append to all new created containers configs.

License
-------
MIT-0
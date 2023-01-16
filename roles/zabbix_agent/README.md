zabbix_agent
=========
This role installs zabbix agent and configures them for services, apps and platform autodiscovery on
[Zabbix Server side](#setup-on-zabbix-server-side).

Configure tags for discovery of: mysql, postgresql, bareos, nginx, apache, isc-dhcp-server, memcached, virtualization,
containerization, baremetal platforms vendors and many more.

Preamble
--------
Zabbix server can detect applications, services, platforms and virtualization by a special hash value (see 
autodiscovery hashes) predefined in a Zabbix agent configuration file. For example:

```
UserParameter=hostmetadata,echo 'Linux ff6e6f6e65 dd6a76'
```

When Zabbix Server tries to discover hosts like this one, the next step is following: add the host to a specified 
group (like Web servers, Cloud nodes, etc) and apply template to monitoring filesystems (ZFS, btrfs, mdadm), 
application (nginx, etc), virtualization or containerization and another hardware and software platforms.

Usage
-----

1. Download and [install](https://galaxy.ansible.com/docs/using/installing.html#roles) this ansible role from ansible
galaxy or clone from github.
2. Set-up your zabbix server IPs and other params in [defaults/main.yml](defaults/main.yml). If you need to add your
custom daemon discovery (something you usually do like `systemctl start my-app-daemon`), add it in the 'daemon hashes'
section.
3. Fill-up [inventory file](examples/inventory) or prepare your
[connection method](https://docs.ansible.com/ansible/latest/user_guide/connection_details.html) then call the role
from [install_agent_example.yml](examples/install_agent_example.yml). Or call this role other way in your ansible playbooks
according to [examples below](#Example-Playbook).

Requirements
------------
- **Zabbix version.** Written for Zabbix 5.0 automation, but probably works for other versions of Zabbix agent. Before
  you setup please confirm package version is available for your version of Linux distribution visiting [Zabbix download
  page](https://www.zabbix.com/download?zabbix=5.0) or directly [repository](https://repo.zabbix.com/zabbix/).

- **Operation systems**. Works only with x64 Linux:
    - **Ubuntu: 14.04, 16.04, 18.04, 20.04, 22.04**. Perhaps this role also works on Non-LTS or older versions, but
      wasn't tested.
    - **Debian: 8, 9, 10, 11**. Perhaps this role also works on another versions of Debian distro, but wasn't tested.
    - **RHEL: 7**. Perhaps this role also works on older versions of RHEL distro, but tested only on CentOS 7.
    - Other Linux distributions, Windows, Solaris, FreeBSD, AIX and armv7/aarch64 architectures aren't supported.

- **Zabbix agent configuration**. Platforms discovery use additional packages (like ipmitool), another platforms and
services detects by binary files search, but the most of services detects by their enabled systemd daemons (`systemctl
is-enabled` command). So deamons autodiscovery works only on linux distros with systemd. Already installed Zabbix agent
re-configuration (check `customize_agent_only` in [Role Variables](#role-variables) using this role also possible.

Dependencies
------------
- [pacman module](https://docs.ansible.com/ansible/latest/collections/community/general/pacman_module.html)
from community.general;
- `xxd` or `vim-common`, `wget` and `policycoreutils-python` packages (will be automatically installed on supported 
Linux distributions, otherwise install them manually);
- `sudo` package on Debian systems installed via 'netinstall' source or docker images. 

Role Variables
--------------
Main parameters:

| Variable                           | default | Comment                                                              |
|------------------------------------|---------|----------------------------------------------------------------------|
| `agent_version`                    | 5.0     | Zabbix agent version                                                 |
| `install_v2_agent`                 | True    | Try to install Zabbix agent v2                                       |
| `customize_agent`                  | True    | Configure agent for automatic services discovery and templates add   | 
| `clean_install`                    | True    | Perform clean install (re-install agent with clean-up)               |
| `force_install_agent_v1`           | True    | Force install Zabbix agent v1 for all versions of Linux distribution |
| `zabbix_agent_conf_incl_dir_clean` | True    | Configure or perform clean install with config directory clean-up    |
| `debug_mode`                       | True    | More outputs                                                         |
| `customize_agent_only`             | False   | Re-configure agent without Zabbix install                            |
| `repo_dl_validate_certs`           | False   | Check certificates on Zabbix repo package download (required on      |
|                                    |         | outdated virtual machines snapshots)                                 |

Zabbix agent settings:

| Parameter                  | Default                          | Comment                         |
|----------------------------|----------------------------------|---------------------------------|
| `remote_cmd_key`           | EnableRemoteCommands             | Enable remote commands key name |
| `remote_cmd_value`         | 1                                | Enable remote commands value    |
| `logfile_size_key`         | LogFileSize                      | Logfile size key name           |
| `logfile_size_value`       | 100                              | Logfile size value              |
| `zabbix_servers_passive`   | 10.10.1.2,10.10.1.3              | Zabbix server IP passive checks |
| `zabbix_servers_active`    | 10.10.11.2:10051,10.10.1.3:10051 | Zabbix server IP active checks  |
| `zabbix_host_metadata`     | Linux                            | Zabbix host metadata            |

Here is another agent settings related variables like `zabbix_agent_conf_name`,  `zabbix_agent_daemon_name`,
`zabbix_agent_binary`, but changing them is useless. This is for Zabbix agents versions differences discovery.

Autodiscovery hashes:

| Binary/platform/daemon name | hex hash               | Comment                                       |
|-----------------------------|------------------------|-----------------------------------------------|
| openvpn                     | dd6f7664               | Openvpn binary/daemon discovery               |
| java                        | dd6a76                 | Java platform binary discovery                |
| mysql / mariadb             | dd6d716c               | MySQL daemon discovery                        |
| postgresql                  | dd70716c               | PostgreSQL daemon discovery                   |
| cloudstack-agent            | dd637361               | Apache Cloudstack agent daemon discovery      |   
| cloudstack-management       | dd63736d               | Apache Cloudstack management daemon discovery |
| bareos-fd                   | dd62726664             | Bareos File Daemon discovery                  |
| bareos-dir                  | 62726472               | Bareos Director Daemon discovery              |
| bareos-sd                   | dd62727364             | Bareos Storage Daemon discovery               |
| hm_memcached                | dd6d6d63636864         | Memcached daemon discovery                    |
| hm_nginx                    | dd6e6778               | Nginx daemon discovery                        |
| hm_php_fpm                  | dd7066706d             | php-fpm daemon discovery                      |
| httpd                       | dd61706368             | Apache web server daemon discovery            |
| named                       | dd6e646e73             | Named (dhcpd) daemon discovery                |
| dhcpd                       | dd69736364686370       | isc-dhcp-server daemon discovery              |
| dnsmasq                     | dd646e736d             | dnsmasq daemon discovery                      |
| fail2ban                    | dd66626e               | fail2ban daemon discovery                     |
| vsftpd                      | dd7673667470           | vstftpd daemon discovery                      |
| gitlab-runner               | dd6774726e             | Gitlab runner daemon discovery                |
| nfs                         | dd6e6673               | NFS Server daemon discovery                   |
| zabbix-agent                | dd7a626131             | Zabbix agent v1 daemon discovery              |
| zabbix-agent2               | dd7a626132             | Zabbix agent v2 daemon discovery              |
|                             | ff6e6f6e65             | baremetal server instance discovery           |
| kvm                         | ff6b766d               | KVM virtualization instance dicovery          |
| lxc                         | ff6c7863               | LXC container instance discovery              |
| docker                      | ff646f636b6572         | Docker container instance discovery           |
| ipmi                        | ee69706d69             | IPMI discovery                                |
| Supermicro                  | ee53757065726d6963726f | Sumermicro(tm) hardware platform disovery     |
|                             | ee<other_vendor_hash>  | <other vendor> hardware platform disovery     |

Example Playbook
----------------

Install and configure Zabbix v1 agent(s):

    - hosts: all
      become: True
      become_method: sudo
      roles:
        - role: zabbix_agent
          install_v2_agent: False
          force_install_agent_v1: True

Install and configure Zabbix v2 agent(s):

    - hosts: all
      become: True
      become_method: sudo
      roles:
        - role: zabbix_agent
          install_v2_agent: True

Perform re-configuration of already installed Zabbix agent(s):

    - hosts: all
      become: True
      become_method: sudo
      roles:
        - role: zabbix_agent
          customize_agent_only: True

Setup on Zabbix Server side
---------------------------
To make services, platforms and apps discovery working on Zabbix Server side you should perform additional settings:

1. Open `Configuration -> Discovery` on Zabbix Server Web UI. Select your discovery rule for network IP range then
add `Zabbix agent "hostmetadata"` to "Checks". After successful Zabbix agent reconfiguration using this role you'll see
`hostmetadata` column in the "Status of discovery" (`Monitoring -> Discovery` on Zabbix Server Web UI). Just wait for
some minutes for IP range scan.
2. Open `Configuration -> Actions -> Dicovery actions` in Zabbix Server Web UI and create your rule for `hostmetadata`
tag. You wish to create "MySQL - discovery on linux for Zabbix agents v1" for example:

Action:
```
(A and B and C) and D and E and F

Conditions:
A   Received value contains Linux
B   Received value contains dd7a626131
C   Received value contains dd6d716c
D   Uptime/Downtime is greater than or equals 3600
E   Discovery status equals Up
F   Service type equals Zabbix agent

```
Operations:
```
Run remote commands on current host
Add to host groups: Services/MySQL servers
Link to templates: Template DB MySQL by Zabbix agent
```
Remote Command (Zabbix agent for current host):
```
wget -N -O /tmp/mysql_selinux.sh http://yourzabbbix.domain/downloads/mysql/mysql_selinux.sh
bash /tmp/mysql_selinux.sh
echo "[client]" > $(getent passwd zabbix | cut -d : -f 6)/my.cnf
echo "user='zbx_monitor'" >> $(getent passwd zabbix | cut -d : -f 6)/my.cnf
echo "password='Yourpassword'" >> $(getent passwd zabbix | cut -d : -f 6)/my.cnf
chown zabbix:zabbix $(getent passwd zabbix | cut -d : -f 6)/my.cnf
wget -N -O /etc/zabbix/zabbix_agentd.d/template_db_mysql.conf http://yourzabbbix.domain/downloads/mysql/template_db_mysql.conf
wget -N -O /tmp/restart_daemon.sh http://yourzabbbix.domain/downloads/restart_daemon.sh
echo "none" > /tmp/zabbix-agent-restart.log
bash /tmp/restart_daemon.sh zabbix-agent
echo "ok" > /tmp/zabbix-agent-restart.log
```
where you should change `yourzabbbix.domain` URL and `'Yourpassword'` password.

The following steps are for the commands execution to add template configurations and settings on Zabbix-agent side after the 
service discovery. It means you didn't put them in `/etc/zabbix` or whatever before discovery on Zabbix Server.
**Otherwise skip the next steps.**

3. Put your template available to download from Zabbix Server, e.g.
`/usr/share/zabbix/downloads/mysql/template_db_mysql.conf`:
```
UserParameter=mysql.ping[*], mysqladmin --defaults-extra-file='/var/lib/zabbix/my.cnf' ping
UserParameter=mysql.get_status_variables[*], mysql --defaults-extra-file='/var/lib/zabbix/my.cnf' -sNX -e "show global status"
UserParameter=mysql.version[*], mysqladmin --defaults-extra-file='/var/lib/zabbix/my.cnf' version
UserParameter=mysql.db.discovery[*], mysql --defaults-extra-file='/var/lib/zabbix/my.cnf' -sN -e "show databases"
UserParameter=mysql.dbsize[*], mysql --defaults-extra-file='/var/lib/zabbix/my.cnf' -sN -e "SELECT SUM(DATA_LENGTH + INDEX_LENGTH) FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA='$3'"
UserParameter=mysql.replication.discovery[*], mysql --defaults-extra-file='/var/lib/zabbix/my.cnf' -sNX -e "show slave status"
UserParameter=mysql.slave_status[*], mysql --defaults-extra-file='/var/lib/zabbix/my.cnf' -sN -e "show slave status"
```
4. Put your script to change Zabbix-agent selinux policies available to download from Zabbix Server (skip this step for
non-RedHat Linux distros), e.g. `/usr/share/zabbix/downloads/mysql/mysql_selinux.sh`:

```bash
#!/usr/bin/env bash


if [[ -f /etc/redhat-release ]] && [[ $(sestatus | head -1 | awk '{print $3}') == "enabled" ]]; then

cat <<EOF > zabbix_home.te
module zabbix_home 1.0;

require {
        type zabbix_agent_t;
        type zabbix_var_lib_t;
        type mysqld_etc_t;
        type mysqld_port_t;
        type mysqld_var_run_t;
        class file { open read };
        class tcp_socket name_connect;
        class sock_file write;
}

#============= zabbix_agent_t ==============

allow zabbix_agent_t zabbix_var_lib_t:file read;
allow zabbix_agent_t zabbix_var_lib_t:file open;
allow zabbix_agent_t mysqld_etc_t:file read;
allow zabbix_agent_t mysqld_port_t:tcp_socket name_connect;
allow zabbix_agent_t mysqld_var_run_t:sock_file write;
EOF
checkmodule -M -m -o zabbix_home.mod zabbix_home.te
semodule_package -o zabbix_home.pp -m zabbix_home.mod
semodule -i zabbix_home.pp
restorecon -R /var/lib/zabbix

fi
```
5. Put your script to restart zabbix-agent daemon (to apply config change) available to download from Zabbix Server,
e.g. ` /usr/share/zabbix/downloads/restart_daemon.sh`:
```
#!/usr/bin/env bash


SERVICE_NAME=$1
echo "Restarting ${SERVICE_NAME}..."

if [[ -f /bin/systemctl ]]; then
    set -x
    systemctl restart ${SERVICE_NAME}
else
    if [[ -f /usr/sbin/service ]]; then
        set -x
        service ${SERVICE_NAME} restart
    else
        if [[ -f /etc/init.d/${SERVICE_NAME} ]]; then
            set -x
            /./etc/init.d/${SERVICE_NAME} restart
        else
            echo "Fatal error: no systemctl, or service or init.d script found for ${SERVICE_NAME}"
            exit 1
        fi
    fi
fi
```
You can also make your custom restart script based on config changes (using diff or whatever). This is just a concept.

License
-------
MIT-0

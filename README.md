An ansible collection with linux-related roles to perform various setups.


## Project content.

#### Roles:

- [**lxcfs**](roles/lxcfs/README.md): Installs [lxc](https://linuxcontainers.org/lxc/introduction/) or
[**lxcfs**](https://linuxcontainers.org/lxcfs/introduction/) on linux system.

- [**zabbix_agent**](roles/zabbix_agent/README.md) - Installs zabbix agent and configures them for services,
apps and platform autodiscovery on
[Zabbix Server side](https://github.com/alexanderbazhenoff/ansible-collection-linux/tree/main/roles/zabbix_agent#setup-on-zabbix-server-side).

- [**bareos**](roles/bareos/README.md) - Installs and configures [Bareos](https://www.bareos.com/) and third-party
components, adds Bareos file daemons to Bareos server, create or revoke user profiles and uploads Bareos configs to the
host (e.g. if you have predefined Bareos server, storage daemon etc).


## Installation.

#### Install from ansible galaxy:

Set-up collection from ansible galaxy: `ansible-galaxy collection install alexanderbazhenoff.linux`. If you need to
use custom installation path, e.g. `ansible-galaxy collection install alexanderbazhenoff.linux -p /your/path` then
edit ansible.cfg. Use `ansible --version` command to find the path of config file. Check
[docs.ansible.com](https://docs.ansible.com/ansible/latest/user_guide/collections_using.html#installing-collections-with-ansible-galaxy)
for more info.

#### Install from sources:

1. Clone via ssh: `git clone git@github.com:alexanderbazhenoff/ansible-collection-linux.git` or https:
`git clone https://github.com/alexanderbazhenoff/ansible-collection-linux.git`.
2. Enter project directory `cd ansible-collection-linux` and build tar.gz archive of collection:
`ansible-galaxy collection build`.
3. Install collection from created tar.gz archive: `ansible-galaxy collection install $(ls -1 | grep ".tar.gz")`.


## Usage.

1. Create inventory and create playbook files. Include these roles into your playbook. These roles include example
playbooks and inventory files already. You can also find usage examples in readme files.
2. If you like it please vote on [ansible galaxy page](https://galaxy.ansible.com/alexanderbazhenoff/linux).

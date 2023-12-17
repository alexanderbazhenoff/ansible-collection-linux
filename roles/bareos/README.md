bareos
======
This role installs and configures [Bareos](https://www.bareos.com/) and required third-party components.

Specification
-------------
Normally this role is for the next cases:

- Typical installation of Bareos director using PostgreSQL, Bareos Web UI using Apache, Bareos storage and Bareos
file daemon on the same host and upload predefined configs to `/etc` (optional, see 
[**bareos_configs_to_copy** variable](#role-variables)). This type of installation is the most ready-to-use (see
[example playbook](#1-install-bareos-director-web-ui-storage-file-daemon-on-the-same-host) for details).
- Create or revoke user profile to access Bareos Web UI (see
[example playbook](#2-make-client-user-profile-to-access-bareos-web-ui) for details).
- Typical installation of Bareos file-daemon and add them to Bareos director. This type of installation is also
ready-to-use (check see [**role_action**](#role-variables) variable description and
[example playbook](#3-install-bareos-file-daemon-and-add-to-server) for details).
- Install a list of additional Bareos packages, e.g.: bareos-traymonitor (optional, see 
[example playbook](#4-install-bareos-components-with-an-additional-list-of-packages)). No additional custom settings for
these components will be performed running this role, just default installation(s).
- Uninstall Bareos components: Bareos Director, Bareos UI, Bareos Director, Bareos Storage (check
[example playbook](#5-uninstall-bareos-component) for details).
- Copy Bareos configs without components re-install (see
[example playbook](#7-copy-bareos-configs-without-reinstalling)).

Additional usage scenarios:

- Installation of Bareos director using PostgreSQL, Bareos Web UI using Apache, Bareos storage and bareos file daemon on
the same host, but without a database (see 
[example](#6-install-bareos-with-a-pre-installed-database-and-copy-pre-defined-configs)). So you can preinstall a
database first.
- Installation of Bareos Web UI (e.g. connect to Bareos director on another host). This usage scenario without Web UI
pre-configuration, but you can upload additional predefined configs to `/etc` (optional, see 
[**bareos_configs_to_copy** variable](#role-variables)).
- Installation of Bareos Director without Web UI and pre-configuration (but you can also upload additional predefined 
configs to `/etc`).
- Installation of Bareos Storage separately without pre-configuration (but you can also upload additional predefined 
configs to `/etc`).

Role limitations:

- You can install Bareos Web UI with Apache web server only. If you wish to use Bareos with nginx, please
[do it manually](https://docs.bareos.org/IntroductionAndTutorial/InstallingBareosWebui.html#nginx).
- Installation of Bareos director with sqlite or mysql/mariadb (for older versions) is not supported.
- This role tested with Bareos v.21 and PostgreSQL v14. Various details of older or newer versions of Bareos
configuration Bareos or third-party components are not included in this role: e.g.
[manual installation of php required](https://blog.remirepo.net/post/2019/12/03/Install-PHP-7.4-on-CentOS-RHEL-or-Fedora)
instead of wrong versions by default like on Oracle Linux 8.7 or Fedora 45.
- No Bareos resources like devices, storages, jobs, pools are possible to add. Only bareos file daemon adds, which is
the most common role usage.
- Remove file daemon from Bareos server using this role is not possible. Some existing job configs might be linked with
this file-daemon. Bareos server will not start after force remove until you delete or change job configs. Anyway, there
is no normal scenario when you want to remove file daemon from the server while you can change job configs and 
optionally disable on Bareos server. 
- Firewall rules will be applied only when firewalld (RedHat) or ufw (Ubuntu) enabled. This will not affect to iptables
on Debian, so use external or manual iptables control.
- TLS certificates and setting not supported in this role, but you can copy all required files setting up
[`bareos_configs_to_copy` variable](#role-variables).

Requirements
------------
- Any version of Ubuntu or any Debian family distribution, any RedHat family distribution or any Alpine Linux (but
please check `override_ansible_distribution_major_version` [role variable description](#main-role-variables-) first).

Dependencies
------------
- PostgreSQL which can be installed using this role or not.
- [python-psycopg](https://www.psycopg.org/) is not required for Bareos, but will be installed together with PostgreSQL.
- Apache2 web server and Epel repo (for `libzstd` package download on CentOS) which will be automatically installed 
running this role.
- This role using 'sudo' become method, so Debian distribution should have them already pre-installed.
- Alpine Linux binaries built unofficially they could have some bugs 
([like this one](https://gitlab.alpinelinux.org/alpine/aports/-/issues/14570)). Before you run this role on Alpine Linux
set your `/etc/apk/repositories` manually (read [documentation](https://wiki.alpinelinux.org/wiki/Repositories)).

Example Playbooks
-----------------
The easiest ways to use this role should look like typical installation: 
[all Bareos components](https://docs.bareos.org/IntroductionAndTutorial/InstallingBareosWebui.html#installation) on the
same host and/or 
[install Bareos file daemon](https://docs.bareos.org/master/IntroductionAndTutorial/InstallingBareosClient.html)
(client) then [add them](https://docs.bareos.org/IntroductionAndTutorial/Tutorial.html#director-configure-client) to
Bareos director (server). You may also wish to create a client user profile to access
[Web UI](https://docs.bareos.org/IntroductionAndTutorial/BareosWebui.html) because there are no available administrator
profiles by default.

In another hand, you can install or uninstall single Bareos components and copy pre-defined configs, install Bareos
director with an already pre-installed database. You can also copy predefined configs if you wish Bareos v20 with an
sqlite database, or mariadb/mysql for an older version of Bareos (I am happy not supported anymore because of lots of
tuning to speed up).

#### 1. Install Bareos Director, Web UI, Storage, File daemon on the same host

    - hosts: all
      become: True
      become_method: sudo
      gather_facts: True
    
      tasks:
    
        - name: "Install bareos-dir, web ui, -fd and -sd"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.bareos
          vars:
            role_action: install
            bareos_components: dir_webui
            init_bareos_database: "{{ (ansible_os_family == 'RedHat') }}"
            postgresql_version: 14

#### 2. Make client user profile to access Bareos Web UI

    - hosts: bareos_server_host
      become: True
      become_method: sudo
      gather_facts: True

      tasks:

        - name: "Create Admin client user profile with webui-admin permissions to get full Web UI access"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.bareos
          vars:
            role_action: access
            webui_username: Admin
            webui_password: your_password_here
            webui_profile: webui-admin
            webui_tls_enable: False

You can set `webui-admin`, `operator`, `webui-limited` or `webui-readonly` access level (webui_profile)  
[your profile needs](https://docs.bareos.org/IntroductionAndTutorial/BareosWebui.html#access-control-configuration).
For revoke access replace the next line:

            role_action: access

If you wish to prompt user and password during playbook execution:

      vars_prompt:

        - name: "webui_username"
          prompt: "Enter username"
          default: ""
          private: False
        - name: "webui_password"
          prompt: "Enter password"
          default: ""
          private: True

      tasks:

        - name: "Create Admin client user profile with webui-admin permissions to get full Web UI access"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.bareos
          vars:
            role_action: access
            webui_profile: webui-admin
            webui_tls_enable: False

#### 3. Install Bareos file daemon and add to server

To install a client, then add them to server, you can use separate groups for client(s) and server(s) in ansible
inventory file, e.g.:

      [clients]
      10.1.1.2
      10.1.1.3
        
      [server]
      10.1.1.1

and ansible playbook like:

    - hosts: all
      become: True
      become_method: sudo
      gather_facts: True
    
      tasks:

        - name: "Install bareos file daemon(s) on host group clients"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.bareos
          vars:
            bareos_components: fd
            role_action: install
          when: inventory_hostname in groups["clients"]
    
        - name: "Add bareos file daemon(s) from host group clients to the first one of the host group servers"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.bareos
          vars:
            role_action: add_client
            add_component_server: "{{ groups.servers[0] }}"
            debug_mode: True
          when: inventory_hostname in groups["clients"]

#### 4. Install Bareos components with an additional list of packages

    - hosts: all
      become: True
      become_method: sudo
      gather_facts: True
    
      tasks:
    
        - name: "Install bareos-dir, web ui, -fd and -sd"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.bareos
          vars:
            role_action: install
            bareos_components: dir_webui
            postgresql_version: 14
            init_bareos_database: "{{ (ansible_distribution == 'CentOS') }}"

The example below shows how to install a client with only 
[**bareos-traymonitor**](https://docs.bareos.org/Configuration/Monitor.html) in the list of additional packages:

    - hosts: all
      become: True
      become_method: sudo
      gather_facts: True
    
      tasks:

        - name: "Install bareos file daemon(s) and bareos-traymonitor"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.bareos
          vars:
            bareos_components: fd
            role_action: install
            install_additional_bareos_packages:
              - bareos-traymonitor

If the list (`install_additional_bareos_packages` variable) is empty Nothing will be installed additionally together
with Bareos components.

#### 5. Uninstall Bareos component

You can uninstall Bareos file daemon (fd), Bareos Storage Daemon (sd), Bareos director (dir), Bareos Web UI (webui)
separately using `bareos_components` variable. The next example shows how to uninstall Bareos Storage Daemon from
the host(s):

    - hosts: all
      become: True
      become_method: sudo
      gather_facts: True
    
      tasks:

        - name: "Uninstall Bareos Storage Daemon"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.bareos
          vars:
            bareos_components: sd
            role_action: uninstall

#### 6. Install Bareos with a pre-installed database and copy pre-defined configs

The next example shows how to skip PostgreSQL installation and copy Bareos director configs from `files` folder of this
role. Put your director configs to `files` folder of this role then run:

    - hosts: all
      become: True
      become_method: sudo
      gather_facts: True
    
      tasks:
    
        - name: "Install bareos-dir, web ui, -fd and -sd; skip PostgreSQL install; copy configs to /etc/bareos-dir.d"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.bareos
          vars:
            bareos_components: dir_webui
            use_postgresql: True
            preinstalled_postgresql: True
            bareos_configs_to_copy:
              - source: etc/
                destination: /etc/bareos-dir.d
                owner: bareos
                group: bareos

Nothing will be additionally copied if `bareos_configs_to_copy` list is empty.

If you wish to use Bareos with an already preinstalled sqlite change set:

            use_postgresql: False
            preinstalled_postgresql: True

#### 7. Copy Bareos configs without reinstalling

You can upload configs to an already installed Bareos server without components re-install, but you need to set what
component(s) should be restarted to apply configuration change in `bareos_components` variable. The example below shows
how to apply bareos-dir, bareos-sd, bareos-fd and Bareos Web UI (by restart Apache web server):

    - hosts: all
      become: True
      become_method: sudo
      gather_facts: True
    
      tasks:
    
        - name: "Copy configs to /etc/bareos-dir.d without Bareos components reinstall"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.bareos
          vars:
            bareos_components: dir_webui
            role_action: copy_configs
            bareos_configs_to_copy:
              - source: etc/
                destination: /etc/bareos-dir.d
                owner: bareos
                group: bareos

Role Variables
--------------

#### Main role variables:

- **role_action** *[Default: install, possible values: install/uninstall/access/add_client/copy_configs]*: Role action
to be performed: install or uninstall Bareos components, create user profile to access Bareos Web UI *(access)*, add
already installed file daemon *(client)* on Bareos server *(add_client)*, copy Bareos config to already installed Bareos
component(s).
- **bareos_components** *[Default: fd, possible values: fd, sd, dir, webui, dir_webui]*: Bareos components to install or
uninstall: Bareos file daemon or client *(fd)*, Bareos storage daemon *(sd)*, Bareos Director *(dir)*, Bareos director 
and Web UI *(dir_webui)*.
- **clean_install** *[Default: True]*: Perform clean installation. All packages and configs will be purged before
install.
- **bareos_release** *[Default: 21]*: [Bareos release version](https://download.bareos.org/bareos/release/), e.g.: 'current', 'next' or 
'experimental/<postfix>'. Affects only for Debian and Redhat Linux distribution families.
- **bareos_url** *[Default: https://download.bareos.org/bareos/release/]*: Bareos repository URL prefix to download from
(for example, if you with to use local or another repo).
- **override_ansible_distribution_major_version** *[Default: None]*: Override ansible linux distribution major version.
Useful when specified Bareos version repo is not available for your Linux distribution version, but beware package
dependencies problems (example: Bareos v21 currently is [not available](https://download.bareos.org/bareos/release/21/)
for any RedHat v9, so try to set `8` here).
- **debug_mode** *[Default: False]*: Verbose output.

#### Firewall related variables:

- **firewall_control** *[Default: True]*: Add firewall rules for firewalld and ufw.

#### Bareos related variables:

- **cleanup_storage_files** *[Default: True]*: Cleanup Bareos storage files on installation or uninstall. Please keep in
mind these files are useless without an indexes stored in the database.
- **install_additional_bareos_packages** *[Default: `[bareos-traymonitor]`]*: Additional Bareos packages to install:
`[]`or `[bareos-traymonitor, bareos-vmware-plugin]`, etc...
- **bareos_configs_to_copy**: [Default: `[{source: etc/, destination: /etc, owner: bareos, group: bareos}]`] List of
additional configs to copy on components install. Performs copying items from item.source to item.destination with
item.owner, item.group and item.mode, otherwise set this list to []. E. g:
```yaml
bareos_configs_to_copy:
  - source: source/inside/files/folder_1
    destination: /destination/path/on/the/host_1
    owner: bareos
    group: bareos
  - source: source/inside/files/folder_2
    destination: /destination/path/on/the/host_2
    owner: bareos
    group: bareos
```

For creating client user profile to access Bareos Web UI (role_action == *'access'*) you could use:

| Variable          | Default value | Comment                                                                    |
|-------------------|---------------|----------------------------------------------------------------------------|
| webui_username    | Admin         | Profile username to login with.                                            |
| webui_password    | admin         | Profile password to login with.                                            |
| webui_profile     | webui-admin   | Access profile, e.g: webui-admin, operator, webui-readonly, webui-limited. |
| webui_tls_enable  | False         | Use TLS.                                                                   |

All available profiles are stored in the directory /etc/bareos/bareos-dir.d/profile (read
[here](https://www.bareos.com/bareos-webui-installation-and-configuration/) for details).

To add file daemon on Bareos server, you could define the next variables:

| Variable                          | Default   | Comment                                                           |
|-----------------------------------|-----------|-------------------------------------------------------------------|
| add_component_name                | ''        | Name to display on Bareos server (leave '' for FQDN or hostname). |
| add_component_password            | ''        | Password to connect with (leave '' to generate random password).  |
| add_component_password_gen_length | 16        | Length of the password to generate.                               |
| add_component_tls_enable          | False     | Use TLS to connect between storage and server.                    |
| add_component_max_jobs            | 1         | Maximum concurrent jobs on avalialable on file daemon.            |
| add_component_server              | 127.0.0.1 | Bareos server IP address, hostname or inventory item to delegate. |
| add_component_force               | False     | Force try to add file daemon when already exists                  |

#### Database related parameters:

The most useful settings are stored in:

- **preinstalled_postgresql** *[Default: False]*: If you wish to use already preinstalled PostgreSQL.
- **use_postgresql** *[Default: True]*: Enable to use Bareos with PostgreSQL, disable for sqlite.
- **postgresql_version** *[Default: 14]*: PostgreSQL version to install by this role. Leave it `''` for a default
version.
- **install_psycopg2** *[Default: True]*: Install [psycopg2](https://pypi.org/project/psycopg2/).
- **postgresql_timezone** *[Default: 'Europe/Moscow']*: PostgreSQL timezone (read 
[manual](https://www.postgresql.org/docs/15/datatype-datetime.html)).

Another variables:

- **postgresql_url** *[Default: https://download.postgresql.org/pub/]*: Download PostgreSQL repository URL prefix.
- **postgresql_debian_key_url** [Default: https://www.postgresql.org/media/keys/ACCC4CF8.asc]: PostgreSQL asc key URL
for Debian.
- **postgresql_debian_keys_dir** *[Default: /etc/apt/keyrings]*: Debian apt keys destination path.
- **postgresql_debian_key_destination** *[Default: `"{{ postgresql_debian_keys_dir }}/pgdg.asc"`]*: Debian apt keys full
path.
- **postgresql_debian_apt_repo_url_prefix** *[Default: http://apt.postgresql.org/pub/repos/apt/]*: Debian PostgreSQL
URL prefix for apt repo.
- **postgresql_debian_repo** *[Default:* 
```
  deb [signed-by={{ postgresql_debian_key_destination }}] {{ postgresql_debian_apt_repo_url_prefix }}
  {{ ansible_distribution_release }}-pgdg main
```
*]*: Debian PostgreSQL repo file content.
- **postgresql_become_user: postgres** *[Default: postgres]*: PostgreSQL become user to init Bareos database.
- **init_bareos_database** *[Default: False]*: Call database and tables creation scripts. Required for old versions of
Bareos or CentOS, because install scripts inside the packages are different.

License
-------
MIT-0

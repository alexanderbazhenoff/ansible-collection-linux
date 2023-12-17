postgresql
==========

Installs and manages PostgreSQL database instance.

Specification
-------------

### 1. Install, uninstall and configuring.

This role provides the following install, uninstall and/or configuring operations:

- Install or uninstall a specified version of PostgreSQL database server (or database instance). When specified
PostgreSQL version repository is not available for your Linux distribution, this will install a default (or recommended)
version for your Linux distribution (see [example playbooks](#1-install-or-uninstall-postgresql-server-instance) for 
details).
- Install or uninstall [pgadmin4](https://www.pgadmin.org/) alongside or single from database instance: pgadmin4 web
version with Apache, pgadmin desktop version or both (see [example playbook tasks](#2-install-or-uninstall-pgadmin4) for
details).
- Applies firewall rules (ufw and firewalld) for database instance and/or pgadmin4 web version installation. Removes
firewall rules during uninstalling.
- Configure `pg_hba.conf` on already installed database instance (see [example playbook tasks](#3-configure-pghbaconf)).
- Configure `postgresql.conf` on already installed database instance 
([example playbook tasks](#4-configure-postgresqlconf)).

Limitations:

- Alpine Linux is not supported for pgadmin4 installation.
- Configuring iptables for Debian systems is not supported.

### 2. Management.

- Create, alter, or remove a user (role) from a PostgreSQL server instance (see
[example playbook tasks](#5-user--role--management) for details).
- Create, drop, dump, rename or restore PostgreSQL databases (see [example playbook tasks](#6-database-management) for
details).
- Add or remove PostgreSQL schemas ([example playbook tasks](#7-schemas-management)).
- Grant or revoke privileges on PostgreSQL database objects ([example playbook tasks](#8-privileges-management)).
- Add or remove replication slots from a PostgreSQL database ([example playbook tasks](#9-slots-management)).

Requirements
------------

- Any LTS version of Ubuntu. Any version of Debian, RHEL, Centos, Fedora, Oracle Linux and Alpine Linux.
- Before you run this role on Alpine Linux set your `/etc/apk/repositories` manually (read
[documentation](https://wiki.alpinelinux.org/wiki/Repositories)).

Dependencies
------------

- [community.general](https://docs.ansible.com/ansible/latest/collections/community/general/index.html) ansible
  collection to install/uninstall PostgreSQL packages on Alpine Linux.
- [community.postgresql](https://docs.ansible.com/ansible/latest/collections/community/postgresql/index.html) ansible
  collection to control database instance using this role.
- This role using 'sudo' become method, so Debian distribution should have them already pre-installed.

Example Playbooks
-----------------

#### 1. Install or uninstall PostgreSQL server instance

The shortest example playbook of database instance installation should look like:

    - hosts: all
      become: True
      become_method: sudo
      gather_facts: True
    
      tasks:

        - name: "Install PostgreSQL v15 instance(es)"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql

while uninstall of the same version playbook tasks should look like:

        - name: "Uninstall PostgreSQL v15 instance(es) without database directory clean-up"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: uninstall

Role checks that specified version of PostgreSQL server is available for your Linux distribution: if this version is not
available, this will install a default version from your distribution repositories or from dnf modules. If you wish to
install PostgreSQL of a default version for your linux destribution set `postgresql_recommended_version: True`:

        - name: "Install PostgreSQL of default version for your linux destribution"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            postgresql_recommended_version: True

More complex examples may include different customisations:

        - name: "Install PostgreSQL v16 with postgresql.conf change and some packages alongside with pgadmin4 web"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            postgresql_version: 16
            install_pgadmin: True
            pgadmin_email: john.doe@company.com
            pgadmin_password: some_password_here
            postgresql_additional_packages:
              - "postgresql{% if ansible_os_family == 'Debian' %}-client-{{ postgresql_version }}{% else -%}
                {{ postgresql_version }}-contrib{% endif %}"
            postgresql_conf:
              port: 5432
              max_connections: 1000
              superuser_reserved_connections: 10
              listen_addresses: "'*'"
              lc_messages: "'en_US.UTF-8'"

#### 2. Install or uninstall pgadmin4

You can also install pgadmin4 web version without database server instance:

        - name: "Install pgadmin4 web version"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: install
            role_subject: pgadmin
            pgadmin_email: john.doe@company.com
            pgadmin_password: some_password_here

Credentials required for pgadmin web login.

Install pgadmin4 desktop version:

        - name: "Install pgadmin4 desktop version"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: install
            role_subject: pgadmin
            pgadmin_installation_type: desktop
            pgadmin_email: john.doe@company.com
            pgadmin_password: some_password_here

To install pgadmin4 desktop locally run playbook from [examples folder](examples) with specified credentials variables,
e.g:

```bash
cd <postgresql_role_directory>
ansible-playbook -c local -i localhost, examples/install_pgadmin4_desktop.yml --extra-vars \
"ansible_python_interpreter=/usr/bin/python3 become_password=<your_local_user_password_for_sudo>"
```

#### 3. Configure pg_hba.conf

You can perform regex search and add line(s) to pg_hba.conf running the next playbook tasks, e.g:

        - name: "Perform regex search and add line(s) to pg_hba.conf"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: configure
            role_subject: hba_conf
            pg_conf_content_mode: regex
            hba_conf_content: |
              local   all             postgres                                trust

Replace mode will write the whole content of `hba_conf_content` variable to pg_hba.conf file:

        - name: "Configure the whole pg_hba.conf"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: configure
            role_subject: hba_conf
            pg_conf_content_mode: replace
            hba_conf_content: |
              local   all             postgres                                trust
              local   all             all                                     trust
              host    all             all             127.0.0.1/32            trust
              host    all             all             0.0.0.0/24              md5
              host    all             all             ::1/128                 trust
              host    all             all             0.0.0.0/0               md5
              local   replication     all                                     peer
              host    replication     all             127.0.0.1/32            trust
              host    replication     all             ::1/128                 trust
              host    replication     all             0.0.0.0/24              scram-sha-256
              host    replication     all             0.0.0.0/0               scram-sha-256

Or you can use templating from a specified location:

        - name: "Templating pg_hba.conf from /path/to/your/hba_conf/template/file"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: configure
            role_subject: hba_conf
            pg_conf_content_mode: /path/to/your/hba_conf/template/file

Using this role all `pg_hba.conf` change actions ends up with postgresql daemon restart.

Please note all settings above are not the only recommendations, check the
[official documentation](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html).

#### 4. Configure postgresql.conf

        - name: "Change postgresql.conf parameters"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: configure
            role_subject: postgresql_conf
            postgresql_conf:
              max_connections: 1000
              log_connections: 'yes'
              listen_addresses: "'*'"
              lc_messages: "'en_US.UTF-8'"

will set the next parameters to postgresql.conf file:

```ini
max_connections = 1000
log_connections = yes
listen_addresses = '*'
lc_messages = 'en_US.UTF-8'
```

All parameters will be written after the comments containing defaults. You can also split parameters and pass them into
playbook tasks how many times as you wish. If you set another value on existing parameter this will overwrite with the
last one.

Using this role all `postgresql.conf` change actions ends up with postgresql daemon restart.

#### 5. User (role) management

        - name: "Create django user (role) on PostgreSQL server instance"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: users
            postgresql_users:
              - name: django
                password: ceec4eif7ya
    
        - name: "Add a comment on django user"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: users
            postgresql_users:
              - name: django
                comment: This is a test user
    
        - name: "Create rails user, set MD5-hashed password, grant privs"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: users
            postgresql_users:
              - name: rails
                password: md59543f1d82624df2b31672ec0f7050460
                role_attr_flags: CREATEDB,NOSUPERUSER

        - name: "Connect to acme database and remove test user"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: drop
            role_subject: users
            postgresql_users:
              - name: test
                database: acme
                fail_on_user: False

        - name: "Connect to test database and remove an existing user's password"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: drop
            role_subject: users
            postgresql_users:
              - name: test
                database: test
                password: ""

#### 6. Database management

        - name: "Create a new database with name 'acme'"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: databases
            postgresql_databases:
              - name: acme

        # Note: If a template different from "template0" is specified,
        # encoding and locale settings must match those of the template.
        - name: "Create a new database with name 'acme' with specific encoding, locale and owner"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: databases
            postgresql_databases:
              - name: acme
                owner: user
                encoding: UTF-8
                lc_collate: de_DE.UTF-8
                lc_ctype: de_DE.UTF-8
                template: template0

        # Note: Default limit for the number of concurrent connections to
        # a specific database is "-1", which means "unlimited"
        - name: "Create a new databases with name 'acme' and 'django' which has a limit of 100 concurrent connections"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: databases
            postgresql_databases:
              - name: acme
                conn_limit: "100"
              - name: django
                conn_limit: "100"

        - name: "Dump an existing databases to a file"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: dump
            role_subject: databases
            postgresql_databases:
              - name: acme
                target: /tmp/acme.sql
              - name: django
                target: /tmp/django.sql

        - name: "Dump an existing database to a file excluding the test table"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: dump
            role_subject: databases
            postgresql_databases:
              - name: acme
                target: /tmp/acme.sql
                dump_extra_args: --exclude-table=test

        - name: "Dump an existing database to a file (with compression)"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: dump
            role_subject: databases
            postgresql_databases:
              - name: acme
                target: /tmp/acme.sql.gz

        - name: "Dump a single schema for an existing database"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: dump
            role_subject: databases
            postgresql_databases:
              - name: acme
                target: /tmp/acme.sql
                target_opts: "-n public"

        - name: "Dump only table1 and table2 from the acme database"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: dump
            role_subject: databases
            postgresql_databases:
              - name: acme
                target: /tmp/table1_table2.sql
                target_opts: "-t table1 -t table2"

        - name: "Dump an existing database using the directory format"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: dump
            role_subject: databases
            postgresql_databases:
              - name: acme
                target: /tmp/acme.dir

        # Note: In the example below, if database foo exists and has another tablespace
        # the tablespace will be changed to foo. Access to the database will be locked
        # until the copying of database files is finished.
        - name: "Create a new database called foo in tablespace bar"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: databases
            postgresql_databases:
              - name: foo
                tablespace: bar

        # Rename the database foo to bar.
        # If the database foo exists, it will be renamed to bar.
        # If the database foo does not exist and the bar database exists,
        # the module will report that nothing has changed.
        # If both the databases exist, an error will be raised.
        - name: "Rename the database foo to bar"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: rename
            role_subject: databases
            postgresql_databases:
              - name: foo
                target: bar

#### 7. Schemas management

        - name: "Create a new schema with name acme in test database"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: schemas
            postgresql_schemas:
              - database: test
                name: acme

        - name: "Create a new schema acme with a user bob who will own it"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: schemas
            postgresql_schemas:
              - name: acme
                owner: bob

        - name: "Drop schema 'acme' with cascade"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: drop
            role_subject: schemas
            postgresql_schemas:
              - name: acme
                cascade_drop: True

#### 8. Privileges management

        # On database "library":
        # GRANT SELECT, INSERT, UPDATE ON TABLE public.books, public.authors
        # TO librarian, reader WITH GRANT OPTION
        - name: "Grant privs to librarian and reader on database library"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                privs: SELECT,INSERT,UPDATE
                type: table
                objs: books,authors
                schema: public
                roles: librarian,reader
                grant_option: True

        - name: "Same as above leveraging default values"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                privs: SELECT,INSERT,UPDATE
                objs: books,authors
                roles: librarian,reader
                grant_option: true

        # REVOKE GRANT OPTION FOR INSERT ON TABLE books FROM reader
        # Note that role "reader" will be *granted* INSERT privilege itself if this
        # isn't already the case (since state: present).
        - name: "Revoke privs from reader"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                priv: INSERT
                obj: books
                role: reader
                grant_option: false

        # "public" is the default schema. This also works for PostgreSQL 8.x.
        - name: "REVOKE INSERT, UPDATE ON ALL TABLES IN SCHEMA public FROM reader"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: revoke
            role_subject: privileges
            postgresql_privileges:
              - database: library
                privs: INSERT,UPDATE
                objs: ALL_IN_SCHEMA
                role: reader

        - name: "GRANT ALL PRIVILEGES ON SCHEMA public, math TO librarian"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                privs: ALL
                type: schema
                objs: public,math
                role: librarian
        
        # Note the separation of arguments with colons.
        - name: "GRANT ALL PRIVILEGES ON FUNCTION math.add(int, int) TO librarian, reader"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                privs: ALL
                type: function
                obj: add(int:int)
                schema: math
                roles: librarian,reader
        
        # Note that group role memberships apply cluster-wide and therefore are not
        # restricted to database "library" here.
        - name: "GRANT librarian, reader TO alice, bob WITH ADMIN OPTION"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                type: group
                objs: librarian,reader
                roles: alice,bob
                admin_option: True
        
        # Note that here "db: postgres" specifies the database to connect to, not the
        # database to grant privileges on (which is specified via the "objs" param)
        - name: "GRANT ALL PRIVILEGES ON DATABASE library TO librarian"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: postgres
                privs: ALL
                type: database
                obj: library
                role: librarian
        
        # If objs is omitted for type "database", it defaults to the database
        # to which the connection is established
        - name: "GRANT ALL PRIVILEGES ON DATABASE library TO librarian"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                privs: ALL
                type: database
                role: librarian

        # Objs must be set, ALL_DEFAULT to TABLES/SEQUENCES/TYPES/FUNCTIONS
        # ALL_DEFAULT works only with privs=ALL
        # For specific
        - name: "ALTER DEFAULT PRIVILEGES ON DATABASE library TO librarian"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                objs: ALL_DEFAULT
                privs: ALL
                type: default_privs
                role: librarian
                grant_option: True

        # Objs must be set, ALL_DEFAULT to TABLES/SEQUENCES/TYPES/FUNCTIONS
        # ALL_DEFAULT works only with privs=ALL
        # For specific
        - name: "ALTER DEFAULT PRIVILEGES ON DATABASE library TO reader, step 1"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                objs: TABLES,SEQUENCES
                privs: SELECT
                type: default_privs
                role: reader
        
        - name: "ALTER DEFAULT PRIVILEGES ON DATABASE library TO reader, step 2"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                objs: TYPES
                privs: USAGE
                type: default_privs
                role: reader

        - name: "GRANT ALL PRIVILEGES ON FOREIGN DATA WRAPPER fdw TO reader"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: test
                objs: fdw
                privs: ALL
                type: foreign_data_wrapper
                role: reader
        
        # Available since community.postgresql 0.2.0
        - name: "GRANT ALL PRIVILEGES ON TYPE customtype TO reader"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: test
                objs: customtype
                privs: ALL
                type: type
                role: reader

        - name: "GRANT ALL PRIVILEGES ON FOREIGN SERVER fdw_server TO reader"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: test
                objs: fdw_server
                privs: ALL
                type: foreign_server
                role: reader

        # Grant 'execute' permissions on all functions in schema 'common' to role 'caller'
        - name: "GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA common TO caller"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - type: function
                privs: EXECUTE
                roles: caller
                objs: ALL_IN_SCHEMA
                schema: common
        
        # Available since collection version 1.3.0
        # Grant 'execute' permissions on all procedures in schema 'common' to role 'caller'
        # Needs PostreSQL 11 or higher and community.postgresql 1.3.0 or higher
        - name: "GRANT EXECUTE ON ALL PROCEDURES IN SCHEMA common TO caller"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - type: prucedure
                privs: EXECUTE
                roles: caller
                objs: ALL_IN_SCHEMA
                schema: common

        # ALTER DEFAULT PRIVILEGES FOR ROLE librarian IN SCHEMA library GRANT SELECT ON TABLES TO reader
        # GRANT SELECT privileges for new TABLES objects created by librarian as
        # default to the role reader.
        # For specific
        - name: "ALTER privs"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: library
                schema: library
                objs: TABLES
                privs: SELECT
                type: default_privs
                role: reader
                target_roles: librarian

        # ALTER DEFAULT PRIVILEGES FOR ROLE librarian IN SCHEMA library REVOKE SELECT ON TABLES FROM reader
        # REVOKE SELECT privileges for new TABLES objects created by librarian as
        # default from the role reader.
        # For specific
        - name: "ALTER privs"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: revoke
            role_subject: privileges
            postgresql_privileges:
              - database: library
                schema: library
                objs: TABLES
                privs: SELECT
                type: default_privs
                role: reader
                target_roles: librarian
        
        # Available since community.postgresql 0.2.0
        - name: "Grant type privileges for pg_catalog.numeric type to alice"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - type: type
                roles: alice
                privs: ALL
                objs: numeric
                schema: pg_catalog
                database: acme
        
        - name: "Alter default privileges grant usage on schemas to datascience"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: grant
            role_subject: privileges
            postgresql_privileges:
              - database: test
                type: default_privs
                privs: usage
                objs: schemas
                role: datascience

#### 9. Slots management

        - name: "Create physical_one physical slot if doesn't exist"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: slots
            postgresql_slots:
              - name: physical_one
                database: ansible
          become_user: postgres
        
        - name: "Remove physical_one slot if exists"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: drop
            role_subject: slots
            postgresql_slots:
              - name: physical_one
                database: ansible
          become_user: postgres
      
        - name: "Create logical_one logical slot to the database acme if doesn't exist"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: create
            role_subject: slots
            postgresql_slots:
              - name: logical_slot_one
                slot_type: logical
                output_plugin: custom_decoder_one
                database: "acme"
      
        - name: "Remove logical_one slot if exists from the cluster running on another host and non-standard port"
          ansible.builtin.include_role:
            name: alexanderbazhenoff.linux.postgresql
          vars:
            role_action: drop
            role_subject: slots
            postgresql_slots:
              - name: logical_one
                login_host: mydatabase.example.org
                port: 5433
                login_user: ourSuperuser
                login_password: thePassword

Role Variables
--------------

### Role parameters:

| Variable                       | Default | Comment                                                                                                                |
|--------------------------------|---------|------------------------------------------------------------------------------------------------------------------------|
| role_action                    | install | Role action: uninstall, install, configure, create, grant, drop, revoke, dump, rename, restore                         |
| role_subject                   | server  | Role subject to perform with: server, pgadmin, hba_conf, postgresql_conf, users, databases, privileges, schemas, slots |
| clean_install                  | True    | Perform clean install                                                                                                  |
 | postgresql_version             | 15      | Version of pgsql repository                                                                                            |
 | cleanup_data_directory         | True    | Clean-up PostgreSQL data directory                                                                                     |
 | postgresql_recommended_version | False   | Install recommended version for current Linux distribution (1)                                                         |
 | postgresql_additional_packages | []      | List of additional PostgreSQL related packages to install (2)                                                          |
 | install_psycopg2               | True    | Install [psycopg2](https://pypi.org/project/psycopg2/)                                                                 |
 | install_pgadmin                | False   | Install [pgadmin4](https://www.pgadmin.org/) for web mode alongside with postgresql server                             |
 | pgadmin_installation_type      | web     | pgadmin installation type: web or desktop. Leave them empty to install both web and desktop.                           |
 | firewall_control               | True    | Add or remove firewalld and/or ufw rules on install or uninstall. Will be skipped when firewall disabled.              |

* (1) Recommended version install based on distribution repository or 
[dnf modules](https://docs.fedoraproject.org/en-US/modularity/using-modules/). Useful when the current version of
PostgreSQL server instance is not available for your Linux distribution (e.g. check
[RedHat repository](https://www.postgresql.org/download/linux/redhat/)).

* (2) List of additional PostgreSQL related packages, e.g: 'postgresql-contrib' or 'postgresql14-contrib' for
`postgresql_recommended_version: True` (explore what packages available
[here](https://www.postgresql.org/download/linux/)). There is no automated version handling in this list, include 
versions to package names.

### PostgreSQL server (or database instance) parameters:

* **postgresql_default_timezone**. PostgreSQL timezone. Required for PostgreSQL v14+, especially for init server
instance data directory on Alpine Linux. You can change it later, e.g: `SET TIME ZONE 'America/Montreal';` and/or add:
`timezone: 'America/Montreal'` to `postgresql_conf` dict variable (See also a
[list of timezone names](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones)). Defaults: 'Europe/Moscow'

* **postgresql_conf** (dict). Parameters from postgresql.conf with the same name here (see
[documentation](https://www.postgresql.org/docs/current/config-setting.html)). Double quotes to keep single 
`'quotes of value'`. Defaults:
```yaml
  port: 5432
  max_connections: 100
  superuser_reserved_connections: 3
  listen_addresses: "'*'"
```
* **pg_conf_content_mode** (string). `pg_hba.conf` write content mode to perform configure hba_conf (see
[documentation](https://www.postgresql.org/docs/current/auth-pg-hba-conf.html)): `regex` (ensure line(s) presents) or 
`block` (replace original file with), otherwise set template filename here (from /templates sub-folder) or full path to
replace with (e.g. `pg_hba.conf.j2`). Defaults: `pg_hba.conf.j2`

* **hba_conf_content** (string). pg_hba.conf block or lines for regex mode. Defaults: `''`

### Pgadmin4 parameters:

Both parameters are affects only for pgadmin4 web version:

| Variable         | Default             | Comment                                              |
|------------------|---------------------|------------------------------------------------------|
| pgadmin_email    | my.name@company.com | pgadmin4 email for administrator login via web UI    |
| pgadmin_password | my_password         | pgadmin4 password for administrator login via web UI |                                          

### PostgreSQL management parameters:

#### User (role) management.

* **postgresql_users** (list of dicts). List of postgresql users (roles) parameters to perform role action (create,
alter or drop) from database instance (see also 
[postgresql_user module documentation](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_user_module.html)).
Defaults:
```yaml
  - name: username
    password: my_password
```

| Parameter           | Type      | Default | Comment                                                                                   |
|---------------------|-----------|---------|-------------------------------------------------------------------------------------------|
| name                | mandatory |         | Name of the user (role) to add or remove.                                                 |
| comment             | optional  | (omit)  | Adds a comment on the user (equivalent to the COMMENT ON ROLE statement)                  |  
| password            | optional  | (omit)  | Set the user's password                                                                   |
| conn_limit          | optional  | (omit)  | Specifies the user (role) connection limit.                                               |
| no_password_changes | optional  | False   | If True, does not inspect the database for password changes (1)                           |
 | database            | optional  | ''      | Name of database to connect to and where user's permissions are granted.                  |
 | encrypted           | optional  | True    | Whether the password is stored hashed in the database (2)                                 |
 | role_attr_flags     | optional  | ''      | PostgreSQL user attributes string in the format: CREATEDB,CREATEROLE,SUPERUSER (3)        |
 | ssl_mode            | optional  | prefer  | Determines how an SSL session is negotiated with the server (4)                           |
 | ca_cert             | optional  | ''      | Specifies the name of a file containing SSL certificate authority (CA) certificate(s) (5) |
 | trust_input         | optional  | True    | It makes sense to use False only when SQL injections through the options are possible (6) |

* (1) If true, does not inspect the database for password changes. If the user already exists, skips all password
related checks. Useful when `pg_authid` is not accessible (such as in AWS RDS). Otherwise, makes password changes as
necessary.
* (2) You can specify an unhashed password, and PostgreSQL ensures the stored password is hashed when `encrypted=True`
is set. If you specify a hashed password, the module uses it as-is, regardless of the setting of encrypted. Note:
Postgresql 10 and newer does not support unhashed passwords.
* (3) Note that `[NO]CREATEUSER` is deprecated. To create a simple role for using it like a group, use `NOLOGIN` flag.
See the full list of supported flags in documentation for your PostgreSQL version.
* (4) See [libpq C Library](https://www.postgresql.org/docs/current/static/libpq-ssl.html) for more
information on the modes. Default of prefer matches libpq default. Choices: "allow", "disable", "prefer", "require",
"verify-ca", "verify-full".
* (5) If the file exists, verifies that the server's certificate is signed by one of these authorities.
* (6) If False, checks whether values of options name, password, privs, expires, role_attr_flags, groups, comment,
session_role are potentially dangerous.

#### Database management.

* **postgresql_databases** (list of dicts). List of postgresql databases parameters to perform role action: create,
drop, dump, rename, restore (see also
[postgresql_db module documentation](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_db_module.html)).
Defaults:
```yaml
  - name: db_name
    owner: username
```

| Parameter       | Type      | Default | Comment                                                                      |
|-----------------|-----------|---------|------------------------------------------------------------------------------|
| name            | mandatory |         | Name of the database to add or remove                                        |
 | owner           | optional  | ''      | Name of the role to set as owner of the database                             |
 | encoding        | optional  | (omit)  | Encoding of the database                                                     |
 | lc_collate      | optional  | (omit)  | Collation order (LC_COLLATE) to use in the database (1)                      |
 | lc_ctype        | optional  | (omit)  | Specifies the database connection limit                                      |
 | template        | optional  | ''      | Template used to create the database                                         |
 | conn_limit      | optional  | (omit)  | Specifies the database connection limit                                      |
 | tablespace      | optional  | ''      | The tablespace to set for the database (2)                                   |
 | target          | optional  | ''      | File to back up or restore from. Used when role_action is dump or restore    |
 | target_opts     | optional  | ''      | Additional arguments for pg_dump or restore program (3)                      |
 | dump_extra_args | optional  | (omit)  | Provides additional arguments when role_action is `dump` (4)                 |
 | ssl_mode        | optional  | prefer  | Determines how an SSL session is negotiated with the server (5)              |
 | ca_cert         | optional  | ''      | Specifies the name of a file containing SSL certificate authority (CA) (6)   |
 | trust_input     | optional  | True    | It makes sense only when SQL injections through the options are possible (7) |

* (1) Collation order (LC_COLLATE) to use in the database must match collation order of a template database unless 
`template0` is used as template.
* (2) The tablespace to set for the database (see
[documentation](https://www.postgresql.org/docs/current/sql-alterdatabase.html)). If you want to move the database back
to the default tablespace, explicitly set this to pg_default.
* (3) Additional arguments for pg_dump or restore program (pg_restore or psql, depending on target's format). Used when
role_action is `dump` or `restore`.
* (4) Cannot be used with dump-file-format-related arguments like `–format=d`.
* (5) Determines whether or with what priority a secure SSL TCP/IP connection will be negotiated with the server. See
[libpq-ssl documentation] for more information on the modes. Default of `prefer` matches libpq default. Choices:
"allow", "disable", "prefer", "require", "verify-ca", "verify-full".
* (6) The name of a file containing SSL certificate authority (CA) certificate(s). If the file exists, the server's 
certificate will be verified to be signed by one of these authorities.
* (7) If `False`, check whether values of parameters owner, conn_limit, encoding, db, template, tablespace,
session_role are potentially dangerous. It makes sense to use `False` only when SQL injections via the parameters are
possible.

#### Schema management.

* **postgresql_schemas** (list of dicts). List of parameters to add or remove schemas (see also
[postgresql_schema module documentation](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_schema_module.html)).
Defaults:
```yaml
  - name: schema_name
    database: db_name
    owner: username
```

| Parameter    | Type      | Default  | Comment                                                                               |
|--------------|-----------|----------|---------------------------------------------------------------------------------------|
| name         | mandatory |          | Name of the schema to add or remove                                                   |
 | cascade_drop | optional  | (omit)   | Drop schema with CASCADE to remove child objects. Choices: False/True                 |
 | database     | optional  | postgres | Name of the database to connect to and add or remove the schema                       |
 | owner        | optional  | (omit)   | Name of the role to set as owner of the schema                                        |
 | ssl_mode     | optional  | prefer   | Determines how an SSL session is negotiated with the server                           |
 | ca_cert      | optional  | ''       | Specifies the name of a file containing SSL certificate authority (CA) certificate(s) |
 | trust_input  | optional  | True     | It makes sense only when SQL injections through the options are possible              |

#### Privileges management.

* **postgresql_privileges** (list of dicts). List of privileges to grant or revoke on database objects (see also
[postgresql_privs module documentation](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_privs_module.html)).
Defaults:
```yaml
  - database: db_name
    roles: username
    privs: ALL
```

| Parameter    | Type      | Default  | Comment                                                                                  |
|--------------|-----------|----------|------------------------------------------------------------------------------------------|
| database     | mandatory |          | Name of database to connect to                                                           |
| roles        | mandatory |          | Comma separated list of role (user/group) names to set permissions for (1)               |
| grant_option | optional  | (omit)   | Whether `role` may grant/revoke the specified privileges/group memberships to others (2) | 
| objs         | optional  | (omit)   | Comma separated list of database objects to set privileges on (3)                        |
| privs        | optional  | (omit)   | Comma separated list of privileges to grant/revoke                                       |
| schema       | optional  | (omit)   | Schema that contains the database objects specified via objs                             |
| target_roles | optional  | (omit)   | A list of existing role names to set as the default permissions for database objects (4) |
 | type         | optional  | (omit)   | Type of database object to set privileges on (5)                                         |
 | ssl_mode     | optional  | prefer   | Determines how an SSL session is negotiated with the server                              |
 | ca_cert      | optional  | ''       | Specifies the name of a file containing SSL certificate authority (CA) certificate(s)    |
 | trust_input  | optional  | True     | It makes sense only when SQL injections through the options are possible                 |

* (1) The special value `PUBLIC` can be provided instead to set permissions for the implicitly defined PUBLIC group.
* (2) Set to `False` to revoke GRANT OPTION, leave unspecified to make no changes. grant_option only has an effect if 
`role_action` is `grant`. Choices: `True`, `False`.
* (3) If type is `table`, `partition table`, `sequence`, `function` or `procedure`, the special value `ALL_IN_SCHEMA`
can be provided instead to specify all database objects of type in the schema specified via schema. (This also works
with PostgreSQL < 9.0). `procedure` is supported since PostgreSQL 11 and community.postgresql collection 1.3.0. If type
is `database`, this parameter can be omitted, in which case privileges are set for the database specified via 
*database*. If type is `function` or `procedure`, colons (":") in object names will be replaced with commas (needed to
specify signatures, see examples).
* (4) A list of existing role (user/group) names to set as the default permissions for database objects subsequently
created by them. Parameter *target_roles* is only available with `type=default_privs`.
* (5) The `type` choice is available since Ansible version 2.10. The `procedure` is supported since collection version
1.3.0 and PostgreSQL 11. Choices: "database", "default_privs", "foreign_data_wrapper", "foreign_server", "function",
"group", "language", "table" (default), "tablespace", "schema", "sequence", "type", "procedure".

#### Slots management.

* **postgresql_slots** (list of dicts). List of parameters to add or remove replication slots from a database (see also
[postgresql_slot module documentation](https://docs.ansible.com/ansible/latest/collections/community/postgresql/postgresql_slot_module.html)).
Defaults:
```yaml
  - name: physical_slot_one
    db: db_name
```

| Parameter           | Type      | Default  | Comment                                                                                    |
|---------------------|-----------|----------|--------------------------------------------------------------------------------------------|
| name                | mandatory |          | Name of the replication slot to add or remove                                              |
| db                  | optional  | (omit)   | Name of database to connect to                                                             |
| immediately_reserve | optional  | False    | Specifies that the LSN slot be reserved immediately, otherwise on the first connection (1) |
| slot_type           | optional  | physical | Slot type: logical or physical                                                             |
| output_plugin       | optional  | (omit)   | All logical slots must indicate which output plugin decoder they're using (2)              |
 | ssl_mode            | optional  | prefer   | Determines how an SSL session is negotiated with the server                                |
 | ca_cert             | optional  | ''       | Specifies the name of a file containing SSL certificate authority (CA) certificate(s)      |
 | trust_input         | optional  | True     | It makes sense only when SQL injections through the options are possible                   |

* (1) Optional parameter that when `True` specifies that the LSN for this replication slot be reserved immediately,
otherwise the default, `False`, specifies that the LSN is reserved on the first connection from a streaming replication
client. Is available from PostgreSQL version 9.6. Uses only with *slot_type=physical*. Mutually exclusive with 
*slot_type=logical*.
* (2) This parameter does not apply to physical slots. It will be ignored with *slot_type=physical*. If it wasn't set 
(ommited) *"test_decoding"* will be set by default.

License
-------

MIT-0

# Ansible Role: PostgreSQL Users

Manage PostgreSQL roles, database grants, schema/table privileges, default
privileges for future tables, and optional `pg_hba.conf` rules.

The role is designed to work with an already installed PostgreSQL server. It can
optionally create databases, but it does not install or configure PostgreSQL
itself.

## Requirements

- Ansible 2.14 or newer
- `community.postgresql` collection
- Local PostgreSQL admin access, usually through the `postgres` OS user and Unix
  socket
- A Python PostgreSQL driver on the managed host (`psycopg` or `psycopg2`)

The role checks for `psycopg`/`psycopg2` and installs
`postgres_users_driver_packages` when the driver is missing.

Install collection dependencies:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Installation

Install the required PostgreSQL collection:

```bash
ansible-galaxy collection install community.postgresql
```

Install the role from Ansible Galaxy:

```bash
ansible-galaxy role install Juiceltd.postgresql_users
```

## Role Variables

Defaults are defined in `defaults/main.yml`.

| Variable | Default | Description |
| --- | --- | --- |
| `postgres_version` | `"16"` | PostgreSQL version used to build the default Debian/Ubuntu `pg_hba.conf` path. |
| `postgres_users_login_unix_socket` | `"/var/run/postgresql"` | Unix socket path used for local PostgreSQL connections. |
| `postgres_users_login_user` | `"postgres"` | OS/database admin user used with `become_user` and module login. |
| `postgres_users_login_db` | `"postgres"` | Database used for admin queries. |
| `postgres_users_default_privs` | `true` | Apply default privileges for new tables unless overridden. |
| `postgres_users_default_privs_owners` | `[]` | Owners whose default privileges are altered. Empty means `postgres_users_login_user`. |
| `postgres_users_manage_pg_hba` | `false` | Manage `pg_hba.conf` entries for users with `login: true`. |
| `postgres_users_pg_hba_file` | `"/etc/postgresql/{{ postgres_version }}/main/pg_hba.conf"` | Path to `pg_hba.conf` when management is enabled. Override on non-Debian layouts. |
| `postgres_users_pg_hba_contype` | `"host"` | Default pg_hba connection type. |
| `postgres_users_pg_hba_databases` | `"all"` | Default pg_hba database selector. |
| `postgres_users_pg_hba_method` | `"md5"` | Default pg_hba authentication method. |
| `postgres_users_driver_packages` | `[python3-psycopg2]` | Packages installed when no Python PostgreSQL driver is found. |
| `postgres_users_create_databases` | `false` | Create databases listed in user `dbs` entries when an owner is provided. |
| `postgres_users_db_owners` | `[]` | Owner roles to create before optional database creation. |
| `postgres_users` | `[]` | PostgreSQL users/roles and their per-database grants. |

## User Definition

Each item in `postgres_users` defines one PostgreSQL role.

```yaml
postgres_users:
  - name: analytics_ro
    password: "{{ vault_postgres_analytics_ro_password }}"
    login: true
    search_path: ["analytics", "public"]
    pg_hba:
      - address: "10.0.0.0/8"
        method: "scram-sha-256"
    dbs:
      - name: appdb
        revoke_unspecified_schemas: true
        grants:
          - type: database
            privs: [CONNECT]
          - type: schema
            schemas: ["analytics"]
            privs: [USAGE]
          - type: table
            schemas: ["analytics"]
            objs: ALL_IN_SCHEMA
            privs: [SELECT]
            default_privs: true
            default_privs_owners: ["app_owner"]
```

User fields:

- `name` is required.
- `password` is required. Store it in Ansible Vault or another secrets manager.
- `login` defaults to `true`. Set to `false` for non-login roles.
- `search_path` is optional and can be a string or list.
- `pg_hba` is optional and used only when `postgres_users_manage_pg_hba: true`.
- `dbs` is required and must contain at least one database entry.

## Database Entries

Each `dbs` item supports:

- `name` is required.
- `owner` is optional and is used only when `postgres_users_create_databases:
  true`.
- `schemas` enables minimal mode.
- `grants` enables explicit custom grants.
- `revoke_unspecified_schemas` defaults to `true`.
- `default_privs` overrides `postgres_users_default_privs` for this database.
- `default_privs_owners` overrides
  `postgres_users_default_privs_owners` for this database.

The role skips grant management for missing databases unless database creation is
enabled and the database item has an explicit `owner`.

## Minimal Mode

If a database item uses `schemas` and does not define `grants`, the role grants:

- `CONNECT` on the database
- `USAGE` on each listed schema
- `SELECT` on all current tables in each listed schema
- `SELECT` default privileges for future tables when default privileges are
  enabled

```yaml
postgres_users:
  - name: analytics_ro
    password: "{{ vault_postgres_analytics_ro_password }}"
    dbs:
      - name: appdb
        schemas:
          - name: analytics
```

## Custom Grants

Use `grants` when you need exact database, schema, or table privileges. When
`grants` is present, it defines all privileges the role applies for that
database entry.

Supported grant types:

- `type: database` with `privs`, commonly `[CONNECT]`, `[CREATE]`, or `[TEMP]`.
- `type: schema` with `schemas` and `privs`, commonly `[USAGE]` or `[CREATE]`.
- `type: table` with `schemas`, optional `objs`, and table `privs`.

For table grants, `objs` defaults to `ALL_IN_SCHEMA`. You can also pass a list of
table names.

```yaml
postgres_users:
  - name: analytics_rw
    password: "{{ vault_postgres_analytics_rw_password }}"
    dbs:
      - name: appdb
        grants:
          - type: database
            privs: [CONNECT]
          - type: schema
            schemas: ["analytics"]
            privs: [USAGE, CREATE]
          - type: table
            schemas: ["analytics"]
            privs: [SELECT, INSERT, UPDATE, DELETE]
            default_privs: true
            default_privs_owners: ["app_owner"]
```

If no `type: database` grant is defined, the role grants `CONNECT` by default.

## Default Privileges

Default privileges are applied with `ALTER DEFAULT PRIVILEGES` through
`community.postgresql.postgresql_privs`.

Owner selection order:

1. Grant-level `default_privs_owners`
2. Database-level `default_privs_owners`
3. Global `postgres_users_default_privs_owners`
4. `postgres_users_login_user`

Set `postgres_users_default_privs: false` to disable default privileges globally.

## pg_hba Management

`pg_hba.conf` management is disabled by default. Enable it explicitly:

```yaml
postgres_users_manage_pg_hba: true
postgres_users_pg_hba_file: "/etc/postgresql/16/main/pg_hba.conf"
postgres_users_pg_hba_method: "scram-sha-256"
```

Each user `pg_hba` item is passed to
`community.postgresql.postgresql_pg_hba`. The role supplies defaults for
`contype`, `databases`, `users`, and `method`; rule entries can override them.
When rules change, the role reloads PostgreSQL with `SELECT pg_reload_conf()`.

## Example Playbook

```yaml
---
- name: Manage PostgreSQL users
  hosts: database_servers
  become: true
  roles:
    - role: postgresql_users
      vars:
        postgres_users_manage_pg_hba: true
        postgres_users_pg_hba_method: "scram-sha-256"
        postgres_users_default_privs_owners: ["app_owner"]
        postgres_users_db_owners:
          - name: app_owner
            password: "{{ vault_postgres_app_owner_password }}"
        postgres_users:
          - name: analytics_ro
            password: "{{ vault_postgres_analytics_ro_password }}"
            login: true
            search_path: ["analytics", "public"]
            pg_hba:
              - address: "10.0.0.0/8"
            dbs:
              - name: appdb
                revoke_unspecified_schemas: true
                grants:
                  - type: database
                    privs: [CONNECT]
                  - type: schema
                    schemas: ["analytics", "reports"]
                    privs: [USAGE]
                  - type: table
                    schemas: ["analytics", "reports"]
                    objs: ALL_IN_SCHEMA
                    privs: [SELECT]
                    default_privs: true
```

## Security Notes

- Do not commit plaintext database passwords. Use Ansible Vault, CI secrets, or
  another secrets manager.
- Tasks that receive role passwords use `no_log: true` so passwords are not
  printed by Ansible.
- `revoke_unspecified_schemas` defaults to `true` and revokes privileges on
  schemas not listed in `schemas` or `grants`, including `public`. Set it to
  `false` for databases where the role should only add grants.
- Review `postgres_users_pg_hba_file` before enabling pg_hba management,
  especially on distributions where PostgreSQL stores config files outside the
  Debian/Ubuntu default path.

## Full Example (All Features)

```yaml
---
- name: Manage PostgreSQL users with all features
  hosts: database_servers
  become: true
  roles:
    - role: postgresql_users
      vars:
        postgres_users_login_unix_socket: "/var/run/postgresql"
        postgres_users_login_user: "postgres"
        postgres_users_login_db: "postgres"
        postgres_users_default_privs: true
        postgres_users_default_privs_owners: ["app_owner", "reporting_owner"]
        postgres_users_create_databases: true
        postgres_users_db_owners:
          - name: app_owner
            password: "{{ vault_postgres_app_owner_password }}"
          - name: reporting_owner
            password: "{{ vault_postgres_reporting_owner_password }}"
        postgres_users:
          - name: analytics_ro
            password: "{{ vault_postgres_analytics_ro_password }}"
            login: true
            search_path: ["analytics", "public"]
            pg_hba:
              - address: "10.0.0.0/8"
            dbs:
              - name: appdb
                owner: app_owner
                revoke_unspecified_schemas: true
                default_privs: true
                default_privs_owners: ["app_owner"]
                grants:
                  - type: database
                    privs: [CONNECT]
                  - type: schema
                    schemas: ["analytics", "reports"]
                    privs: [USAGE]
                  - type: table
                    schemas: ["analytics", "reports"]
                    objs: ALL_IN_SCHEMA
                    privs: [SELECT]
                    default_privs: true
                    default_privs_owners: ["app_owner"]
              - name: metricsdb
                owner: app_owner
                revoke_unspecified_schemas: true
                grants:
                  - type: database
                    privs: [CONNECT]
                  - type: schema
                    schemas: ["metrics"]
                    privs: [USAGE]
                  - type: table
                    schemas: ["metrics"]
                    objs: ["users", "orders", "payments"]
                    privs: [SELECT]

          - name: analytics_rw
            password: "{{ vault_postgres_analytics_rw_password }}"
            login: true
            search_path: ["analytics", "public"]
            pg_hba:
              - address: "10.0.0.0/8"
              - address: "192.168.10.0/24"
                databases: "mydb"
                method: "scram-sha-256"
            dbs:
              - name: appdb
                revoke_unspecified_schemas: true
                grants:
                  - type: database
                    privs: [CONNECT]
                  - type: schema
                    schemas: ["analytics"]
                    privs: [USAGE, CREATE]
                  - type: table
                    schemas: ["analytics"]
                    objs: ALL_IN_SCHEMA
                    privs: [SELECT, INSERT, UPDATE, DELETE]
                    default_privs: true
                    default_privs_owners: ["app_owner"]

          - name: reporting_user
            password: "{{ vault_postgres_reporting_user_password }}"
            login: true
            pg_hba:
              - address: "10.0.0.0/8"
                databases: "mydb,reporting,analytics"
                method: "scram-sha-256"
            dbs:
              - name: appdb
                revoke_unspecified_schemas: false
                grants:
                  - type: database
                    privs: [CONNECT]
                  - type: schema
                    schemas: ["reports"]
                    privs: [USAGE]
                  - type: table
                    schemas: ["reports"]
                    objs: ALL_IN_SCHEMA
                    privs: [SELECT]
                    default_privs: true
                    default_privs_owners: ["reporting_owner"]
              - name: logsdb
                owner: reporting_owner
                revoke_unspecified_schemas: true
                grants:
                  - type: database
                    privs: [CONNECT]
                  - type: schema
                    schemas: ["public"]
                    privs: [USAGE]
                  - type: table
                    schemas: ["public"]
                    objs: ALL_IN_SCHEMA
                    privs: [SELECT]
```

## License

MIT

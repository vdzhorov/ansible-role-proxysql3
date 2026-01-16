# Ansible Role: ProxySQL3

End-to-end installation and configuration of **ProxySQL 3** on Ubuntu/Debian:

- Install ProxySQL 3 from official repo
- Configure `mysql-interfaces` (multi-port frontend)
- Configure monitoring credentials (`mysql-monitor_username/password`)
- Manage backend servers (`mysql_servers`)
- (Optional) Manage Galera hostgroups (`mysql_galera_hostgroups`)
- Manage ProxySQL frontend users (`mysql_users`)
- Manage query rules (`mysql_query_rules`)
- Optional validation (ports + runtime tables)

## Requirements

### Ansible Collections Requirements

Install:

```bash
ansible-galaxy collection install community.proxysql community.mysql
```

## Notes / Behavior

- mysql-interfaces is written to disk and requires a ProxySQL restart. The role restarts only when the value changes (read -> compare -> set -> notify handler).
- Galera config is optional and controlled via proxysql3_enable_galera.
- Validation ports are derived automatically from proxysql_mysql_interfaces.

## Variables

### Admin endpoint

```yaml
proxysql_admin_host: "127.0.0.1"
proxysql_admin_port: 6032
proxysql_admin_user: "admin"
proxysql_admin_password: "{{ vault_proxysql_admin_password }}"
```

### Interfaces (multi-port support)

```yaml
proxysql_mysql_interfaces: "0.0.0.0:6033"
```

If you want multiple interfaces, separate them by semicolon, E.g. "0.0.0.0:6033;0.0.0.0:6034".

### Monitor credentials

```yaml
proxysql_monitor_username: "proxysql_monitor"
proxysql_monitor_password: "PASSWORD" # Change me.
```

### Backends

```yaml
proxysql_backend_servers:
  - hostgroup_id: 10
    hostname: "192.168.50.10"
    port: 3306
    comment: "percona-server1"
  - hostgroup_id: 10
    hostname: "192.168.50.11"
    port: 3306
    comment: "percona-server2"
  - hostgroup_id: 10
    hostname: "192.168.50.12"
    port: 3306
    comment: "percona-server3"
```

### Galera (optional)

```yaml
proxysql3_enable_galera: true
proxysql_galera_hostgroup:
  writer_hostgroup: 10
  backup_writer_hostgroup: 11
  reader_hostgroup: 20
  offline_hostgroup: 30
  max_transactions_behind: 100
  comment: "PXC main cluster"
```

### Frontend users (ProxySQL)

```yaml
proxysql_mysql_users:
  - name: appuser
    pass: "APPUSERPASS" # Change me.
    default_hostgroup: 10
    transaction_persistent: true
    comment: "App user via ProxySQL"
```

### Query rules (per port)

```yaml
proxysql_query_rules:
  - rule_id: 10
    active: true
    proxy_port: 6033
    match_pattern: ".*"
    destination_hostgroup: 10
    apply: true
    comment: "3326 -> HG10 writer"

  - rule_id: 20
    active: true
    proxy_port: 6034
    match_pattern: ".*"
    destination_hostgroup: 10
    apply: true
    comment: "3327 -> HG10 writer (read-your-writes)"
```

## Extra tuning: MySQL global variables (`mysql-*`)

The role can optionally apply additional **ProxySQL MySQL global variables** (the `mysql-*` variables in `global_variables`).
This is useful for standardizing behavior across environments (timeouts, ping intervals, version string, digest behavior, etc).

### How it works

- Variables are applied **only if** `proxysql3_mysql_global_variables` is non-empty.
- Each entry is set with `load_to_runtime: true` and `save_to_disk: true`.
- Restart-only variables are **blocked** in this mechanism:
  - `mysql-interfaces` (managed separately by the role with restart)
  - `mysql-threads`
  - `mysql-stacksize`

### Configuration example

```yaml
proxysql3_mysql_global_variables:
  mysql-server_version: "5.7.44"
  mysql-connect_timeout_client: "10000"
  mysql-connect_timeout_server: "3000"
  mysql-ping_interval_server_msec: "120000"
  mysql-ping_timeout_server: "500"
  mysql-shun_on_failures: "5"
  mysql-shun_recovery_time_sec: "10"
  mysql-query_rules_fast_routing_algorithm: "1"
  mysql-parse_failure_logs_digest: "true"
```

### Validation

```yaml
proxysql3_run_validation: true
# ports regex is derived automatically from proxysql_mysql_interfaces
```

## Example Playbook

```yaml
- name: ProxySQL 3 on internal LB
  hosts: proxysql
  become: true
  roles:
    - role: vdzhorov.proxysql3
```

## Run only a part (tags)

```bash
ansible-playbook -i inventory playbooks/proxysql.yml --tags proxysql3_install
ansible-playbook -i inventory playbooks/proxysql.yml --tags proxysql3_config
ansible-playbook -i inventory playbooks/proxysql.yml --tags proxysql3_servers
ansible-playbook -i inventory playbooks/proxysql.yml --tags proxysql3_users
ansible-playbook -i inventory playbooks/proxysql.yml --tags proxysql3_query_rules
ansible-playbook -i inventory playbooks/proxysql.yml --tags proxysql3_validate
```

## Example minimal vars

```yaml
---
## Example: your `group_vars/proxysql.yml` (minimal)

proxysql_admin_password: "admin"
proxysql_monitor_password: "MONITORPASS" # Change me.

proxysql_mysql_interfaces: "0.0.0.0:6033"

proxysql_backend_servers:
  - hostgroup_id: 10
    hostname: "192.168.50.10"
    port: 3306
    comment: "percona-server1"
  - hostgroup_id: 10
    hostname: "192.168.50.11"
    port: 3306
    comment: "percona-server2"
  - hostgroup_id: 10
    hostname: "192.168.50.12"
    port: 3306
    comment: "percona-server3"

proxysql_mysql_users:
  - name: appuser
    pass: "APPUSERPASS" # Change me.
    default_hostgroup: 10
    transaction_persistent: true
    comment: "App user via ProxySQL"

proxysql_query_rules:
  - rule_id: 10
    active: true
    proxy_port: 6033
    match_pattern: ".*"
    destination_hostgroup: 10
    apply: true
    comment: "3326 -> HG10 writer"

  - rule_id: 20
    active: true
    proxy_port: 6034
    match_pattern: ".*"
    destination_hostgroup: 10
    apply: true
    comment: "3327 -> HG10 writer (read-your-writes)"
```

# Prometheus HA Monitoring Stack

## Project structure

```
prometheus-ha/
├── ansible.cfg
├── ssh_config                        # SSH config with ProxyJump via bastion
├── site.yml                          # Full stack deployment
├── inventory/
│   ├── hosts.yml                     # SOURCE OF TRUTH — edit here to add/remove targets
│   ├── group_vars/
│   │   ├── all/
│   │   │   ├── main.yml              # Global defaults (versions, ports, settings)
│   │   │   └── vault.yml             # Encrypted secrets (Ansible Vault)
│   │   └── monitoring.yml            # Monitoring-node-specific vars
│   └── host_vars/
│       ├── pNode1.yml                # Per-host Node Exporter scrape config
│       ├── pNode2.yml                # Per-host labels, Node Exporter, and Postgres config
│       ├── bob.yaml                  # Example custom_service scrape config
│       ├── pNode3.yml
│       ├── pNode4.yml
│       ├── maas_host.yml             # Per-host MAAS scrape config
│       ├── proxmox_fake_node.yml     # Per-node Proxmox scrape config
│       ├── zs4128-a.yml              # Per-switch SNMP scrape config
│       ├── zs4128-b.yml
│       └── zs4128-c.yml
├── roles/
│   ├── prometheus/                   # Prometheus binary + config + rules
│   ├── alertmanager/                 # Alertmanager + cluster config
│   ├── grafana/                      # Grafana + datasource + dashboard provisioning
│   ├── haproxy/                      # HAProxy frontend for Grafana
│   ├── node_exporter/                # Node Exporter for Linux hosts
│   ├── postgres_exporter/            # postgres_exporter for database hosts
│   ├── proxmox_exporter/             # prometheus-pve-exporter (runs on monitoring nodes)
│   └── snmp_exporter/                # snmp_exporter (runs on monitoring nodes)
└── playbooks/
    ├── update-prometheus-config.yml  # Fast re-render and reload (use after inventory change)
    ├── smoke-test.yml                # Validate stack health post-deploy
    └── test-ha-failover.yml          # HA failure scenario test
```

## Quick start

### 0. Environment settings

- Controller Ansible: `ansible-core 2.12.10`
- Controller Python: `Python 3.8.20`
- Monitor node OS target: `Ubuntu Server 22.04` (Jammy)

### 1. Update inventory

Edit `inventory/hosts.yml` to reflect your environment. The IPs in the file
are placeholders — replace with your actual host addresses.

For each scrape target, keep target-specific scrape settings in
`inventory/host_vars/<host>.yml`.
For database hosts, also set `postgres_exporter_dsn` in the host_vars file.
For Proxmox, keep per-node scrape settings in host_vars and keep API token
credentials in `inventory/group_vars/all/main.yml` with secret token values
coming from Vault.
For services that already expose their own Prometheus exporter, add them to
`custom_service` and configure scraping in `inventory/host_vars/<host>.yml`.

### 1.1 SSH jump host (required for internal nodes)

This project uses repository-local `ssh_config` and forces Ansible to use it
via `ansible.cfg` (`ssh_args = -F ./ssh_config ...`).

Before running playbooks, edit `ssh_config` and set:

- `Host bastion` -> real bastion `HostName`
- bastion `User`
- bastion/private key `IdentityFile`

All `10.8.50.*` targets will then connect through `ProxyJump bastion`.

### 2. Sensitive values — Ansible Vault

Never commit credentials to source control. Use Vault:

(Vault default Password abcd1234@)

```bash
# Create vault file
ansible-vault create inventory/group_vars/all/vault.yml

# Inside vault.yml:
vault_grafana_admin_password: "your-secure-password"
vault_proxmox_fake_node_api_token_value: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
vault_postgres_exporter_password: "exporterpassword"
vault_alertmanager_smtp_auth_username: "youraccount@gmail.com"
vault_alertmanager_smtp_auth_password: "your-gmail-app-password"
```

Reference in `group_vars/all/main.yml`:
```yaml
grafana_admin_password: "{{ vault_grafana_admin_password }}"
proxmox_api_credentials:
  proxmox_fake_node:
    user: "prometheus@pve"
    token_name: "monitoring"
    token_value: "{{ vault_proxmox_fake_node_api_token_value }}"
    verify_ssl: false
alertmanager_smtp_auth_username: "{{ vault_alertmanager_smtp_auth_username }}"
alertmanager_smtp_auth_password: "{{ vault_alertmanager_smtp_auth_password }}"
```

### 3. Full deployment

```bash
ansible-playbook site.yml
# With vault:
ansible-playbook site.yml --ask-vault-pass
```

### 4. Validate deployment

```bash
ansible-playbook playbooks/smoke-test.yml
```

### 4.1 Run only on monitoring nodes

Use this when you only changed Prometheus / Alertmanager / Grafana / exporters
that run on monitor nodes, and do not want to touch monitored hosts.

```bash
# Full monitoring stack on monitoring nodes only
ansible-playbook site.yml --limit monitoring

# Same, with Vault
ansible-playbook site.yml --limit monitoring --ask-vault-pass

# Fast config refresh after inventory/rule/template updates
ansible-playbook playbooks/update-prometheus-config.yml --limit monitoring
```

### 5. Adding a new monitored host

Add the host to the relevant group(s) in `inventory/hosts.yml`, then run:

```bash
# Deploy exporters to the new host
ansible-playbook site.yml --limit <new-host> --tags exporters

# Re-render Prometheus config on monitoring nodes to pick up the new target
ansible-playbook playbooks/update-prometheus-config.yml
```

That's it. No manual edits to `prometheus.yml`.

### 5.1 Per-target scrape configuration

Prometheus scrape jobs are rendered from
`roles/prometheus/templates/prometheus.yml.j2`. These inventory groups use
per-host files under `inventory/host_vars/`:

- `linux_nodes`
- `postgres_servers`
- `proxmox_nodes`
- `maas_controllers`
- `snmp_switches`
- `custom_service`

For `linux_nodes`, use:

```yaml
node_exporter_job_name: node_pNode1
node_exporter_port: 9100
node_exporter_scheme: http
node_exporter_metrics_path: /metrics
node_exporter_scrape_interval: 15s
# node_exporter_scrape_timeout: 10s
# node_exporter_basic_auth_username: "{{ vault_pNode1_node_exporter_username }}"
# node_exporter_basic_auth_password: "{{ vault_pNode1_node_exporter_password }}"
# node_exporter_basic_auth_password_hash: "{{ vault_pNode1_node_exporter_password_hash }}"
# node_exporter_bearer_token: "{{ vault_pNode1_node_exporter_bearer_token }}"
# node_exporter_tls_insecure_skip_verify: false
# node_exporter_tls_ca_file: /etc/prometheus/certs/pNode1-node-exporter-ca.crt
# node_exporter_tls_cert_file: /etc/prometheus/certs/pNode1-node-exporter-client.crt
# node_exporter_tls_key_file: /etc/prometheus/certs/pNode1-node-exporter-client.key

node_exporter_labels:
  role: linux
```

When `node_exporter_basic_auth_username` is defined with either
`node_exporter_basic_auth_password` or `node_exporter_basic_auth_password_hash`,
the `node_exporter` role automatically:

- creates `/etc/node_exporter/web.yml`
- adds `--web.config.file=/etc/node_exporter/web.yml` to the systemd unit
- restarts `node_exporter`

Recommended pNode1 setup:

```yaml
# inventory/host_vars/pNode1.yml
node_exporter_basic_auth_username: "{{ vault_pNode1_node_exporter_username }}"
node_exporter_basic_auth_password_hash: "{{ vault_pNode1_node_exporter_password_hash }}"
```

Generate the bcrypt hash on any machine with `htpasswd`:

```bash
htpasswd -nBC 12 prometheus
```

Put the username and hash in Vault:

```yaml
vault_pNode1_node_exporter_username: prometheus
vault_pNode1_node_exporter_password_hash: "$2y$12$..."
```

The role also supports this simpler form:

```yaml
node_exporter_basic_auth_username: "{{ vault_pNode1_node_exporter_username }}"
node_exporter_basic_auth_password: "{{ vault_pNode1_node_exporter_password }}"
```

In that mode, the target host installs `htpasswd` tooling and generates
`web.yml` the first time it is missing. To rotate a plaintext-managed password,
set `node_exporter_basic_auth_force_regenerate: true` for one run or remove
`/etc/node_exporter/web.yml` and rerun the role.

For `postgres_servers`, use:

```yaml
postgres_exporter_dsn: "postgresql://exporter:{{ postgres_exporter_password }}@localhost:5432/postgres?sslmode=disable"
postgres_exporter_job_name: postgres_pNode2
postgres_exporter_port: 9187
postgres_exporter_scheme: http
postgres_exporter_metrics_path: /metrics
postgres_exporter_scrape_interval: 15s
# postgres_exporter_scrape_timeout: 10s
# postgres_exporter_basic_auth_username: "{{ vault_pNode2_postgres_exporter_username }}"
# postgres_exporter_basic_auth_password: "{{ vault_pNode2_postgres_exporter_password }}"
# postgres_exporter_bearer_token: "{{ vault_pNode2_postgres_exporter_bearer_token }}"
# postgres_exporter_tls_insecure_skip_verify: false
# postgres_exporter_tls_ca_file: /etc/prometheus/certs/pNode2-postgres-exporter-ca.crt
# postgres_exporter_tls_cert_file: /etc/prometheus/certs/pNode2-postgres-exporter-client.crt
# postgres_exporter_tls_key_file: /etc/prometheus/certs/pNode2-postgres-exporter-client.key
```

For `proxmox_nodes`, use:

```yaml
# proxmox_module_name must match a key in proxmox_api_credentials.
proxmox_job_name: proxmox_fake_node
proxmox_module_name: proxmox_fake_node
proxmox_cluster: "1"
proxmox_node: "1"
proxmox_metrics_path: /pve
proxmox_scrape_interval: 15s
# proxmox_scrape_timeout: 10s
```

For `maas_controllers`, use:

```yaml
maas_job_name: maas_host
maas_port: 5240
maas_scheme: http
maas_metrics_path: /MAAS/metrics
maas_scrape_interval: 15s
# maas_scrape_timeout: 10s
# maas_basic_auth_username: "{{ vault_maas_username }}"
# maas_basic_auth_password: "{{ vault_maas_password }}"
# maas_bearer_token: "{{ vault_maas_bearer_token }}"
# maas_tls_insecure_skip_verify: false
# maas_tls_ca_file: /etc/prometheus/certs/maas-ca.crt
# maas_tls_cert_file: /etc/prometheus/certs/maas-client.crt
# maas_tls_key_file: /etc/prometheus/certs/maas-client.key
```

For `snmp_switches`, use:

```yaml
snmp_job_name: snmp_zs4128_a
snmp_module_name: if_mib
snmp_auth_name: v2_public
snmp_scrape_interval: 1h
snmp_scrape_timeout: 30s
# snmp_basic_auth_username: "{{ vault_zs4128_a_snmp_exporter_username }}"
# snmp_basic_auth_password: "{{ vault_zs4128_a_snmp_exporter_password }}"
# snmp_basic_auth_password_hash: "{{ vault_zs4128_a_snmp_exporter_password_hash }}"
# snmp_bearer_token: "{{ vault_zs4128_a_snmp_exporter_bearer_token }}"
# snmp_tls_insecure_skip_verify: false
# snmp_tls_ca_file: /etc/prometheus/certs/snmp-exporter-ca.crt
# snmp_tls_cert_file: /etc/prometheus/certs/snmp-exporter-client.crt
# snmp_tls_key_file: /etc/prometheus/certs/snmp-exporter-client.key
```

Most scrape target types support the same three HTTP authentication patterns:

- `*_basic_auth_username` and `*_basic_auth_password`
- `*_bearer_token`
- `*_tls_*` client/CA settings

For `snmp_exporter` Basic Auth, define both plaintext and hash values:

```yaml
# inventory/host_vars/zs4128-a.yml
snmp_basic_auth_username: "{{ vault_zs4128_a_snmp_exporter_username }}"
snmp_basic_auth_password: "{{ vault_zs4128_a_snmp_exporter_password }}"
snmp_basic_auth_password_hash: "{{ vault_zs4128_a_snmp_exporter_password_hash }}"
```

The plaintext password is used by Prometheus when scraping `snmp_exporter`.
The bcrypt hash is used by the `snmp_exporter` role to generate
`/etc/snmp_exporter/web.yml` on monitoring nodes. The role collects all
`snmp_basic_auth_username`/`snmp_basic_auth_password_hash` pairs from
`groups['snmp_switches']`.

Generate the hash with:

```bash
htpasswd -nBC 12 prometheus
```

Then store both in Vault:

```yaml
vault_zs4128_a_snmp_exporter_username: prometheus
vault_zs4128_a_snmp_exporter_password: "plain-password-used-by-prometheus"
vault_zs4128_a_snmp_exporter_password_hash: "$2y$12$..."
```

Proxmox is the exception: Prometheus scrapes the local `pve_exporter`, and
`pve_exporter` authenticates to Proxmox using the Proxmox API token configured in
`proxmox_api_credentials`. The `proxmox_module_name` in each host_vars file must
match one key in that group-vars map.

For multiple Proxmox servers with different API tokens:

```yaml
# inventory/group_vars/all/main.yml
proxmox_api_credentials:
  pve_site_a:
    user: "prometheus@pve"
    token_name: "monitoring-a"
    token_value: "{{ vault_pve_site_a_api_token_value }}"
    verify_ssl: false
  pve_site_b:
    user: "prometheus@pve"
    token_name: "monitoring-b"
    token_value: "{{ vault_pve_site_b_api_token_value }}"
    verify_ssl: true
```

```yaml
# inventory/host_vars/pve-site-a.yml
proxmox_module_name: pve_site_a
```

```yaml
# inventory/host_vars/pve-site-b.yml
proxmox_module_name: pve_site_b
```

The `*_bearer_token` values make Prometheus send an HTTP header like:

```text
Authorization: Bearer <token>
```

Store real token values in `inventory/group_vars/all/vault.yml`, not directly
in host_vars.

Important: the token in Prometheus only sends credentials. The exporter endpoint
must also be configured to validate that token. Many Prometheus exporters,
including common node/postgres/snmp exporter setups, do not enforce arbitrary
API bearer tokens by default. In practice, use one of these patterns:

- Put the exporter behind a reverse proxy that validates bearer tokens.
- Use exporter-supported web auth where available, commonly TLS or Basic Auth.
- Restrict exporter ports with firewall rules so only monitoring nodes can
  connect.

Bearer-token/API-token flow:

1. Store the token in Vault, for example
   `vault_zs4128_a_snmp_exporter_bearer_token`.
2. Reference it from host_vars as `snmp_bearer_token`.
3. Run `playbooks/update-prometheus-config.yml` so Prometheus sends
   `Authorization: Bearer <token>`.
4. Configure the exporter endpoint to validate that token. For exporters that
   do not validate bearer tokens themselves, put Nginx/HAProxy/Caddy in front
   of the exporter and validate the header there.

TLS client-certificate flow:

1. Issue a CA, server certificate for the exporter endpoint, and optionally a
   client certificate for Prometheus.
2. Install the CA/client files on every monitoring node at the paths referenced
   by host_vars, for example `snmp_tls_ca_file`, `snmp_tls_cert_file`, and
   `snmp_tls_key_file`.
3. Change the scrape scheme to `https` when the exporter endpoint serves TLS.
4. Configure the exporter endpoint, or a reverse proxy in front of it, to use
   the server certificate and require/verify client certificates if using mTLS.
5. Run `playbooks/update-prometheus-config.yml` to render Prometheus
   `tls_config`.

For SNMP specifically, these HTTP authentication settings protect access to the
`snmp_exporter` HTTP endpoint on the monitoring nodes. They do not replace the
SNMP v2/v3 credentials used by `snmp_exporter` to probe the switch; those remain
controlled by `snmp_auth_name` and `roles/snmp_exporter/templates/snmp.yml.j2`.

Manual checks look like this:

```bash
# Bearer token
curl -H "Authorization: Bearer ${TOKEN}" http://10.8.50.58:9100/metrics

# Basic Auth
curl -u "${USER}:${PASSWORD}" http://10.8.50.54:9187/metrics

# TLS with a trusted CA and optional client certificate
curl --cacert /etc/prometheus/certs/exporter-ca.crt \
  --cert /etc/prometheus/certs/exporter-client.crt \
  --key /etc/prometheus/certs/exporter-client.key \
  https://10.8.50.58:9100/metrics

# pve_exporter running on a monitoring node and querying a Proxmox target.
# The module name selects the API token block from proxmox_api_credentials.
curl "http://127.0.0.1:9221/pve?target=10.0.1.226&module=proxmox_fake_node&cluster=1&node=1"

# snmp_exporter running on a monitoring node and probing a switch
curl -H "Authorization: Bearer ${TOKEN}" \
  "http://127.0.0.1:9116/snmp?target=10.8.50.202&module=if_mib&auth=v2_public"
```

### 5.2 Deploy only HAProxy frontend

If you changed only HAProxy frontend settings:

```bash
ansible-playbook site.yml --limit haproxy --tags haproxy
```

### 5.3 Add a custom service exporter

Use the `custom_service` inventory group for hosts that already run an exporter.
This project does not deploy anything to those hosts; it only renders Prometheus
scrape jobs from host variables.

Add the host in `inventory/hosts.yml`:

```yaml
custom_service:
  hosts:
    bob:
      ansible_host: 13.48.17.69
```

Then create or update `inventory/host_vars/<host>.yml`:

```yaml
---
custom_service_job_name: custom_service_bob
custom_service_port: 9100
custom_service_scheme: http
custom_service_metrics_path: /metrics
custom_service_scrape_interval: 30s
custom_service_scrape_timeout: 10s

# Prefer Vault for real credentials.
# custom_service_basic_auth_username: "{{ vault_bob_exporter_username }}"
# custom_service_basic_auth_password: "{{ vault_bob_exporter_password }}"
# custom_service_bearer_token: "{{ vault_bob_exporter_bearer_token }}"

custom_service_labels:
  service: bob
  environment: custom
```

Apply the scrape config change:

```bash
ansible-playbook playbooks/update-prometheus-config.yml --limit monitoring --ask-vault-pass
```

The implementation lives in
`roles/prometheus/templates/prometheus.yml.j2`. Each host in `custom_service`
gets its own scrape job so credentials, scrape interval, timeout, scheme, and
metrics path can differ per host.

#### custom_service host_vars API

| Variable | Required | Default | Description |
|---|---:|---|---|
| `custom_service_job_name` | No | `custom_service_<host>` | Prometheus `job_name`. |
| `custom_service_target` | No | `<ansible_host>:<custom_service_port>` | Full target address. Use this to override host and port together. |
| `custom_service_port` | No | `9100` | Exporter port when `custom_service_target` is not set. |
| `custom_service_scheme` | No | `http` | Prometheus scrape scheme, usually `http` or `https`. |
| `custom_service_metrics_path` | No | `/metrics` | Metrics endpoint path. |
| `custom_service_scrape_interval` | No | `prometheus_scrape_interval` | Per-service scrape interval. |
| `custom_service_scrape_timeout` | No | unset | Per-service scrape timeout. |
| `custom_service_basic_auth_username` | No | unset | Basic Auth username. Requires password too. |
| `custom_service_basic_auth_password` | No | unset | Basic Auth password. Store real values in Vault. |
| `custom_service_bearer_token` | No | unset | Bearer token for token-authenticated exporters. Store real values in Vault. |
| `custom_service_tls_insecure_skip_verify` | No | unset | Sets Prometheus `tls_config.insecure_skip_verify`. |
| `custom_service_tls_ca_file` | No | unset | CA file path on the monitoring host. |
| `custom_service_tls_cert_file` | No | unset | Client cert file path on the monitoring host. |
| `custom_service_tls_key_file` | No | unset | Client key file path on the monitoring host. |
| `custom_service_params` | No | unset | Query params passed to the scrape endpoint. Values should be YAML lists. |
| `custom_service_labels` | No | unset | Extra labels added to this target. |
| `prometheus_labels` | No | unset | Existing shared per-host labels; also applied to custom service targets. |

## Architecture notes

### HA scraping

Both Prometheus nodes independently scrape all targets on the same interval.
There is no synchronisation between nodes — this is by design. Each maintains
its own local TSDB. A node failure leaves a gap in that node's data only;
the other node has continuous coverage.

### Alert deduplication

Both Prometheus nodes send firing alerts to both Alertmanager instances.
Alertmanager clusters via gossip protocol (Memberlist/SWIM). Deduplication
is by alert fingerprint (hash of label set, excluding value and timestamp).

**Critical**: do not include per-instance identity in alert rule labels.
The `external_labels` block differentiates TSDB data — it is stripped before
fingerprinting, so deduplication works correctly across both nodes.

### Grafana datasources

Grafana datasources are generated dynamically from `groups['monitoring']`.
For each monitoring node, one datasource is created:

- name: `Prometheus <inventory_hostname>`
- uid: `prometheus-<inventory_hostname>`
- url: `http://<ansible_host>:9090`

The first node in the `monitoring` group is set as default datasource.
Dashboards use a `$datasource` template variable so operators can switch
between Prometheus replicas quickly.

### Grafana via HAProxy

HAProxy runs on hosts in the `haproxy` inventory group and exposes a single
Grafana entrypoint (`haproxy_listen_port`, default `3000`). Backends are
generated automatically from `groups['monitoring']`, with health checks on
`/api/health`.

### Proxmox exporter placement

`prometheus-pve-exporter` runs on the monitoring nodes (not on Proxmox itself).
It queries the PVE API remotely using a read-only API token. Both monitoring
nodes run their own instance, so scraping continues if one monitoring node fails.

A dedicated read-only API token must be created in Proxmox before deploying:
- Datacenter → Permissions → API Tokens → Add
- Role: `PVEAuditor`
- Privilege separation: enabled

### MAAS metrics

MAAS exposes metrics directly on port 5240 at `/MAAS/metrics` — no separate
exporter is needed. Prometheus scrapes the region controller directly.

THIS IS A SUGGESTION!

## Useful PromQL queries

```promql
# CPU saturation across all nodes
1 - avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Disk space free percent, excluding tmpfs
100 * node_filesystem_avail_bytes{fstype!~"tmpfs|fuse.lxcfs"}
    / node_filesystem_size_bytes{fstype!~"tmpfs|fuse.lxcfs"}

# All targets currently down
up == 0

# PostgreSQL connection saturation
pg_stat_activity_count / pg_settings_max_connections

# Proxmox VMs that are offline
pve_up{id=~"qemu/.*"} == 0

# Rate of scrape errors per job (monitoring platform health)
rate(scrape_samples_post_metric_relabeling[5m])

# Compare data between replicas (run against each datasource)
count(up)
```

## Failure scenarios to test

| Scenario | Expected behaviour | How to test |
|---|---|---|
| One Prometheus node stops | Switch Grafana datasource to another monitoring node; alerts continue from remaining replicas | `playbooks/test-ha-failover.yml` |
| One Alertmanager stops | Alerts continue routing via surviving instance | `systemctl stop alertmanager` on one node |
| Scrape target goes down | `up == 0` fires `InstanceDown` after 2m | Stop `node_exporter` on any monitored host |
| Prometheus A restarts | TSDB gap visible on A; B has continuous data | Check both datasources in Grafana |
| Network partition between AMs | Both AMs may fire — intentional over-notification | Block port 9094 between monitoring nodes |

## Scaling considerations

Two monitoring nodes is sufficient for this environment. For larger scale:

- **More targets**: static inventory scales to ~1000 targets without issue.
  Beyond that, consider Prometheus service discovery (Consul, DNS, EC2).
- **Longer retention**: add Thanos sidecars + object storage. Thanos Query
  provides a merged deduplicated datasource for Grafana.
- **Geographic redundancy**: run a Prometheus pair per site, federate via
  Thanos or use remote_write to a central store.
- **Higher cardinality**: tune `--storage.tsdb.max-block-duration` and
  consider sharding by job if scrape durations become too long.

## SNMP switch metrics (rate-limited)

SNMP targets are scraped through `snmp_exporter` running on monitoring nodes.
Each switch is listed in `inventory/hosts.yml` under `snmp_switches`, and its
scrape settings live in `inventory/host_vars/<switch>.yml`.

Example:

```yaml
---
snmp_job_name: snmp_zs4128_a
snmp_module_name: if_mib
snmp_auth_name: v2_public
snmp_scrape_interval: 1h
snmp_scrape_timeout: 30s

snmp_labels:
  device: zs4128-a
  role: switch
```

To respect infrastructure limits, the current switch host_vars use:

- `scrape_interval: 1h`
- `scrape_timeout: 30s`

`snmp_auth_name` must reference an auth profile defined in the snmp_exporter
configuration, such as `v2_public` or `v3_default`.

Add/remove switches in `inventory/hosts.yml` under `snmp_switches`, and add a
matching file in `inventory/host_vars/` for each switch.

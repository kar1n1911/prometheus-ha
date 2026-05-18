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
│       ├── pNode2.yml                # Per-host labels and DSN
│       ├── bob.yaml                  # Example custom_service scrape config
│       ├── pNode3.yml
│       └── pNode4.yml
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

For database hosts, set `postgres_exporter_dsn` in `inventory/host_vars/<host>.yml`.
For Proxmox, set `proxmox_api_token_value` (store in Ansible Vault — see below).
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
vault_proxmox_api_token_value: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
vault_postgres_exporter_password: "exporterpassword"
vault_alertmanager_smtp_auth_username: "youraccount@gmail.com"
vault_alertmanager_smtp_auth_password: "your-gmail-app-password"
```

Reference in `group_vars/all/main.yml`:
```yaml
grafana_admin_password: "{{ vault_grafana_admin_password }}"
proxmox_api_token_value: "{{ vault_proxmox_api_token_value }}"
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

### 5.1 Deploy only HAProxy frontend

If you changed only HAProxy frontend settings:

```bash
ansible-playbook site.yml --limit haproxy --tags haproxy
```

### 5.2 Add a custom service exporter

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
To respect infrastructure limits, the `snmp_switches` job is configured with:

- `scrape_interval: 1h`
- `scrape_timeout: 30s`

Add/remove switches only in `inventory/hosts.yml` under `snmp_switches`.

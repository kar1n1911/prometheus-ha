# Prometheus HA Monitoring Stack

## Project structure

```
prometheus-ha/
├── ansible.cfg
├── site.yml                          # Full stack deployment
├── inventory/
│   ├── hosts.yml                     # SOURCE OF TRUTH — edit here to add/remove targets
│   ├── group_vars/
│   │   ├── all.yml                   # Global defaults (versions, ports, settings)
│   │   └── monitoring.yml            # Monitoring-node-specific vars
│   └── host_vars/
│       ├── db-01.yml                 # Per-host labels and DSN
│       └── db-02.yml
├── roles/
│   ├── prometheus/                   # Prometheus binary + config + rules
│   ├── alertmanager/                 # Alertmanager + cluster config
│   ├── grafana/                      # Grafana + datasource + dashboard provisioning
│   ├── node_exporter/                # Node Exporter for Linux hosts
│   ├── postgres_exporter/            # postgres_exporter for database hosts
│   └── proxmox_exporter/             # prometheus-pve-exporter (runs on monitoring nodes)
└── playbooks/
    ├── update-prometheus-config.yml  # Fast re-render and reload (use after inventory change)
    ├── smoke-test.yml                # Validate stack health post-deploy
    └── test-ha-failover.yml          # HA failure scenario test
```

## Quick start

### 1. Update inventory

Edit `inventory/hosts.yml` to reflect your environment. The IPs in the file
are placeholders — replace with your actual host addresses.

For database hosts, set `postgres_exporter_dsn` in `inventory/host_vars/<host>.yml`.
For Proxmox, set `proxmox_api_token_value` (store in Ansible Vault — see below).

### 2. Sensitive values — Ansible Vault

Never commit credentials to source control. Use Vault:

```bash
# Create vault file
ansible-vault create inventory/group_vars/vault.yml

# Inside vault.yml:
vault_grafana_admin_password: "your-secure-password"
vault_proxmox_api_token_value: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
vault_postgres_exporter_password: "exporterpassword"
```

Reference in `group_vars/all.yml`:
```yaml
grafana_admin_password: "{{ vault_grafana_admin_password }}"
proxmox_api_token_value: "{{ vault_proxmox_api_token_value }}"
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

### 5. Adding a new monitored host

Add the host to the relevant group(s) in `inventory/hosts.yml`, then run:

```bash
# Deploy exporters to the new host
ansible-playbook site.yml --limit <new-host> --tags exporters

# Re-render Prometheus config on monitoring nodes to pick up the new target
ansible-playbook playbooks/update-prometheus-config.yml
```

That's it. No manual edits to `prometheus.yml`.

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

Two named datasources are provisioned: `Prometheus A` (default) and `Prometheus B`.
All dashboards use a `$datasource` template variable — operators can switch the
entire dashboard to the other node with one dropdown selection. This is both a
failover mechanism and a useful cross-check tool.

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

# Compare data between Prometheus A and B (run against each datasource)
count(up)
```

## Failure scenarios to test

| Scenario | Expected behaviour | How to test |
|---|---|---|
| Prometheus A stops | Grafana on datasource B unaffected; alerts continue from B | `playbooks/test-ha-failover.yml` |
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

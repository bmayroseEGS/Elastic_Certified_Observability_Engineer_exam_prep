# 01 - Collecting System and Host Metrics with Elastic Agent

## Overview

Elastic Agent collects system and host metrics through the **System integration** and other host-level integrations. Metrics are shipped to Elasticsearch as data streams and visualized in the Infrastructure UI and Metrics Explorer.

---

## Key Concepts

- Metrics are stored in `metrics-*` data streams (e.g., `metrics-system.cpu-default`)
- The **System integration** covers CPU, memory, disk, network, and process metrics
- Elastic Agent replaces standalone Metricbeat for metric collection
- Metrics collection intervals are configurable per integration

---

## System Integration

The System integration is the primary source of host-level metrics.

### Datasets Collected

| Dataset | Metrics |
|---|---|
| `system.cpu` | CPU usage (user, system, idle, iowait, steal) |
| `system.memory` | RAM usage, swap, page faults |
| `system.disk` | Disk I/O (read/write bytes, operations) |
| `system.filesystem` | Disk space (used, free, total) per mount point |
| `system.network` | Network I/O (bytes in/out, packets, errors) |
| `system.load` | System load averages (1m, 5m, 15m) |
| `system.process` | Per-process CPU, memory, file descriptors |
| `system.process.summary` | Count of processes by state (running, sleeping, zombie) |
| `system.uptime` | System uptime |
| `system.socket_summary` | Open socket counts by protocol |

---

## Adding the System Integration in Fleet

1. **Kibana → Fleet → Agent Policies** → select or create a policy
2. **Add integration** → search for **"System"**
3. Configure collection intervals and enabled datasets:

| Setting | Default | Description |
|---|---|---|
| CPU collection interval | `10s` | How often CPU metrics are collected |
| Memory collection interval | `10s` | How often memory metrics are collected |
| Process collection interval | `10s` | How often process metrics are collected |
| Process include top N | `false` | Collect top N processes by CPU/memory |

4. **Save and deploy** — enrolled agents begin collecting immediately

---

## Collection Intervals

Shorter intervals = more granular data but higher storage and ingest costs.

```yaml
# Example: set a 30s collection interval for all system metrics
period: 30s
```

Recommended intervals:
- **Production monitoring**: 10–30s for CPU/memory, 60s for disk/network
- **Capacity planning**: 60s–5m is sufficient
- **Exam default**: assume `10s` unless told otherwise

---

## Key Metric Fields (ECS + System Integration)

### CPU
| Field | Description |
|---|---|
| `system.cpu.total.pct` | Total CPU usage percentage (0.0–1.0) |
| `system.cpu.user.pct` | User-space CPU usage |
| `system.cpu.system.pct` | Kernel-space CPU usage |
| `system.cpu.iowait.pct` | Time waiting on I/O |
| `system.cpu.cores` | Number of CPU cores |

### Memory
| Field | Description |
|---|---|
| `system.memory.used.pct` | Memory used percentage (0.0–1.0) |
| `system.memory.used.bytes` | Memory used in bytes |
| `system.memory.total` | Total memory in bytes |
| `system.memory.free` | Free memory in bytes |
| `system.memory.swap.used.pct` | Swap usage percentage |

### Disk I/O
| Field | Description |
|---|---|
| `system.disk.read.bytes` | Bytes read from disk |
| `system.disk.write.bytes` | Bytes written to disk |
| `system.disk.name` | Disk device name (e.g., `sda`) |

### Filesystem
| Field | Description |
|---|---|
| `system.filesystem.used.pct` | Disk space used percentage |
| `system.filesystem.used.bytes` | Disk space used in bytes |
| `system.filesystem.total` | Total disk space |
| `system.filesystem.mount_point` | Mount point (e.g., `/`, `/data`) |

### Network
| Field | Description |
|---|---|
| `system.network.in.bytes` | Bytes received |
| `system.network.out.bytes` | Bytes sent |
| `system.network.in.errors` | Receive errors |
| `system.network.name` | Network interface name (e.g., `eth0`) |

### Process
| Field | Description |
|---|---|
| `system.process.cpu.total.pct` | Process CPU usage |
| `system.process.memory.rss.bytes` | Resident memory size |
| `system.process.name` | Process name |
| `system.process.pid` | Process ID |
| `system.process.state` | Process state (running, sleeping, etc.) |

### Host Metadata (added automatically)
| Field | Description |
|---|---|
| `host.name` | Hostname |
| `host.ip` | Host IP addresses |
| `host.os.name` | OS name (e.g., `Ubuntu`) |
| `host.os.version` | OS version |
| `host.architecture` | CPU architecture (e.g., `x86_64`) |

---

## Data Stream Naming

System metrics land in:
```
metrics-system.<dataset>-<namespace>

Examples:
  metrics-system.cpu-default
  metrics-system.memory-default
  metrics-system.filesystem-default
  metrics-system.network-default
```

---

## Verifying Metric Collection

### Via API
```json
# Check a data stream exists and has documents
GET metrics-system.cpu-default/_count

# Sample the latest CPU metric
GET metrics-system.cpu-default/_search
{
  "size": 1,
  "sort": [{ "@timestamp": "desc" }]
}
```

### Via Kibana
- **Observability → Infrastructure** → host should appear in the inventory
- **Observability → Metrics Explorer** → select `system.cpu.total.pct` and group by `host.name`

---

## Other Host-Level Metric Integrations

Beyond the System integration, other integrations add application-level metrics:

| Integration | Metrics Collected |
|---|---|
| **Docker** | Container CPU, memory, network, disk I/O |
| **Nginx** | Request rate, active connections, error rate |
| **MySQL** | Query rate, connections, InnoDB metrics |
| **Redis** | Memory usage, connected clients, ops/sec |
| **Elasticsearch** | Cluster health, shard counts, JVM metrics |
| **AWS** | EC2, RDS, ELB, S3 via CloudWatch |

---

## Exam Tips

- Know the key System integration datasets: cpu, memory, filesystem, disk, network, process
- `system.cpu.total.pct`, `system.memory.used.pct`, and `system.filesystem.used.pct` are the most exam-relevant fields
- Metrics data stream naming: `metrics-<integration>.<dataset>-<namespace>`
- The System integration is added via Fleet → Agent Policy → Add integration
- Collection intervals are per-dataset and configurable — default is `10s`
- Host metadata (`host.name`, `host.ip`, etc.) is added automatically by the agent

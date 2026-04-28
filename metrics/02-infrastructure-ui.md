# 02 - Infrastructure UI and Inventory Views

## Overview

The **Infrastructure UI** in Kibana (Observability → Infrastructure) provides a unified view of your hosts, containers, pods, and services. It is the primary tool for infrastructure health monitoring and capacity analysis in the Elastic Certified Observability Engineer exam.

---

## Key Concepts

- The Infrastructure UI requires metrics collected by the **System integration** (or compatible integrations)
- It provides both **inventory views** (current state) and **metric trends** (over time)
- Hosts, Kubernetes nodes/pods, and Docker containers each have dedicated views
- Clicking any entity drills down into detailed metric charts and correlated logs

---

## Navigating the Infrastructure UI

**Kibana → Observability → Infrastructure**

### Top-Level Tabs

| Tab | Description |
|---|---|
| **Hosts** | All monitored hosts and their current health |
| **Kubernetes** | Cluster, node, pod, and container views |
| **AWS** | EC2 instances and other AWS services (if AWS integration enabled) |

---

## Hosts View

### Inventory Panel
The default view shows all hosts as tiles or a table, colored by a selected metric:

- **Tile view** — quick visual scan; color intensity reflects metric severity
- **Table view** — sortable columns for CPU, memory, disk, network

### Coloring Metric
Select the metric used to color the inventory tiles:

| Metric | Field |
|---|---|
| CPU usage | `system.cpu.total.pct` |
| Memory usage | `system.memory.used.pct` |
| Disk usage | `system.filesystem.used.pct` |
| Network inbound | `system.network.in.bytes` |
| Network outbound | `system.network.out.bytes` |
| Load (1m) | `system.load.1` |

### Filtering the Inventory
Use the search bar to filter by any field:
```
host.name: web-*
host.os.name: "Ubuntu"
cloud.region: us-east-1
```

### Time Range
The time picker controls which time window is used to compute the displayed metrics. The inventory shows the **average** of the selected metric over the chosen time range.

---

## Host Detail View

Click any host tile or row to open the **Host Detail** panel.

### Available Metric Charts

| Chart | Description |
|---|---|
| CPU usage | Total, user, system breakdown over time |
| Memory usage | Used vs available memory |
| Load | 1m, 5m, 15m load averages |
| Disk I/O | Read and write throughput |
| Disk usage | Used percentage per filesystem |
| Network traffic | Inbound and outbound bytes/sec |
| Log rate | Log event rate from the same host |

### Correlated Logs
The Host Detail view includes a **Logs** tab showing the most recent logs from that host — no separate query needed. This links directly to Log Explorer filtered to `host.name: "<selected host>"`.

### Open in Metrics Explorer
Each chart has an **"Open in Metrics Explorer"** link for deeper ad-hoc analysis.

---

## Grouping and Sorting

In the Hosts table view, group and sort by:
- `host.name`
- `cloud.provider`
- `cloud.region`
- `cloud.availability_zone`
- `host.os.name`
- Custom metadata fields added via agent policy

### Example: Group by Cloud Region
Useful for spotting whether a performance issue is region-specific:
1. Hosts view → **Group by** → `cloud.region`
2. Each group shows aggregate metrics for that region

---

## Saving a View

Custom filters and groupings can be saved as a **Saved View** for quick access later:
- Set your filters and grouping
- Click **Save** (top right of inventory)
- Name the view (e.g., "Production Web Tier")

---

## Infrastructure Metadata Fields

The Infrastructure UI uses these fields to populate inventory and filters:

| Field | Description |
|---|---|
| `host.name` | Hostname |
| `host.ip` | Host IP |
| `host.os.name` | Operating system |
| `host.architecture` | CPU architecture |
| `cloud.provider` | Cloud provider (aws, gcp, azure) |
| `cloud.region` | Cloud region |
| `cloud.availability_zone` | Availability zone |
| `cloud.instance.id` | Cloud instance ID |
| `cloud.instance.name` | Cloud instance name |
| `cloud.machine.type` | Instance type (e.g., `t3.medium`) |

These are populated automatically by the `add_cloud_metadata` processor in Elastic Agent.

---

## Kubernetes View

The Kubernetes view requires the **Kubernetes integration** to be deployed on cluster nodes.

### Views Available

| View | Description |
|---|---|
| **Cluster overview** | Node count, pod count, CPU/memory usage |
| **Nodes** | Per-node CPU, memory, disk, pod count |
| **Pods** | Per-pod CPU and memory usage |
| **Containers** | Per-container resource usage |
| **Namespaces** | Resource usage grouped by namespace |

### Key Kubernetes Metric Fields

| Field | Description |
|---|---|
| `kubernetes.node.name` | Node name |
| `kubernetes.pod.name` | Pod name |
| `kubernetes.namespace` | Kubernetes namespace |
| `kubernetes.container.name` | Container name |
| `kubernetes.node.cpu.usage.pct` | Node CPU usage |
| `kubernetes.pod.cpu.usage.pct` | Pod CPU usage |
| `kubernetes.pod.memory.usage.bytes` | Pod memory usage |

---

## Alerts from Infrastructure UI

Create metric threshold alerts directly from the Infrastructure UI:
1. Hosts view → **Alerts** → **Create alert**
2. Select metric (e.g., CPU > 90% for 5 minutes)
3. Set notification action (email, Slack, PagerDuty)

This is a shortcut to the full alerting rule creation covered in `alerting/`.

---

## Practical Exam Tasks

### Find the host with the highest CPU usage
1. Observability → Infrastructure → Hosts
2. Table view → sort by **CPU usage** descending
3. Top row is the highest CPU consumer

### Investigate a host with high memory usage
1. Click the host tile → Host Detail panel
2. Review the **Memory usage** chart for the time range of interest
3. Click the **Logs** tab to correlate with log events at the same time

### Filter inventory to a specific environment
```
# In the search bar:
host.name: prod-*
```
Or use a custom field added via agent policy:
```
labels.environment: production
```

### Check disk space across all hosts
1. Hosts → Table view
2. Change the coloring metric to **Disk usage**
3. Sort by disk usage to find hosts approaching capacity

---

## Exam Tips

- Infrastructure UI = **Observability → Infrastructure** (not under Analytics)
- The inventory color reflects the **average** of the selected metric over the time range
- Host Detail view has a **Logs tab** — correlated logs without a separate query
- `cloud.*` fields are auto-populated by the `add_cloud_metadata` processor
- Know how to create an alert directly from the Infrastructure UI
- Kubernetes view requires the Kubernetes integration — separate from the System integration
- Saved Views preserve filters and groupings for recurring investigations

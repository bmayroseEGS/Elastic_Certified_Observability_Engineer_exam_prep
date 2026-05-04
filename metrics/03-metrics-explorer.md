# 03 - Metrics Explorer

## Overview

**Metrics Explorer** is a purpose-built ad-hoc analysis tool in the Infrastructure UI for charting and comparing metrics across hosts and services. It is ideal for quick investigations, capacity analysis, and building visualizations without needing to go to Lens or Discover.

**Kibana → Observability → Infrastructure → Metrics Explorer**

---

## Key Concepts

- Metrics Explorer queries `metrics-*` data streams directly
- It supports aggregating metrics (avg, max, min, sum, rate) over time
- Multiple metrics can be plotted on the same chart for correlation
- Results can be grouped by any field (e.g., `host.name`, `cloud.region`)
- Charts can be saved and added to dashboards

---

## Interface Overview

| Component | Description |
|---|---|
| **Metric selector** | Choose one or more metrics to plot (e.g., `system.cpu.total.pct`) |
| **Aggregation** | How to aggregate the metric: avg, max, min, sum, rate, cardinality |
| **Group by** | Split the chart by a field (e.g., one line per `host.name`) |
| **Filter bar** | KQL filter to scope the data (e.g., `cloud.region: us-east-1`) |
| **Time picker** | Time range for the chart |
| **Chart type** | Line or area chart |

---

## Selecting Metrics

Type any metric field name in the metric selector. Common choices:

| Metric Field | Description |
|---|---|
| `system.cpu.total.pct` | Total CPU usage (0.0–1.0) |
| `system.memory.used.pct` | Memory used percentage |
| `system.filesystem.used.pct` | Disk space used percentage |
| `system.network.in.bytes` | Network bytes received |
| `system.network.out.bytes` | Network bytes sent |
| `system.load.1` | 1-minute load average |
| `system.disk.read.bytes` | Disk read bytes |
| `system.disk.write.bytes` | Disk write bytes |
| `kubernetes.pod.cpu.usage.pct` | Kubernetes pod CPU usage |
| `kubernetes.pod.memory.usage.bytes` | Kubernetes pod memory usage |

---

## Aggregation Types

| Aggregation | Use Case |
|---|---|
| `avg` | Smoothed trend — best for CPU, memory percentages |
| `max` | Peak usage — best for catching spikes |
| `min` | Lowest value — useful for floor analysis |
| `sum` | Total across all grouped entities |
| `rate` | Per-second rate of change — best for bytes, packets, errors |
| `cardinality` | Count of unique values |

> **Tip:** Use `rate` for cumulative counters like `system.network.in.bytes` — it converts the raw counter to a per-second throughput value.

---

## Grouping

Group by a field to split the chart into one series per unique value.

### Common Group-By Fields

| Field | Result |
|---|---|
| `host.name` | One line per host |
| `cloud.region` | One line per cloud region |
| `cloud.availability_zone` | One line per AZ |
| `kubernetes.namespace` | One line per Kubernetes namespace |
| `kubernetes.node.name` | One line per K8s node |
| `data_stream.namespace` | One line per deployment namespace |

### Example: CPU Usage per Host
- Metric: `system.cpu.total.pct`
- Aggregation: `avg`
- Group by: `host.name`

Result: one line per host, showing average CPU usage over time — instantly spot which host is the outlier.

---

## Adding Multiple Metrics

Metrics Explorer supports plotting multiple metrics on the same chart for correlation:

**Example: Correlate CPU and Load**
1. Add metric: `system.cpu.total.pct` (avg)
2. Click **+ Add metric**
3. Add metric: `system.load.1` (avg)
4. Both plot on the same time axis — spikes in load should correlate with CPU spikes

**Example: Compare Network In vs Out**
1. `system.network.in.bytes` (rate)
2. `system.network.out.bytes` (rate)

---

## Filtering

Use KQL in the filter bar to scope your analysis:

```
# Scope to a specific host
host.name: "web-01"

# Scope to production hosts only
host.name: prod-*

# Scope to a specific cloud region
cloud.region: us-east-1

# Scope to Linux hosts
host.os.platform: linux

# Exclude a noisy host
NOT host.name: "legacy-host"
```

---

## Practical Analysis Workflows

### Identify the Most CPU-Intensive Host
1. Metric: `system.cpu.total.pct`, Aggregation: `max`
2. Group by: `host.name`
3. Change time range to last 24 hours
4. The line at the top is your most CPU-intensive host

### Spot a Memory Leak
1. Metric: `system.memory.used.pct`, Aggregation: `avg`
2. Group by: `host.name`
3. Look for a line that steadily trends upward over hours/days — that host has a memory leak

### Check Disk Space Approaching Capacity
1. Metric: `system.filesystem.used.pct`, Aggregation: `max`
2. Group by: `host.name`
3. Filter: `system.filesystem.used.pct > 0.80`
4. Any line near 1.0 (100%) needs attention

### Compare Network Throughput Across Regions
1. Metric: `system.network.in.bytes`, Aggregation: `rate`
2. Group by: `cloud.region`
3. Spot which region is receiving the most traffic

---

## Saving and Adding to Dashboards

1. Configure your chart (metrics, aggregation, group by, filter)
2. Click **Save** → name the chart
3. Optionally add it directly to an existing dashboard

Saved charts are accessible from the Infrastructure UI and can be embedded in dashboards for ongoing monitoring.

---

## Metrics Explorer vs Other Tools

| Tool | Best For |
|---|---|
| **Metrics Explorer** | Quick ad-hoc metric charting, grouping by field, multi-metric correlation |
| **Infrastructure UI (Hosts)** | Inventory view, current state, host detail drill-down |
| **Lens** | Custom visualizations for dashboards, complex aggregations |
| **Discover** | Raw document exploration, non-metric data |

---

## Exam Tips

- Metrics Explorer is at **Observability → Infrastructure → Metrics Explorer** — not under Analytics
- Use **`rate`** aggregation for cumulative counter fields (bytes, packets, errors)
- Use **`max`** instead of `avg` to catch spikes that would be smoothed out by averaging
- Group by `host.name` is the most common exam scenario — know how to set it up
- Multiple metrics on one chart = correlation analysis — a high-value exam skill
- KQL filters in Metrics Explorer work the same as in Discover and Log Explorer
- Saved charts from Metrics Explorer can be added directly to dashboards

# 05 - Kubernetes and Container Metrics

## Overview

Kubernetes and container metrics provide visibility into cluster health, node resource usage, pod performance, and container behavior. The **Kubernetes integration** for Elastic Agent is the primary way to collect these metrics and ship them to Elasticsearch.

---

## Key Concepts

- The Kubernetes integration collects metrics from the **kubelet API** and **kube-state-metrics**
- Elastic Agent is deployed as a **DaemonSet** so every node runs a collector
- Metrics land in `metrics-kubernetes.*` data streams
- The **Infrastructure UI → Kubernetes** view provides built-in visualization
- Container metrics require the Docker or containerd runtime to expose stats

---

## Deployment Architecture

```
Kubernetes Cluster
├── Node 1
│   ├── elastic-agent (DaemonSet pod)  ← collects kubelet metrics from this node
│   ├── App Pod A
│   └── App Pod B
├── Node 2
│   ├── elastic-agent (DaemonSet pod)
│   └── App Pod C
└── kube-state-metrics (Deployment)    ← cluster-level state metrics
    └── elastic-agent scrapes this once per cluster
```

---

## Setting Up the Kubernetes Integration

### Prerequisites
- Elastic Agent deployed as a DaemonSet in the cluster
- `kube-state-metrics` deployed in the cluster
- RBAC permissions for the agent to read node, pod, and container metrics

### Fleet Setup
1. Fleet → Agent Policies → Create policy (or use existing)
2. Add integration → **Kubernetes**
3. Configure:
   - **kubelet** endpoint: `https://${NODE_NAME}:10250` (auto-detected on DaemonSet)
   - **kube-state-metrics** endpoint: `http://kube-state-metrics:8080/metrics`
4. Select datasets to enable
5. Deploy the agent as a DaemonSet using the provided manifests

### Helm Deployment (recommended)
```bash
helm repo add elastic https://helm.elastic.co
helm install elastic-agent elastic/elastic-agent \
  --set agent.fleet.enabled=true \
  --set agent.fleet.url=https://<fleet-server-url>:8220 \
  --set agent.fleet.token=<enrollment-token>
```

---

## Kubernetes Datasets

| Dataset | Source | Description |
|---|---|---|
| `kubernetes.node` | kubelet | Node CPU, memory, filesystem, network |
| `kubernetes.pod` | kubelet | Pod CPU, memory, network |
| `kubernetes.container` | kubelet | Per-container CPU, memory, filesystem |
| `kubernetes.system` | kubelet | System container metrics (kubelet, runtime) |
| `kubernetes.volume` | kubelet | Persistent volume usage |
| `kubernetes.state_node` | kube-state-metrics | Node conditions, allocatable resources |
| `kubernetes.state_pod` | kube-state-metrics | Pod phase, conditions, restart count |
| `kubernetes.state_container` | kube-state-metrics | Container requested/limit resources |
| `kubernetes.state_deployment` | kube-state-metrics | Desired vs available replicas |
| `kubernetes.state_replicaset` | kube-state-metrics | ReplicaSet ready/desired counts |
| `kubernetes.state_statefulset` | kube-state-metrics | StatefulSet ready/desired counts |
| `kubernetes.state_daemonset` | kube-state-metrics | DaemonSet desired/scheduled/available counts |
| `kubernetes.event` | Kubernetes API | Cluster events (warnings, normal events) |

---

## Key Metric Fields

### Node Metrics
| Field | Description |
|---|---|
| `kubernetes.node.name` | Node name |
| `kubernetes.node.cpu.usage.nanocores` | CPU usage in nanocores |
| `kubernetes.node.cpu.allocatable.cores` | Allocatable CPU cores |
| `kubernetes.node.memory.usage.bytes` | Memory usage in bytes |
| `kubernetes.node.memory.allocatable.bytes` | Allocatable memory |
| `kubernetes.node.fs.used.bytes` | Filesystem used bytes |
| `kubernetes.node.fs.capacity.bytes` | Filesystem capacity |
| `kubernetes.node.network.rx.bytes` | Network bytes received |
| `kubernetes.node.network.tx.bytes` | Network bytes sent |

### Pod Metrics
| Field | Description |
|---|---|
| `kubernetes.pod.name` | Pod name |
| `kubernetes.namespace` | Kubernetes namespace |
| `kubernetes.pod.cpu.usage.nanocores` | Pod CPU usage |
| `kubernetes.pod.memory.usage.bytes` | Pod memory usage |
| `kubernetes.pod.memory.rss.bytes` | Pod RSS memory |
| `kubernetes.pod.network.rx.bytes` | Pod network bytes received |
| `kubernetes.pod.network.tx.bytes` | Pod network bytes sent |

### Container Metrics
| Field | Description |
|---|---|
| `kubernetes.container.name` | Container name |
| `kubernetes.container.cpu.usage.nanocores` | Container CPU usage |
| `kubernetes.container.memory.usage.bytes` | Container memory usage |
| `kubernetes.container.memory.limit.bytes` | Container memory limit |
| `kubernetes.container.memory.request.bytes` | Container memory request |
| `kubernetes.container.rootfs.used.bytes` | Container root filesystem usage |

### State Metrics (kube-state-metrics)
| Field | Description |
|---|---|
| `kubernetes.state_pod.status.phase` | Pod phase: Running, Pending, Failed, Succeeded |
| `kubernetes.state_pod.status.ready` | Pod ready condition (true/false) |
| `kubernetes.state_container.status.restarts` | Container restart count |
| `kubernetes.state_container.cpu.request.cores` | Requested CPU |
| `kubernetes.state_container.cpu.limit.cores` | CPU limit |
| `kubernetes.state_container.memory.request.bytes` | Requested memory |
| `kubernetes.state_container.memory.limit.bytes` | Memory limit |
| `kubernetes.state_deployment.replicas.available` | Available replicas |
| `kubernetes.state_deployment.replicas.desired` | Desired replicas |
| `kubernetes.state_node.status.ready` | Node ready condition |

---

## Infrastructure UI — Kubernetes View

**Observability → Infrastructure → Kubernetes**

### Views Available

| View | What It Shows |
|---|---|
| **Cluster overview** | Total nodes, pods, CPU/memory usage across the cluster |
| **Nodes** | Per-node CPU, memory, pod count, disk, network |
| **Pods** | Per-pod CPU, memory, grouped by namespace |
| **Containers** | Per-container resource usage |

### Filtering in Kubernetes View
```
# Scope to a namespace
kubernetes.namespace: production

# Scope to pods with high restart counts
kubernetes.state_container.status.restarts > 5

# Scope to a specific node
kubernetes.node.name: "worker-node-01"
```

---

## Docker / Container Metrics (Non-Kubernetes)

For standalone Docker hosts (not Kubernetes), use the **Docker integration**:

| Dataset | Description |
|---|---|
| `docker.container` | Container CPU, memory, network, disk I/O |
| `docker.cpu` | Per-CPU usage breakdown |
| `docker.memory` | Memory usage, cache, limits |
| `docker.network` | Network bytes in/out per container |
| `docker.diskio` | Disk read/write per container |

### Key Docker Fields
| Field | Description |
|---|---|
| `docker.container.name` | Container name |
| `docker.container.id` | Container ID |
| `docker.cpu.total.pct` | Total CPU usage percentage |
| `docker.memory.usage.pct` | Memory usage percentage |
| `docker.network.in.bytes` | Network bytes received |
| `docker.network.out.bytes` | Network bytes sent |

---

## Practical Exam Tasks

### Find pods with the most restarts
```
# In Discover or Metrics Explorer
kubernetes.state_container.status.restarts > 0
```
Sort by `kubernetes.state_container.status.restarts` descending.

### Check if a deployment is fully available
```
GET metrics-kubernetes.state_deployment-default/_search
{
  "query": {
    "term": { "kubernetes.deployment.name": "payment-service" }
  },
  "sort": [{ "@timestamp": "desc" }],
  "size": 1
}
```
Compare `replicas.desired` vs `replicas.available`.

### Find pods not in Running phase
```
kubernetes.state_pod.status.phase: (Pending OR Failed OR Unknown)
```

### Identify containers approaching memory limits
In Metrics Explorer:
- Metric 1: `kubernetes.container.memory.usage.bytes` (avg)
- Metric 2: `kubernetes.state_container.memory.limit.bytes` (avg)
- Group by: `kubernetes.container.name`
- Compare the two lines — containers where usage approaches limit are at risk of OOMKill

---

## Metadata Enrichment

The Kubernetes integration automatically adds metadata to all events:

| Field | Description |
|---|---|
| `kubernetes.labels.*` | Pod labels (e.g., `kubernetes.labels.app`) |
| `kubernetes.annotations.*` | Pod annotations |
| `kubernetes.replicaset.name` | Owning ReplicaSet |
| `kubernetes.deployment.name` | Owning Deployment |
| `kubernetes.statefulset.name` | Owning StatefulSet |
| `kubernetes.daemonset.name` | Owning DaemonSet |

This allows filtering and grouping by application label:
```
kubernetes.labels.app: payment-service
```

---

## Exam Tips

- Elastic Agent runs as a **DaemonSet** — one pod per node
- **kubelet** provides real-time usage metrics; **kube-state-metrics** provides desired/current state
- `kubernetes.state_container.status.restarts` is the key field for crash-looping containers
- Pod phase is in `kubernetes.state_pod.status.phase` — not in kubelet metrics
- Know the difference: `kubernetes.container.*` (actual usage) vs `kubernetes.state_container.*` (requested/limits/state)
- Use `kubernetes.labels.*` to filter by application across pods and deployments
- Docker integration is for standalone containers; Kubernetes integration is for cluster workloads

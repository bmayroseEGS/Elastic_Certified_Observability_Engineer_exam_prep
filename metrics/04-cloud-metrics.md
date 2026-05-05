# 04 - Cloud Provider Metrics (AWS, GCP, Azure)

## Overview

Elastic integrations collect metrics from major cloud providers — AWS, Google Cloud, and Azure. Cloud metrics are pulled from provider APIs (CloudWatch, Cloud Monitoring, Azure Monitor) and shipped to Elasticsearch as `metrics-*` data streams. This enables unified observability across on-prem and cloud infrastructure in a single Kibana instance.

---

## Key Concepts

- Cloud metrics are **pulled** from provider APIs, not pushed by an agent on the instance
- A dedicated **Elastic Agent** (or agentless integration) acts as the collector
- Credentials (API keys, service accounts, IAM roles) must be configured for the integration
- Cloud metadata (`cloud.provider`, `cloud.region`, `cloud.instance.id`) is added automatically

---

## AWS Metrics

### How It Works
The **AWS integration** uses the AWS CloudWatch API to pull metrics. Elastic Agent needs AWS credentials — either via:
- **IAM role** attached to the EC2 instance running the agent (recommended)
- **Access key + secret key** configured in the integration settings

### Setting Up the AWS Integration in Fleet
1. Fleet → Agent Policies → Add integration → **AWS**
2. Configure credentials (IAM role ARN or access key)
3. Select datasets to enable (EC2, RDS, ELB, S3, etc.)
4. Set the collection period (default: `300s` — CloudWatch standard resolution)

### Key AWS Datasets

| Dataset | Metrics |
|---|---|
| `aws.ec2` | CPU utilization, network in/out, disk read/write, status checks |
| `aws.rds` | CPU, connections, read/write IOPS, free storage, replica lag |
| `aws.elb` | Request count, latency, HTTP 4xx/5xx, healthy host count |
| `aws.alb` | Same as ELB plus target response time, unhealthy host count |
| `aws.s3` | Bucket size, number of objects, request count |
| `aws.sqs` | Messages sent/received/deleted, queue depth, message age |
| `aws.lambda` | Invocations, errors, duration, throttles, concurrent executions |
| `aws.cloudwatch` | Custom CloudWatch metrics via namespace configuration |

### Key AWS Metric Fields

| Field | Description |
|---|---|
| `aws.ec2.metrics.CPUUtilization.avg` | EC2 CPU usage percentage |
| `aws.ec2.metrics.NetworkIn.sum` | Bytes received by EC2 instance |
| `aws.ec2.metrics.NetworkOut.sum` | Bytes sent by EC2 instance |
| `aws.rds.metrics.CPUUtilization.avg` | RDS CPU usage |
| `aws.rds.metrics.FreeStorageSpace.avg` | RDS free disk space in bytes |
| `aws.rds.metrics.DatabaseConnections.avg` | Active DB connections |
| `aws.elb.metrics.HTTPCode_ELB_5XX_Count.sum` | ELB 5xx error count |
| `aws.elb.metrics.Latency.avg` | ELB average request latency |
| `cloud.instance.id` | EC2 instance ID |
| `cloud.region` | AWS region (e.g., `us-east-1`) |
| `cloud.availability_zone` | Availability zone |

### AWS CloudWatch Collection Period
- **Standard resolution**: 5-minute intervals (default, lower cost)
- **High resolution**: 1-minute intervals (costs more CloudWatch API calls)

```yaml
# In Fleet integration settings
period: 300s   # 5 minutes — standard resolution
# period: 60s  # 1 minute — high resolution
```

---

## Google Cloud (GCP) Metrics

### How It Works
The **GCP integration** uses the **Cloud Monitoring API** (formerly Stackdriver). Authentication requires a **service account** JSON key file with the `Monitoring Viewer` role.

### Setting Up the GCP Integration in Fleet
1. Fleet → Agent Policies → Add integration → **Google Cloud Platform**
2. Upload or paste the service account JSON credentials
3. Set the GCP project ID
4. Select datasets to enable

### Key GCP Datasets

| Dataset | Metrics |
|---|---|
| `gcp.compute` | VM CPU utilization, network in/out, disk I/O |
| `gcp.gke` | GKE cluster CPU, memory, pod count |
| `gcp.cloudsql` | Database CPU, connections, disk usage |
| `gcp.loadbalancing` | Request count, latency, error rate |
| `gcp.pubsub` | Message count, undelivered messages, subscription age |
| `gcp.storage` | Bucket request count, bytes received/sent |
| `gcp.firestore` | Read/write/delete operations |

### Key GCP Metric Fields

| Field | Description |
|---|---|
| `gcp.compute.instance.cpu.utilization.value` | VM CPU utilization (0.0–1.0) |
| `gcp.compute.instance.network.received_bytes_count.value` | Bytes received |
| `gcp.compute.instance.network.sent_bytes_count.value` | Bytes sent |
| `gcp.gke.node.cpu.allocatable_utilization.value` | GKE node CPU allocatable usage |
| `gcp.gke.pod.volume.used_bytes.value` | GKE pod volume usage |
| `cloud.project.id` | GCP project ID |
| `cloud.region` | GCP region (e.g., `us-central1`) |

---

## Azure Metrics

### How It Works
The **Azure integration** uses the **Azure Monitor API**. Authentication requires an **Azure Active Directory service principal** with the `Monitoring Reader` role on the subscription.

### Required Credentials
- Client ID (Application ID)
- Client Secret
- Tenant ID
- Subscription ID

### Setting Up the Azure Integration in Fleet
1. Fleet → Agent Policies → Add integration → **Azure**
2. Enter service principal credentials
3. Select datasets to enable
4. Set collection period

### Key Azure Datasets

| Dataset | Metrics |
|---|---|
| `azure.compute_vm` | VM CPU, network in/out, disk read/write |
| `azure.database_account` | Cosmos DB request count, latency, RU consumption |
| `azure.app_service` | HTTP requests, response time, CPU, memory |
| `azure.container_instance` | CPU, memory usage |
| `azure.load_balancer` | Packet count, byte count, health probe status |
| `azure.storage` | Blob/file/queue/table transactions, latency |

### Key Azure Metric Fields

| Field | Description |
|---|---|
| `azure.compute_vm.percentage_cpu.avg` | VM CPU usage percentage |
| `azure.compute_vm.network_in_total.sum` | Total bytes received |
| `azure.compute_vm.network_out_total.sum` | Total bytes sent |
| `azure.app_service.requests.total` | Total HTTP requests |
| `azure.app_service.average_response_time.avg` | Average HTTP response time (ms) |
| `cloud.provider` | Always `azure` |
| `cloud.region` | Azure region (e.g., `eastus`) |

---

## Data Stream Naming for Cloud Metrics

```
metrics-<integration>.<dataset>-<namespace>

Examples:
  metrics-aws.ec2-default
  metrics-aws.rds-production
  metrics-gcp.compute-default
  metrics-azure.compute_vm-default
```

---

## Viewing Cloud Metrics in Kibana

### Infrastructure UI
- **Observability → Infrastructure → AWS** tab (if AWS integration enabled)
- Shows EC2 instances in the inventory alongside on-prem hosts
- Filter by `cloud.provider: aws` to scope to cloud resources only

### Metrics Explorer
- Metric: `aws.ec2.metrics.CPUUtilization.avg`
- Group by: `cloud.availability_zone`
- Filter: `cloud.region: us-east-1`

### Discover
```
GET metrics-aws.ec2-default/_search
{
  "size": 1,
  "sort": [{ "@timestamp": "desc" }]
}
```

---

## Cross-Cloud Correlation

When running multi-cloud, correlate metrics using shared ECS fields:

```
# Find all high-CPU instances regardless of cloud provider
system.cpu.total.pct > 0.9 OR aws.ec2.metrics.CPUUtilization.avg > 90

# Group by cloud provider in Metrics Explorer
Group by: cloud.provider
```

---

## Exam Tips

- AWS credentials: prefer **IAM roles** over access keys when the agent runs on EC2
- GCP credentials: **service account JSON** with `Monitoring Viewer` role
- Azure credentials: **service principal** with `Monitoring Reader` role on the subscription
- Default AWS collection period is `300s` (5 min) — CloudWatch standard resolution
- Cloud metrics data stream naming: `metrics-<provider>.<dataset>-<namespace>`
- `cloud.provider`, `cloud.region`, `cloud.instance.id` are auto-populated ECS fields
- AWS, GCP, and Azure each have a dedicated integration in Fleet — they are separate from the System integration
- The **Infrastructure UI AWS tab** appears only when the AWS integration is active

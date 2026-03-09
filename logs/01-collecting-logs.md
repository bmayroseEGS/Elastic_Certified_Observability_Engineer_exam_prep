# 01 - Collecting Logs with Elastic Agent

## Overview

Elastic Agent is the unified way to collect logs in the Elastic Stack. It replaces standalone Filebeat and is managed through Fleet in Kibana.

---

## Key Concepts

### Elastic Agent vs Filebeat
- **Elastic Agent** is the preferred modern approach — single agent, centrally managed via Fleet
- **Filebeat** is the legacy standalone log shipper, still supported but not recommended for new deployments
- Elastic Agent uses **integrations** to define what data to collect

### Fleet
- Fleet is the Kibana UI for managing Elastic Agents at scale
- Agents are assigned to **Agent Policies** which define integrations and settings
- Changes to a policy are automatically pushed to all enrolled agents

---

## Agent Policy & Integrations

An **Agent Policy** contains one or more integrations. Each integration defines:
- What logs to collect (file paths, log format)
- How to ship them (output — Elasticsearch or Logstash)
- Any preprocessing (processors)

### Common Log Integrations
| Integration | Description |
|---|---|
| System | syslog, auth.log, system events |
| Nginx | access and error logs |
| Apache | access and error logs |
| AWS | CloudWatch, S3, VPC Flow |
| Custom Logs | any log file by path |

---

## Collecting Logs — Step by Step

### 1. Install Elastic Agent
```bash
# Download and extract Elastic Agent
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-<version>-linux-x86_64.tar.gz
tar xzvf elastic-agent-<version>-linux-x86_64.tar.gz
cd elastic-agent-<version>-linux-x86_64

# Enroll agent with Fleet (get enrollment token from Kibana Fleet UI)
sudo ./elastic-agent install \
  --fleet-server-es=https://<elasticsearch-url>:9200 \
  --fleet-server-service-token=<service-token> \
  --fleet-server-policy=<policy-id>
```

### 2. Create an Agent Policy in Fleet
- Kibana > Fleet > Agent Policies > Create policy
- Add integrations to the policy (e.g., System, Nginx, Custom Logs)

### 3. Add a Custom Logs Integration
- Integration: **Custom Logs**
- Set the log file path: `/var/log/myapp/*.log`
- Set the dataset name: `myapp.logs`
- Optionally add tags and processors

### 4. Enroll the Agent
- Kibana > Fleet > Enrollment Tokens > Copy token
- Run the install command on the host with the enrollment token

---

## Log File Path Patterns

Elastic Agent supports glob patterns for log paths:

```yaml
paths:
  - /var/log/nginx/access.log
  - /var/log/nginx/*.log
  - /var/log/myapp/**/*.log
```

---

## Processors

Processors can be applied at collection time to modify events before shipping:

```yaml
processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  - drop_fields:
      fields: ["agent.ephemeral_id"]
  - add_fields:
      target: ''
      fields:
        env: production
```

---

## Key Fields in Collected Logs

| Field | Description |
|---|---|
| `@timestamp` | Event timestamp |
| `message` | Raw log message |
| `log.file.path` | Source file path |
| `agent.name` | Collecting agent name |
| `host.name` | Source host |
| `data_stream.dataset` | Dataset name |
| `data_stream.namespace` | Namespace (default: `default`) |

---

## Data Streams

Logs collected by Elastic Agent are stored in **data streams**:

- Naming convention: `logs-<dataset>-<namespace>`
- Example: `logs-nginx.access-default`
- Managed by index lifecycle management (ILM) automatically

---

## Exam Tips

- Know the difference between Elastic Agent (Fleet-managed) and standalone Filebeat
- Understand Agent Policies and how integrations are assigned
- Know how to enroll an agent via Fleet UI and CLI
- Understand data stream naming: `logs-<dataset>-<namespace>`
- Know common processors and when to use them

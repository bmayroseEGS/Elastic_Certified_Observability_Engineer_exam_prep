# 04 - Custom Log Integrations

## Overview

When a built-in Elastic integration doesn't cover your log source, you create a **custom log integration** (also called a custom logs input). This is common for in-house applications, legacy systems, or any log file that doesn't have a prebuilt integration.

---

## Key Concepts

- A **custom logs input** tells Elastic Agent which files to tail and how to label the data
- Data lands in the `logs-*` data stream family by default: `logs-<dataset>-<namespace>`
- You can apply an ingest pipeline at the integration level to parse the raw message
- ECS fields (`host.*`, `agent.*`, `@timestamp`) are added automatically — you only need to parse your application-specific fields

---

## Creating a Custom Log Integration in Fleet

### Step-by-step

1. **Kibana → Fleet → Agent Policies** → select or create a policy
2. **Add integration** → search for **"Custom Logs"**
3. Configure the integration:

| Field | Description | Example |
|---|---|---|
| Log file path(s) | Glob patterns of files to tail | `/var/log/myapp/*.log` |
| Dataset name | Suffix for the data stream | `myapp.application` |
| Namespace | Deployment environment | `production` |
| Custom ingest pipeline | Pipeline to run on each document | `myapp-parse-pipeline` |
| Custom fields | Static key-value fields added to every doc | `service.name: myapp` |

4. **Save and deploy** the policy — enrolled agents pick up the change within seconds

### Resulting Data Stream
```
logs-myapp.application-production
```

---

## Custom Fields

Add static metadata at the integration level so every log document carries it without needing a pipeline processor:

```yaml
# In the Fleet UI "Custom fields" section:
service.name: payment-service
deployment.environment: production
team: platform
```

These are added as top-level fields on every document and are queryable in Log Explorer immediately.

---

## Applying an Ingest Pipeline

Attach a pipeline to parse `message` into structured fields:

1. Create the pipeline first (see `02-ingest-pipelines.md`)
2. In the Custom Logs integration config → **Advanced options** → **Custom ingest pipeline** → enter the pipeline name
3. Elastic Agent will route all documents through that pipeline before indexing

> **Tip:** Always add a Grok or Dissect processor in the pipeline to break up the raw `message` field. Otherwise all your logs are unsearchable free text.

---

## File Path Patterns

Elastic Agent supports glob patterns for log file paths:

| Pattern | Matches |
|---|---|
| `/var/log/myapp/*.log` | All `.log` files in the directory |
| `/var/log/myapp/app-*.log` | Rotation-friendly: `app-1.log`, `app-2.log` |
| `/var/log/**/*.log` | Recursive — all `.log` files under `/var/log/` |
| `/var/log/myapp/app.log*` | Current + rotated files (`app.log.1`, `app.log.2.gz`) |

> **Note:** Elastic Agent automatically handles log rotation — it tracks the inode, not just the filename, so rotated files don't produce duplicate events.

---

## Configuring via `elastic-agent.yml` (Advanced)

When not using Fleet (standalone mode), define inputs directly:

```yaml
inputs:
  - type: logfile
    streams:
      - paths:
          - /var/log/myapp/*.log
        parsers:
          - multiline:
              type: pattern
              pattern: '^\d{4}-\d{2}-\d{2}'
              negate: true
              match: after
        processors:
          - add_fields:
              target: ''
              fields:
                service.name: myapp
                deployment.environment: production
    data_stream:
      dataset: myapp.application
      namespace: default
```

---

## Data Stream Naming Convention

```
logs-<dataset>-<namespace>
     ^^^^^^^^   ^^^^^^^^^
     Integration  Environment
     dataset      (default, prod, staging)
     name

Example: logs-myapp.application-production
```

- **Dataset** should be dot-separated: `<integration>.<sub-type>` (e.g., `nginx.access`, `myapp.audit`)
- **Namespace** is free-form but conventionally matches environment or team

---

## Index Template and Mappings

Custom log data streams automatically use the `logs` index template, which provides:
- Dynamic mapping for ECS fields
- ILM (Index Lifecycle Management) policy — rolls over at 50 GB or 30 days by default
- `@timestamp` as the time field

To add custom field mappings (e.g., prevent a field from being analyzed):

```json
PUT _component_template/myapp-custom-mappings
{
  "template": {
    "mappings": {
      "properties": {
        "transaction.id": { "type": "keyword" },
        "response.time_ms": { "type": "long" }
      }
    }
  }
}
```

Then reference this component template in a custom index template that matches your data stream.

---

## Verifying Data Arrives

After deploying the integration:

```
# Check the data stream exists
GET _data_stream/logs-myapp.application-production

# Count documents
GET logs-myapp.application-production/_count

# Sample a document
GET logs-myapp.application-production/_search
{
  "size": 1,
  "sort": [{ "@timestamp": "desc" }]
}
```

In Kibana: **Log Explorer → select your dataset** from the integrations dropdown.

---

## Troubleshooting

| Symptom | Check |
|---|---|
| No data arriving | Agent status in Fleet (green?), file path correct, file permissions readable by agent |
| `message` field not parsed | Pipeline attached? Test pipeline with `POST _ingest/pipeline/<name>/_simulate` |
| Duplicate documents | Log rotation config — verify agent is tracking by inode |
| Wrong timestamp | Pipeline has `date` processor? `@timestamp` set correctly? |
| Data stream not created | Dataset name valid? Must be lowercase, no spaces |

---

## Exam Tips

- Know the **data stream naming pattern**: `logs-<dataset>-<namespace>`
- The **Custom Logs integration** in Fleet is the exam-standard way to onboard a new log source
- Custom fields added in Fleet UI → every document carries them automatically (no pipeline needed)
- Attaching a pipeline at the integration level is cleaner than using `on_failure` or agent-level processors
- Always verify with `_data_stream` API or Log Explorer after deploying

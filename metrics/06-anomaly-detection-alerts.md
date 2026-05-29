# 06 - Metric Anomaly Detection and Alerts

## Overview

Elastic provides two complementary approaches to alerting on metrics:

1. **Metric threshold alerts** — rule-based alerts triggered when a metric crosses a fixed value
2. **Anomaly detection** — machine learning-based alerts triggered when metric behavior deviates from its learned baseline

Both are configured in **Observability → Alerts** and integrate with action connectors for notifications.

---

## Metric Threshold Alerts

### What It Does
Fires an alert when a metric value crosses a static threshold for a defined duration.

### Creating a Metric Threshold Alert

**Observability → Alerts → Create rule → Metric threshold**

| Field | Description | Example |
|---|---|---|
| **Metric** | The field to evaluate | `system.cpu.total.pct` |
| **Aggregation** | How to aggregate the metric | `average`, `max`, `min`, `sum`, `cardinality` |
| **Comparator** | Threshold comparison | `>`, `>=`, `<`, `<=`, `between`, `outside range` |
| **Threshold** | The value to compare against | `0.9` (90% CPU) |
| **Duration** | How long the condition must be true | `5 minutes` |
| **Group by** | Split alert by field (one alert per group) | `host.name` |
| **Filter** | KQL to scope the data | `host.name: prod-*` |

### Example: Alert on High CPU per Host
```
Metric:      system.cpu.total.pct
Aggregation: average
Comparator:  >
Threshold:   0.9
Duration:    5 minutes
Group by:    host.name
Filter:      host.name: prod-*
```
This creates one alert instance per host when average CPU exceeds 90% for 5 consecutive minutes.

### Example: Alert on Low Disk Space
```
Metric:      system.filesystem.used.pct
Aggregation: max
Comparator:  >
Threshold:   0.85
Duration:    2 minutes
Group by:    host.name
```

### Example: Alert on High Memory Usage
```
Metric:      system.memory.used.pct
Aggregation: average
Comparator:  >
Threshold:   0.95
Duration:    3 minutes
Group by:    host.name
```

---

## Alert Status Lifecycle

| Status | Description |
|---|---|
| **Active** | Condition is currently met — alert is firing |
| **Recovered** | Condition no longer met — alert has resolved |
| **No data** | No data received for the metric in the evaluation window |

Actions can be triggered on **Active**, **Recovered**, and **No data** status changes independently.

---

## Action Connectors

Connectors define where alert notifications are sent.

### Supported Connectors

| Connector | Use Case |
|---|---|
| **Email** | Send alert details to an email address |
| **Slack** | Post a message to a Slack channel |
| **PagerDuty** | Create/resolve incidents in PagerDuty |
| **Webhook** | HTTP POST to any endpoint |
| **ServiceNow** | Create incidents in ServiceNow |
| **Jira** | Create issues in Jira |
| **Microsoft Teams** | Post to a Teams channel |

### Creating a Connector
**Stack Management → Connectors → Create connector**

Connectors are reusable — create once, reference in multiple rules.

### Attaching an Action to a Rule
In the rule creation wizard:
1. Scroll to **Actions**
2. Select a connector
3. Choose when to trigger: **Active**, **Recovered**, or **No data**
4. Customize the message body using action variables

### Action Variables (for message templates)
```
{{rule.name}}           - Name of the alert rule
{{context.group}}       - The group value (e.g., host name)
{{context.metric}}      - The metric that triggered
{{context.value}}       - The current metric value
{{context.threshold}}   - The configured threshold
{{context.timestamp}}   - When the alert fired
{{alert.url}}           - Link to the alert in Kibana
```

---

## ML-Based Anomaly Detection

### What It Does
Uses Elastic Machine Learning to learn the **normal behavior** of a metric over time (including time-of-day and day-of-week patterns) and alerts when the metric deviates significantly from that baseline.

### Advantages over Threshold Alerts
- No need to know the "right" threshold in advance
- Automatically adapts to seasonal patterns (e.g., higher traffic on weekdays)
- Detects subtle degradations that wouldn't cross a fixed threshold
- Produces an **anomaly score** (0–100) indicating severity

### Setting Up Anomaly Detection for Metrics

**Observability → Infrastructure → Anomaly detection** (or via ML Jobs)

#### Using the Infrastructure Anomaly Detection Wizard
1. **Observability → Infrastructure → Hosts**
2. Click **Anomaly detection** → **Enable**
3. Select the metric to monitor (e.g., `system.cpu.total.pct`)
4. Select hosts to include (or all hosts)
5. Kibana creates an ML job automatically

#### Via ML Job Management
**Machine Learning → Anomaly Detection → Create job → Use wizard**

### Key ML Job Concepts

| Concept | Description |
|---|---|
| **Detector** | The metric and function being analyzed (e.g., `mean(system.cpu.total.pct)`) |
| **Influencer** | Field whose values help explain anomalies (e.g., `host.name`) |
| **Bucket span** | Time interval for analysis (e.g., `15m`, `1h`) — shorter = more granular but slower |
| **Datafeed** | Continuously queries Elasticsearch and feeds data to the ML job |
| **Model** | The learned baseline; continuously updated as new data arrives |

### Anomaly Score
- **0–25**: Low — minor deviation, informational
- **25–50**: Warning — moderate deviation, worth investigating
- **50–75**: Minor — significant deviation
- **75–100**: Major/Critical — severe deviation, likely an incident

### Viewing Anomalies
- **ML → Anomaly Detection → Anomaly Explorer** — timeline view of anomalies with swimlanes per influencer
- **ML → Anomaly Detection → Single Metric Viewer** — detailed view of one metric with anomaly markers
- **Infrastructure UI** — anomaly markers appear on metric charts when an ML job is active

---

## Anomaly Detection Alerts

Create an alert that fires when the ML anomaly score exceeds a threshold:

**Observability → Alerts → Create rule → Anomaly detection**

| Field | Description |
|---|---|
| **ML job** | The anomaly detection job to monitor |
| **Anomaly score threshold** | Minimum score to trigger (e.g., `75` for major anomalies) |
| **Lookback interval** | How far back to check for anomalies |

### Example: Alert on Critical CPU Anomalies
```
ML job:                 hosts-cpu-anomaly-detection
Anomaly score:          >= 75
Notify every:           1 hour
Action:                 Slack connector → #ops-alerts channel
```

---

## Inventory Alerts (Infrastructure UI)

Create alerts directly from the Infrastructure UI host inventory:

1. **Observability → Infrastructure → Hosts**
2. Select a host → click **...** menu → **Create alert**
3. Pre-populated with the host's name as a filter

This is a shortcut to the metric threshold alert creation wizard.

---

## Alert Rule Management

### Viewing Active Alerts
**Observability → Alerts** — shows all active and recent alert instances with:
- Rule name
- Group (e.g., which host triggered)
- Status (Active / Recovered)
- Duration active
- Last updated

### Muting and Snoozing
- **Mute** — permanently suppresses notifications for a rule
- **Snooze** — temporarily suppresses for a specified duration (e.g., during maintenance)

**Observability → Alerts → Rules → select rule → Snooze**

### Maintenance Windows
Suppress all alerts during a scheduled maintenance period:

**Stack Management → Maintenance Windows → Create**

| Field | Description |
|---|---|
| Name | Descriptive name (e.g., "Weekly patching window") |
| Schedule | Start time, duration, recurrence (once, daily, weekly, monthly) |
| Scope | All rules or specific rule categories |

---

## Practical Exam Tasks

### Create a CPU threshold alert for all production hosts
1. Observability → Alerts → Create rule → Metric threshold
2. Metric: `system.cpu.total.pct`, avg > 0.9 for 5 minutes
3. Group by: `host.name`, Filter: `host.name: prod-*`
4. Action: email connector on Active

### Enable anomaly detection on hosts
1. Observability → Infrastructure → Hosts → Anomaly detection → Enable
2. Select metric: `system.cpu.total.pct`
3. Review anomalies in Anomaly Explorer after the model initializes

### Snooze an alert during a maintenance window
1. Observability → Alerts → Rules
2. Find the rule → click **Snooze** → set duration

### Test a connector
**Stack Management → Connectors → select connector → Test**
Sends a test notification to verify the connector is working.

---

## Exam Tips

- **Metric threshold** = fixed value, rule-based; **Anomaly detection** = ML baseline, dynamic
- Know all alert statuses: **Active**, **Recovered**, **No data** — actions can be set per status
- `Group by` in metric threshold alerts creates **one alert instance per group value** (e.g., per host)
- Connectors are created in **Stack Management → Connectors**, not in the alert wizard
- Anomaly score thresholds: 25 = warning, 50 = minor, 75 = major, 100 = critical
- **Snooze** is temporary; **Mute** is permanent — know the difference
- Maintenance windows suppress all matching alerts for their duration — no individual rule changes needed
- The Infrastructure UI anomaly detection wizard is the quickest path to enabling ML on host metrics

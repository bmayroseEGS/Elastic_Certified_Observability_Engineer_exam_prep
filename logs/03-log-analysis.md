# 03 - Log Analysis

## Overview

Log analysis in the Elastic Stack centers on two tools: **Log Explorer** (Observability-focused, purpose-built for logs) and **Discover** (general-purpose search and exploration). Both are built on the same KQL/Lucene query engine and share the same underlying data.

---

## Key Concepts

- Logs land in **data streams** (e.g., `logs-system.syslog-default`)
- The **`@timestamp`** field drives time-based filtering in both tools
- **ECS (Elastic Common Schema)** standardizes field names so queries work across integrations
- Log levels, host names, service names, and message content are the most-queried fields

---

## Log Explorer

Log Explorer is available at **Observability > Logs > Explorer**.

### When to Use
- Investigating live or recent log activity
- Filtering logs by data stream, integration, or namespace
- Correlating logs with traces and metrics in the same view

### Data Stream Selector
- Top-left dropdown lets you scope to a specific integration (e.g., `nginx.access`, `system.syslog`)
- Namespaces further scope to environments (e.g., `default`, `production`)

### Key Features
| Feature | Description |
|---|---|
| Live streaming | Auto-refreshes to show new log events in near-real-time |
| Log rate histogram | Visual bar chart of log volume over time |
| Field statistics | Click a field to see top values and distribution |
| Surrounding documents | View log context around a specific event |
| Open in Discover | Preserves current query when switching tools |

---

## Discover

Discover is available at **Analytics > Discover**.

### When to Use
- Ad hoc exploration across any index or data view
- Building saved searches to reuse in dashboards
- Exporting query results

### Data View
- A **data view** (formerly "index pattern") maps to one or more indices or data streams
- Select a data view from the top-left dropdown
- The `logs-*` data view covers all log data streams

---

## Querying Logs

### KQL (Kibana Query Language) — Preferred
```
# Match a field value
log.level: "error"

# Wildcard
message: *timeout*

# Boolean operators
log.level: "error" AND host.name: "web-01"

# Range
http.response.status_code >= 500

# Existence check
_exists_: error.message

# Negate
NOT log.level: "debug"
```

### Lucene — Alternative
```
# Field:value
log.level:error

# Wildcard
message:*timeout*

# Range
http.response.status_code:[500 TO *]

# Phrase match (exact multi-word)
message:"connection refused"
```

### Tips
- KQL is case-insensitive for field values by default
- Use quotes around multi-word values: `host.name: "my web server"`
- KQL does **not** support regex — use Lucene for regex: `message:/err.*/`

---

## Filtering

### Time Filter
- Set the time range via the date picker (top-right in both tools)
- Supports absolute ranges, relative ranges (`Last 15 minutes`), and "now"
- Use **Refresh** or **Auto-refresh** intervals for live monitoring

### Field Filters (Filter Pills)
- Click a field value in the document table → **+** to include, **-** to exclude
- Filters appear as pills above the search bar
- Right-click a pill to temporarily disable, negate, or remove it
- Filters persist when switching between Log Explorer and Discover

### Pinned Filters
- Pin a filter so it applies across all data views in the session
- Useful when you always want to scope to a specific environment: `host.environment: "prod"`

---

## Common ECS Fields for Log Analysis

| Field | Description | Example |
|---|---|---|
| `@timestamp` | Event time | `2024-01-15T10:23:00Z` |
| `log.level` | Severity | `error`, `warn`, `info`, `debug` |
| `message` | Raw log message | `"Failed to connect to DB"` |
| `host.name` | Source host | `web-01.example.com` |
| `host.ip` | Source IP | `10.0.1.5` |
| `service.name` | Application name | `payment-service` |
| `error.message` | Error detail | `"null pointer exception"` |
| `http.request.method` | HTTP method | `GET`, `POST` |
| `http.response.status_code` | HTTP status | `200`, `404`, `500` |
| `url.path` | Request path | `/api/v1/orders` |
| `event.dataset` | Integration dataset | `nginx.access` |
| `data_stream.namespace` | Deployment namespace | `production` |

---

## Correlating Log Events

### By Host
Filter to a single host and expand the time range to trace an incident:
```
host.name: "web-01" AND log.level: "error"
```

### By Trace ID (APM Correlation)
When APM is enabled, logs carry `trace.id` and `transaction.id`:
```
trace.id: "abc123def456"
```
This links a log line directly to an APM trace in the same request.

### By Time Window
1. Identify the first error timestamp
2. Expand the query to ±5 minutes to find related events
3. Use the histogram to spot log rate spikes correlating to the incident

### Surrounding Documents
In Log Explorer, click **View surrounding documents** on any log entry to see events immediately before and after — useful for root cause analysis without knowing exact timestamps.

---

## Saved Searches

In Discover, save a query + filter + column set as a **Saved Search**:
1. Run your query
2. Click **Save** (top toolbar) → name the search
3. Saved searches can be embedded in dashboards as panels

---

## Practical Exam Tasks

### Find all errors on a host in the last hour
```
host.name: "web-01" AND log.level: "error"
```
Set time picker to "Last 1 hour".

### Find HTTP 5xx errors
```
http.response.status_code >= 500
```

### Find logs containing a specific string
```
message: *"out of memory"*
```

### Count errors by host
1. Discover → run `log.level: "error"`
2. Click **Visualize** to open in Lens with the current query pre-applied
3. Use a **Top values** breakdown on `host.name`

### Check if a field exists in a document
```
_exists_: error.stack_trace
```

### Exclude noisy debug logs
```
NOT log.level: "debug"
```

---

## Exam Tips

- **Log Explorer** is the go-to for operational log investigation; **Discover** is better for saved searches and dashboard content
- KQL is the default and preferred query language on the exam — know the operators
- ECS field names are consistent across integrations — `log.level`, `host.name`, `service.name` work everywhere
- The **histogram** at the top of both tools is the fastest way to spot anomalies in log volume
- `event.dataset` is the quickest way to filter to a specific integration's logs
- Surrounding documents + trace correlation are high-value exam topics for incident investigation scenarios

# 02 - Ingest Pipelines

## Overview

Ingest pipelines are server-side processing pipelines in Elasticsearch that transform and enrich documents before they are indexed. They are the primary way to parse, structure, and enrich log data.

---

## Key Concepts

- Pipelines run on **ingest nodes** in Elasticsearch
- A pipeline consists of an ordered list of **processors**
- Pipelines can be chained (a processor can call another pipeline)
- Elastic Agent integrations automatically apply default pipelines for known log formats

---

## Creating an Ingest Pipeline

### Via Kibana UI
- Stack Management > Ingest Pipelines > Create pipeline
- Add processors via the visual editor

### Via API
```json
PUT _ingest/pipeline/my-pipeline
{
  "description": "Parse my application logs",
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": ["%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:log.level} %{GREEDYDATA:log.message}"]
      }
    },
    {
      "date": {
        "field": "timestamp",
        "formats": ["ISO8601"]
      }
    },
    {
      "remove": {
        "field": "timestamp"
      }
    }
  ]
}
```

---

## Common Processors

### Grok
Extracts structured fields from unstructured text using patterns.
```json
{
  "grok": {
    "field": "message",
    "patterns": ["%{IP:client.ip} %{WORD:http.method} %{URIPATHPARAM:url.path} %{NUMBER:http.response.status_code:int}"],
    "ignore_failure": true
  }
}
```

**Built-in Grok patterns:**
| Pattern | Matches |
|---|---|
| `%{IP}` | IPv4/IPv6 address |
| `%{NUMBER}` | Numeric value |
| `%{WORD}` | Single word |
| `%{TIMESTAMP_ISO8601}` | ISO 8601 timestamp |
| `%{LOGLEVEL}` | Log level (INFO, WARN, ERROR, etc.) |
| `%{GREEDYDATA}` | Everything remaining |
| `%{URIPATHPARAM}` | URI path with parameters |

### Dissect
Faster alternative to Grok for logs with consistent delimiters.
```json
{
  "dissect": {
    "field": "message",
    "pattern": "%{timestamp} %{log.level} %{log.message}"
  }
}
```

### Date
Parses a date string and sets `@timestamp`.
```json
{
  "date": {
    "field": "timestamp",
    "formats": ["dd/MMM/yyyy:HH:mm:ss Z", "ISO8601"],
    "timezone": "UTC"
  }
}
```

### Set
Sets a field to a fixed or dynamic value.
```json
{
  "set": {
    "field": "event.kind",
    "value": "event"
  }
}
```

### Rename
Renames a field.
```json
{
  "rename": {
    "field": "old_field",
    "target_field": "new_field"
  }
}
```

### Remove
Removes one or more fields.
```json
{
  "remove": {
    "field": ["raw_message", "temp_field"]
  }
}
```

### Append
Appends a value to an array field.
```json
{
  "append": {
    "field": "tags",
    "value": "production"
  }
}
```

### Convert
Converts a field value to a different type.
```json
{
  "convert": {
    "field": "http.response.status_code",
    "type": "integer"
  }
}
```

### User Agent
Parses a User-Agent string into structured fields.
```json
{
  "user_agent": {
    "field": "user_agent.original"
  }
}
```

### GeoIP
Enriches an IP address with geolocation data.
```json
{
  "geoip": {
    "field": "client.ip",
    "target_field": "client.geo"
  }
}
```

### Pipeline
Calls another pipeline.
```json
{
  "pipeline": {
    "name": "another-pipeline"
  }
}
```

---

## Error Handling

### ignore_failure
Skips a processor if it fails and continues with the next one.
```json
{
  "grok": {
    "field": "message",
    "patterns": ["%{GREEDYDATA:parsed}"],
    "ignore_failure": true
  }
}
```

### on_failure
Defines processors to run when a processor fails.
```json
{
  "grok": {
    "field": "message",
    "patterns": ["%{IP:client.ip}"],
    "on_failure": [
      {
        "set": {
          "field": "parse_error",
          "value": "failed to parse IP"
        }
      }
    ]
  }
}
```

---

## Testing a Pipeline

### Via Kibana UI
- Stack Management > Ingest Pipelines > Select pipeline > Test pipeline
- Paste a sample document and run

### Via API
```json
POST _ingest/pipeline/my-pipeline/_simulate
{
  "docs": [
    {
      "_source": {
        "message": "2024-01-15T10:30:00Z INFO User logged in successfully"
      }
    }
  ]
}
```

---

## Applying a Pipeline

### To an index or data stream
```json
PUT logs-myapp-default/_settings
{
  "index.default_pipeline": "my-pipeline"
}
```

### In a bulk request
```
POST _bulk?pipeline=my-pipeline
```

### In Elastic Agent integration settings
- Set the pipeline name in the integration's advanced settings in Fleet

---

## Exam Tips

- Know the difference between Grok (regex-based, flexible) and Dissect (delimiter-based, faster)
- Know how to test pipelines using the Simulate API
- Understand `ignore_failure` vs `on_failure`
- Know how to apply a pipeline to a data stream
- GeoIP and User Agent processors are common enrichment steps
- The `date` processor is required to set `@timestamp` from a parsed field

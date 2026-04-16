# 06 - Grok and Dissect Processors

## Overview

Grok and Dissect are the two primary processors for extracting structured fields from unstructured log messages. Both are used in ingest pipelines. Choosing the right one depends on the log format — Dissect is faster for consistent formats, Grok is more flexible for irregular ones.

---

## Grok

### What It Does
Grok matches a log message against a regex-based pattern and extracts named fields from it.

### Syntax
```
%{PATTERN_NAME:field_name:type}
  ^             ^           ^
  Pattern       Output      Optional
  name          field       data type
                name        (int, float, boolean)
```

### Basic Example
**Raw message:**
```
2024-01-15T10:23:01.000Z ERROR [payment-service] Failed to charge card for user 42
```

**Grok pattern:**
```
%{TIMESTAMP_ISO8601:@timestamp} %{LOGLEVEL:log.level} \[%{DATA:service.name}\] %{GREEDYDATA:message}
```

**Result:**
```json
{
  "@timestamp": "2024-01-15T10:23:01.000Z",
  "log.level": "ERROR",
  "service.name": "payment-service",
  "message": "Failed to charge card for user 42"
}
```

---

## Built-in Grok Patterns

### Commonly Used
| Pattern | Matches | Example |
|---|---|---|
| `%{TIMESTAMP_ISO8601}` | ISO 8601 timestamp | `2024-01-15T10:23:01.000Z` |
| `%{LOGLEVEL}` | Log level keywords | `INFO`, `ERROR`, `WARN`, `DEBUG` |
| `%{IP}` | IPv4 or IPv6 address | `192.168.1.1` |
| `%{NUMBER}` | Integer or float | `42`, `3.14` |
| `%{INT}` | Integer only | `42` |
| `%{WORD}` | Single word (no spaces) | `myvalue` |
| `%{DATA}` | Any characters (non-greedy) | `any text` |
| `%{GREEDYDATA}` | Everything remaining (greedy) | `rest of line` |
| `%{NOTSPACE}` | Non-whitespace characters | `value_no_spaces` |
| `%{SPACE}` | One or more whitespace chars | ` ` |
| `%{URI}` | Full URI | `https://example.com/path` |
| `%{URIPATH}` | URI path only | `/api/v1/users` |
| `%{URIPATHPARAM}` | Path with query string | `/api/v1/users?id=42` |
| `%{HTTPDATE}` | Apache/Nginx date format | `15/Jan/2024:10:23:01 +0000` |
| `%{COMMONAPACHELOG}` | Full Apache access log line | _(full line)_ |
| `%{COMBINEDAPACHELOG}` | Apache combined log format | _(full line with user agent)_ |

---

## Grok in an Ingest Pipeline

```json
PUT _ingest/pipeline/myapp-logs-pipeline
{
  "processors": [
    {
      "grok": {
        "field": "message",
        "patterns": [
          "%{TIMESTAMP_ISO8601:event.created} %{LOGLEVEL:log.level} %{IP:client.ip} %{NUMBER:http.response.status_code:int} %{GREEDYDATA:url.path}"
        ],
        "ignore_failure": true
      }
    }
  ]
}
```

---

## Multiple Patterns (Fallback)

Grok tries each pattern in order and uses the first one that matches. This handles logs with different formats in the same file:

```json
{
  "grok": {
    "field": "message",
    "patterns": [
      "%{TIMESTAMP_ISO8601:@timestamp} %{LOGLEVEL:log.level} %{GREEDYDATA:log.message}",
      "%{HTTPDATE:@timestamp} %{WORD:http.method} %{URIPATHPARAM:url.path} %{NUMBER:http.response.status_code:int}",
      "%{GREEDYDATA:log.message}"
    ]
  }
}
```

---

## Custom Grok Patterns

Define your own named patterns for reuse:

```json
{
  "grok": {
    "field": "message",
    "patterns": ["%{TXID:transaction.id} %{GREEDYDATA:message}"],
    "pattern_definitions": {
      "TXID": "[A-F0-9]{8}-[A-F0-9]{4}-[A-F0-9]{4}-[A-F0-9]{4}-[A-F0-9]{12}"
    }
  }
}
```

---

## Grok Debugger

Test patterns interactively in Kibana:
- **Dev Tools → Grok Debugger**
- Paste your raw log line and pattern
- Instantly see extracted fields and any errors

---

## Dissect

### What It Does
Dissect splits a log message by fixed delimiters into named fields. It does not use regex — it processes strings sequentially from left to right, making it significantly faster than Grok.

### Syntax
```
%{field_name}           <- capture until next delimiter
%{}                     <- skip (throw away)
%{+field_name}          <- append to existing field
%{field_name->}         <- trim trailing whitespace
%{?field_name}          <- skip (named skip)
```

### Basic Example
**Raw message:**
```
2024-01-15T10:23:01Z ERROR payment-service Failed to charge card
```

**Dissect pattern:**
```
%{@timestamp} %{log.level} %{service.name} %{message}
```

**Result:**
```json
{
  "@timestamp": "2024-01-15T10:23:01Z",
  "log.level": "ERROR",
  "service.name": "payment-service",
  "message": "Failed to charge card"
}
```

---

## Dissect Special Operators

### Append (`+`)
Combine multiple tokens into one field:
```
%{+log.message} %{+log.message}
```
Input: `Hello World` → `log.message: "Hello World"`

### Append with order (`+field/N`)
Control the order when appending:
```
%{+message/2} %{+message/1}
```
Input: `World Hello` → `message: "Hello World"`

### Skip (`?` or empty `%{}`)
Discard a token:
```
%{@timestamp} %{?ignore_this} %{log.level} %{message}
```

### Right padding (`->`)
Absorb extra whitespace between delimiters:
```
%{log.level->} %{message}
```
Handles `ERROR   some message` where spacing is inconsistent.

---

## Dissect in an Ingest Pipeline

```json
PUT _ingest/pipeline/myapp-dissect-pipeline
{
  "processors": [
    {
      "dissect": {
        "field": "message",
        "pattern": "%{event.created} [%{log.level}] %{service.name}: %{log.message}",
        "ignore_failure": true
      }
    }
  ]
}
```

---

## Grok vs Dissect — When to Use Which

| | Grok | Dissect |
|---|---|---|
| **Speed** | Slower (regex engine) | Faster (string splitting) |
| **Flexibility** | High — handles variable spacing, optional fields | Low — delimiters must be consistent |
| **Best for** | Irregular formats, variable-length fields | Structured formats with fixed delimiters |
| **Fallback patterns** | Yes — multiple patterns tried in order | No |
| **Custom patterns** | Yes — `pattern_definitions` | No |
| **Handles optional fields** | Yes (with `?` in regex) | No |

### Rule of Thumb
- Start with **Dissect** if the format is consistent and delimiters are reliable
- Fall back to **Grok** if Dissect can't handle the variation in the log format
- Use **both** in the same pipeline: Dissect first, then Grok as a fallback

---

## Using Both in One Pipeline

```json
{
  "processors": [
    {
      "dissect": {
        "field": "message",
        "pattern": "%{@timestamp} %{log.level} %{log.message}",
        "ignore_failure": true
      }
    },
    {
      "grok": {
        "field": "message",
        "patterns": ["%{TIMESTAMP_ISO8601:@timestamp} %{LOGLEVEL:log.level} %{GREEDYDATA:log.message}"],
        "if": "ctx.log?.level == null",
        "ignore_failure": true
      }
    }
  ]
}
```

The `if` condition ensures Grok only runs if Dissect didn't already extract `log.level`.

---

## Conditional Processor Execution

Both Grok and Dissect support `if` conditions using Painless script:

```json
{
  "grok": {
    "field": "message",
    "patterns": ["%{COMBINEDAPACHELOG}"],
    "if": "ctx.event?.dataset == 'nginx.access'"
  }
}
```

---

## Exam Tips

- **Grok** = regex-based, flexible, slower — use for irregular logs
- **Dissect** = delimiter-based, fast, strict — use for consistent formats
- Know the common built-in patterns: `%{TIMESTAMP_ISO8601}`, `%{LOGLEVEL}`, `%{IP}`, `%{GREEDYDATA}`, `%{DATA}`
- `ignore_failure: true` prevents the pipeline from failing when a pattern doesn't match
- Multiple Grok patterns are tried in order — always put the most specific pattern first
- Use the **Grok Debugger** in Kibana Dev Tools to test patterns before adding to a pipeline
- Know the Dissect special operators: `+` (append), `?` (skip), `->` (right-pad)
- A pipeline can use Dissect first and Grok as a fallback with the `if` condition

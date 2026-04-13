# 05 - Multiline Log Handling

## Overview

Many log formats span multiple lines — Java stack traces, Python tracebacks, and multiline JSON are common examples. By default, Elastic Agent treats each line as a separate log event. **Multiline configuration** tells the agent how to stitch related lines together into a single document.

---

## Key Concepts

- Multiline config is applied at **collection time** (in the Elastic Agent input config or Fleet integration settings)
- The goal is to group lines that belong to the same logical event into one document before shipping
- Multiline is configured per log stream/input

---

## Multiline Configuration Parameters

| Parameter | Description |
|---|---|
| `pattern` | Regex that identifies the start (or end) of a new event |
| `negate` | If `true`, lines that do NOT match the pattern are continuation lines |
| `match` | `after` — append non-matching lines to the previous match; `before` — prepend to the next match |
| `max_lines` | Max lines to merge into one event (default: 500) |
| `timeout` | Flush incomplete multiline event after this duration (default: 5s) |

---

## Common Multiline Patterns

### Java Stack Trace
Java exceptions start with a log-level prefix; stack trace lines start with whitespace or `at `:

```yaml
multiline:
  type: pattern
  pattern: '^[[:space:]]+(at|\.{3})[[:space:]]|^Caused by:'
  negate: false
  match: after
```

Or more simply — a new event starts with a timestamp:
```yaml
multiline:
  type: pattern
  pattern: '^\d{4}-\d{2}-\d{2}'
  negate: true
  match: after
```

**How it works:**
- Lines that do NOT start with a date (`negate: true`) are appended to the previous event (`match: after`)
- The next line that starts with a date begins a new event

**Example input:**
```
2024-01-15 10:23:01 ERROR NullPointerException
    at com.example.Service.process(Service.java:42)
    at com.example.Main.main(Main.java:10)
2024-01-15 10:23:02 INFO Request completed
```

**Result:** Two documents — one for the exception (3 lines merged), one for the INFO line.

---

### Python Traceback
Python tracebacks start with `Traceback (most recent call last):` — new events start with a timestamp:

```yaml
multiline:
  type: pattern
  pattern: '^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}'
  negate: true
  match: after
```

---

### Go Panic
Go panics start with `goroutine` or `panic:`:

```yaml
multiline:
  type: pattern
  pattern: '^(goroutine|panic:)'
  negate: false
  match: after
```

---

### Multiline JSON
When each log event is a JSON object that spans multiple lines:

```yaml
multiline:
  type: pattern
  pattern: '^\{'
  negate: false
  match: after
```

> **Better approach:** Most modern apps write single-line JSON. If you control the app, configure it to write one JSON object per line and skip multiline config entirely.

---

## Configuring Multiline in Fleet (Custom Logs Integration)

In the Fleet UI for a Custom Logs integration:

1. Open the integration settings
2. Expand **Advanced options**
3. Under **Custom configurations**, add:

```yaml
parsers:
  - multiline:
      type: pattern
      pattern: '^\d{4}-\d{2}-\d{2}'
      negate: true
      match: after
```

---

## Configuring Multiline in `elastic-agent.yml` (Standalone)

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
        data_stream:
          dataset: myapp.application
          namespace: default
```

---

## Multiline Types

### `pattern` (most common)
Groups lines based on whether they match a regex pattern.

### `count`
Groups a fixed number of lines into one event regardless of content.
```yaml
multiline:
  type: count
  count_lines: 4
```

### `while_pattern`
Continues adding lines as long as they match the pattern.
```yaml
multiline:
  type: while_pattern
  pattern: '^[[:space:]]'
```

---

## match: after vs match: before

### `match: after` (most common)
Lines following the pattern match are appended to the matched line.

```
[MATCH]     <- new event starts here
[no match]  <- appended to above
[no match]  <- appended to above
[MATCH]     <- new event starts here
```

Use when: new events start with a recognizable pattern (timestamp, log level).

### `match: before`
Lines before the next pattern match are prepended to the matched line.

```
[no match]  <- will be prepended to next match
[no match]  <- will be prepended to next match
[MATCH]     <- these lines become one event
[no match]
[MATCH]     <- these lines become one event
```

Use when: events end with a recognizable pattern.

---

## Limits and Timeouts

```yaml
multiline:
  type: pattern
  pattern: '^\d{4}-\d{2}-\d{2}'
  negate: true
  match: after
  max_lines: 1000    # prevent runaway merging
  timeout: 10s       # flush if no new line arrives within 10s
```

- `max_lines` protects against extremely long stack traces consuming excessive memory
- `timeout` ensures incomplete events are flushed and not lost if the app stops writing mid-event

---

## Testing Multiline Config

1. Use `elastic-agent` in standalone mode with a test input
2. Check the resulting document in Discover — the `message` field should contain all merged lines
3. Verify the line count with a field stats check in Log Explorer

---

## Exam Tips

- `pattern` + `negate: true` + `match: after` is the most common pattern — new event starts with a timestamp
- Know the difference between `match: after` and `match: before`
- Multiline is applied at **collection time**, not in an ingest pipeline
- `max_lines` and `timeout` are safety valves — know what they do
- For JSON logs, prefer single-line JSON over multiline config whenever possible
- In Fleet, multiline goes in the **Custom configurations** YAML block of the Custom Logs integration

# Structured Logging: JSON Formats and Structlog Integration

Traditional log files are written as flat, unstructured text lines. While easy for a human developer to read during development, flat text is difficult for automated log indexers (like Elasticsearch, Logstash, Loki, or Datadog) to parse. If you want to search specifically for requests where `user_id = 42` and `latency > 500ms`, a flat text indexer must run expensive regular expressions across millions of rows.

**Structured logging** solves this by outputting log records in a structured machine-readable format—typically **JSON**. Every log line becomes a self-contained JSON database document. Search engines can instantly parse, index, and query these attributes.

In this guide, we will implement a custom JSON formatter using Python's standard library, introduce the **`structlog`** framework, and build a production-ready structured logger for API services.

---

## Unstructured vs. Structured Logs

### Unstructured Text Log
```text
2026-06-27 18:00:00 [ERROR] app.api: Request to /checkout failed for user 1404 after 3 retries (database timed out).
```
*Note: To extract the endpoint, username, or retry count, you must design custom regular expressions.*

### Structured JSON Log
```json
{
  "timestamp": "2026-06-27T18:00:00Z",
  "level": "ERROR",
  "logger": "app.api",
  "message": "Request failed",
  "endpoint": "/checkout",
  "user_id": 1404,
  "retries": 3,
  "error": "database timed out"
}
```
*Note: Log indexers can parse this directly into separate, queryable database fields.*

---

## 1. Custom JSON Formatter using the Standard Library

You don't need third-party packages to write JSON logs. You can extend the built-in `logging.Formatter` class to convert `LogRecord` objects to JSON strings.

```python
import logging
import json
from datetime import datetime

class JsonFormatter(logging.Formatter):
    """
    Custom Formatter that outputs logs in structured JSON format.
    """
    def format(self, record):
        # 1. Compile baseline log metrics
        log_document = {
            "timestamp": datetime.utcfromtimestamp(record.created).isoformat() + "Z",
            "level": record.levelname,
            "logger": record.name,
            "message": record.getMessage(),
            "file": record.filename,
            "line": record.lineno,
            "function": record.funcName
        }

        # 2. Extract extra context parameters passed in the 'extra' keyword
        if hasattr(record, "extra_fields"):
            log_document.update(record.extra_fields)
            
        # 3. Serialize to a single-line JSON string
        return json.dumps(log_document)

# --- Configuration & Usage ---
logger = logging.getLogger("api_service")
handler = logging.StreamHandler()
handler.setFormatter(JsonFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Log event with extra fields
logger.info(
    "User authentication attempt", 
    extra={"extra_fields": {"user_id": 1404, "ip_address": "10.0.0.12", "action": "login"}}
)
```

---

## 2. Structured Logging with `structlog`

For large-scale applications, the **`structlog`** library is the industry standard for structured logging in Python. It provides a processing pipeline that allows you to format, validate, and structure log payloads dynamically.

Install the library:
```bash
pip install structlog
```

Here is a complete structured logging pipeline implementation:

```python
import structlog
import logging
import sys

# Configure the structlog processing pipeline
structlog.configure(
    processors=[
        # 1. Add standard log level key (info, error, etc.)
        structlog.processors.add_log_level,
        # 2. Add high-precision ISO timestamp
        structlog.processors.TimeStamper(fmt="iso", utc=True),
        # 3. Format tracebacks if an exception is logged
        structlog.processors.format_exc_info,
        # 4. Render the final output as a JSON string
        structlog.processors.JSONRenderer()
    ],
    # Direct outputs to standard out
    logger_factory=structlog.PrintLoggerFactory(sys.stdout),
    cache_logger_on_first_use=True
)

# Instantiate structlog logger
logger = structlog.get_logger()

# Log events with structured context keys
logger.info(
    "http_request_processed",
    method="GET",
    path="/items/45",
    status_code=200,
    latency_ms=45.2
)

try:
    result = 10 / 0
except ZeroDivisionError:
    # Captures exception info automatically
    logger.exception("calculation_failed", task_id="batch_402")
```

### Output:
```json
{"event": "http_request_processed", "method": "GET", "path": "/items/45", "status_code": 200, "latency_ms": 45.2, "timestamp": "2026-06-27T12:30:00.123456Z", "level": "info"}
{"event": "calculation_failed", "task_id": "batch_402", "timestamp": "2026-06-27T12:30:01.456789Z", "level": "error", "exception": "Traceback..."}
```

---

## Summary

Structured logging formats your log lines as queryable database records. By implementing custom JSON formatters or configuring the `structlog` processing pipeline, you can generate logs that are easily indexed by log aggregators. In the next section, we will cover **Logging Best Practices**.

# Log Formatters: Customizing Metadata Layouts

A raw message like "database timeout" is hard to debug without context. When did it occur? Which server process raised the error? What thread was executing the query? **Formatters** answer these questions by mapping metadata fields directly into your log strings.

Python's `logging` module creates a dynamic `LogRecord` object for every log event. This object contains attributes representing the runtime context. Formatters translate this metadata into a structured text line.

In this guide, we will examine the standard `LogRecord` attributes, customize date and time formats (including millisecond accuracy), and write a custom Formatter subclass that dynamically injects request IDs into console log lines.

---

## Standard LogRecord Attributes

When a log message is generated, you can use the following variables in your formatter string:

| Attribute Name | String Format Specifier | Description |
| :--- | :--- | :--- |
| **Time** | `%(asctime)s` | Date and time when the log was created. |
| **Level** | `%(levelname)s` | String representation of log severity (e.g. `INFO`). |
| **Message** | `%(message)s` | The log message passed by the developer. |
| **Filename** | `%(filename)s` | Source code file name containing the logging call. |
| **Line Number** | `%(lineno)d` | Line number of the source file where log was emitted. |
| **Function** | `%(funcName)s` | Function name containing the logging call. |
| **Process ID** | `%(process)d` | Operating system process ID (PID) executing the script. |
| **Thread Name** | `%(threadName)s` | Execution thread name (useful for concurrent apps). |

---

## 1. Customizing Date and Time Formatting

By default, `asctime` formats time using the format: `YYYY-MM-DD HH:MM:SS,mmm` (where `mmm` represents milliseconds). 

You can customize this layout by passing a standard `datefmt` string to the `Formatter` constructor, utilizing standard **`time.strftime`** directives (e.g. `%Y`, `%m`, `%d`, `%H`, `%M`, `%S`).

```python
import logging

logger = logging.getLogger("time_formatter")
handler = logging.StreamHandler()

# Define format string: [ISO Timestamp] [Level] - Message
format_pattern = "%(asctime)s - %(levelname)s - %(message)s"

# Define date structure (ISO-8601 representation)
# Note: Custom datefmt drops milliseconds by default in standard Python logging
date_pattern = "%Y-%m-%dT%H:%M:%S"

formatter = logging.Formatter(fmt=format_pattern, datefmt=date_pattern)
handler.setFormatter(formatter)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

logger.info("Time formatting configured successfully.")
```

### Output:
```text
2026-06-27T17:55:00 - INFO - Time formatting configured successfully.
```

---

## 2. Dynamic Metadata Injection: Custom Formatter Subclass

In modern microservices, tracing a transaction across multiple systems requires a **Correlation ID** (or Request ID). You want this Request ID automatically printed in every log statement during the request context.

We can implement this by subclassing `logging.Formatter` and dynamically checking a thread-local context object or the `extra` dictionary.

```python
import logging
import threading

# Thread-local storage to hold the active request ID
thread_context = threading.local()

class RequestIdFormatter(logging.Formatter):
    """
    Custom Formatter that injects a unique Request ID into every log line.
    """
    def format(self, record):
        # 1. Fetch request ID from thread-local context (default to 'SYSTEM' if none)
        request_id = getattr(thread_context, "request_id", "SYSTEM")
        
        # 2. Inject the request ID directly into the LogRecord object
        record.request_id = request_id
        
        # 3. Call parent format method to construct the string
        return super().format(record)

# --- Configuration and Test ---
def initialize_logger():
    logger = logging.getLogger("api_request_tracker")
    handler = logging.StreamHandler()

    # Format string includes our custom %(request_id)s parameter
    format_pattern = "[%(asctime)s] [%(request_id)s] [%(levelname)s] (%(filename)s:%(lineno)d) - %(message)s"
    
    formatter = RequestIdFormatter(fmt=format_pattern, datefmt="%H:%M:%S")
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    logger.setLevel(logging.INFO)
    return logger

logger = initialize_logger()

# Scenario A: Log event outside request context (default to SYSTEM)
logger.info("Server started listening on port 8080.")

# Scenario B: Log events inside request thread
def process_client_request(client_ip, req_id):
    # Set thread context value
    thread_context.request_id = req_id
    
    logger.info(f"Incoming connection from {client_ip}")
    logger.info("Querying client account credentials...")
    
    # Reset context after request completes
    thread_context.request_id = "SYSTEM"

process_client_request("192.168.1.100", "req_98b3f11a")
```

### Output:
```text
[17:56:00] [SYSTEM] [INFO] (app.py:38) - Server started listening on port 8080.
[17:56:01] [req_98b3f11a] [INFO] (app.py:43) - Incoming connection from 192.168.1.100
[17:56:01] [req_98b3f11a] [INFO] (app.py:44) - Querying client account credentials...
```

---

## Summary

Formatters add structural context to raw log messages. By utilizing date formatting patterns and subclassing `logging.Formatter` to inject dynamic values like Request IDs, you can create logs that are easy to parse and trace. In the next section, we will cover **Logging to Files**.

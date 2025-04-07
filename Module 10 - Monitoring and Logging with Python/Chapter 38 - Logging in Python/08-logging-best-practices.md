# Logging Best Practices: Writing Secure and Performant Logs

Writing logs is simple, but writing *good* logs is challenging. In production, logs must strike a balance: they must contain enough details to debug failures, without leaking customer passwords, slowing down the CPU, or consuming massive amounts of disk space.

In this guide, we will implement four essential logging best practices: properly capturing exception tracebacks, protecting sensitive data by writing a custom **Redaction Filter**, optimizing CPU performance using **Lazy String Evaluation**, and avoiding root logger pollution.

---

## 1. Capturing Exception Tracebacks Correctly

When writing an `except` block, developers often log the raw string conversion of the exception (e.g. `logger.error(str(e))`). This discards the stack trace, leaving you with no context on which file or line number triggered the crash.

Always use **`logger.exception()`** inside `except` blocks. This automatically appends the complete stack trace to the log message.

```python
import logging

logger = logging.getLogger("calculator_app")

try:
    result = 10 / 0
except ZeroDivisionError as e:
    # logger.exception automatically logs at the ERROR level and appends the traceback
    logger.exception("Failed to execute division operation.")
```

### Output:
```text
ERROR:calculator_app:Failed to execute division operation.
Traceback (most recent call last):
  File "app.py", line 6, in <module>
    result = 10 / 0
ZeroDivisionError: division by zero
```

---

## 2. Preventing Credential Leaks with a Redaction Filter

To ensure compliance with security standards (like PCI-DSS or GDPR), your logs must never contain sensitive parameters like passwords, API keys, or credit card numbers. 

Instead of relying on developers to manually hide passwords, you can write a custom **`logging.Filter`** that intercepts all logs and redacts sensitive data using regular expressions before it is written to disk.

```python
import logging
import re

class PersonalDataRedactorFilter(logging.Filter):
    """
    Filter that automatically redacts sensitive patterns (like passwords or API tokens)
    from log messages before they are processed by handlers.
    """
    def __init__(self):
        super().__init__()
        # Regex patterns to identify sensitive data
        # Matches formats like: password=XYZ, token: XYZ, secret=XYZ
        self.sensitive_patterns = [
            re.compile(r"(password|token|secret|auth_key)\s*[:=]\s*['\"]?\w+['\"]?", re.IGNORECASE)
        ]

    def filter(self, record):
        # Obtain the raw log message
        message = record.getMessage()
        
        # Scan and replace patterns
        for pattern in self.sensitive_patterns:
            message = pattern.sub(r"\1=******", message)
            
        # Write the scrubbed message back to the log record
        record.msg = message
        # Clear args to prevent logging from trying to interpolate old variables
        record.args = () 
        
        return True # Return True to allow the record to be processed

# --- Configuration & Testing ---
logger = logging.getLogger("secure_app")
handler = logging.StreamHandler()
handler.setFormatter(logging.Formatter("%(levelname)s - %(message)s"))

# Attach the custom filter to the handler
handler.addFilter(PersonalDataRedactorFilter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Test logs containing sensitive data
logger.info("Connecting to server using API_KEY: token=ghp_secretTokenValue")
logger.warning("Failed login attempt for user 'admin' with password=SuperSecretPassword123")
```

### Output:
```text
INFO - Connecting to server using API_KEY: token=******
WARNING - Failed login attempt for user 'admin' with password=******
```

---

## 3. Lazy String Evaluation for Performance

In high-performance systems, formatting strings inside log statements can waste significant CPU cycles. 

If you use f-strings or `.format()` inside log calls:
```python
logger.debug(f"Query returned: {expensive_database_call()}")
```
Python evaluates the `expensive_database_call()` function and formats the string *before* passing it to the logger. If the logger is configured at the `WARNING` level, the debug message is immediately discarded, meaning you wasted CPU cycles generating a string that was never written.

Instead, use **lazy formatting** by passing arguments as comma-separated values:

```python
# The string is only formatted if the logger's level is set to DEBUG
logger.debug("Query returned: %s", expensive_database_call)
```

---

## 4. Never Use the Root Logger Directly

Calling `logging.info()` or `logging.error()` directly uses the global root logger. This causes problems because third-party libraries (like `urllib3` or `boto3`) also log to the root logger, resulting in cluttered log files.

Instead, always create named, module-specific loggers in each of your files:

```python
# Instantiate a dedicated logger named after the module path
logger = logging.getLogger(__name__)
```

---

## Summary

Following logging best practices ensures that your logs are searchable, performant, and secure. Capturing stack traces with `logger.exception()`, implementing redaction filters to mask secrets, and utilizing lazy formatting allows you to write production-grade monitoring systems. In the next chapter, we will shift focus to **Metrics with Prometheus**.

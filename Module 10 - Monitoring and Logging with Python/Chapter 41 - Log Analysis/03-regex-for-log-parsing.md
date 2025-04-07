# Regex for Log Parsing: Regular Expressions and Named Capture Groups

While basic string splits (`.split()`) work for simple log formats, they fail on complex log lines (like Nginx, Apache, or Kubernetes logs) that wrap parameters in quotes or brackets. 

Python's built-in **`re`** (Regular Expression) module provides a powerful pattern-matching engine to parse complex text formats. By using **Named Capture Groups**, you can extract specific fields from raw log lines directly into Python dictionaries.

In this guide, we will cover how to use Python's `re` module, compile regex patterns to optimize performance, write regex patterns for log components, and implement a multi-format log parser.

---

## Why Compile Regular Expressions?

When parsing a log file containing millions of lines, performance is critical. If you use inline regex calls inside a loop:
```python
# AVOID: Re-compiles the regex pattern string for every line
for line in file:
    match = re.match(r"\d{4}-\d{2}-\d{2}", line)
```
Python compiles the regex pattern string into byte code on every iteration, which degrades performance.

Instead, compile the pattern once outside the loop using **`re.compile()`**:
```python
# RECOMMENDED: Compiles the pattern once
pattern = re.compile(r"\d{4}-\d{2}-\d{2}")

for line in file:
    match = pattern.match(line)
```
This optimization can speed up your log parser by **300%**.

---

## 1. Writing Regex Patterns for Log Primitives

To construct a log parser, we combine regular expression components into named capture groups using the syntax: `(?P<group_name>pattern)`.

Here are patterns for common log variables:

### IP Address (IPv4)
```regex
(?P<ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3})
```

### Timestamps (ISO-8601 / RFC 3339)
Matches formats like `2026-06-27T18:10:00Z` or `2026-06-27 18:10:00.123`:
```regex
(?P<timestamp>\d{4}-\d{2}-\d{2}[T ]\d{2}:\d{2}:\d{2}(?:\.\d+)?[Z\d\:\+\-]*)
```

### HTTP Request Method
Matches standard REST methods:
```regex
(?P<method>GET|POST|PUT|DELETE|PATCH|HEAD|OPTIONS)
```

### HTTP Status Code
Matches 3-digit status codes:
```regex
(?P<status>[1-5]\d{2})
```

---

## 2. Implementing a Multi-Format Log Parser

In production systems, a single log file can contain multiple different log formats (e.g. standard output logs mixed with runtime error exception stack traces). 

Here is a Python class that maintains a list of compiled regex patterns. It tests each pattern sequentially and returns the first matching dictionary structure.

```python
import re

class MultiFormatLogParser:
    def __init__(self):
        # Define and compile multiple log schemas
        # Schema A: Standard info logs
        # Format: "2026-06-27 18:10:00 [INFO] User logged in"
        self.pattern_standard = re.compile(
            r"^(?P<timestamp>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\s+"
            r"\[(?P<level>[A-Z]+)\]\s+"
            r"(?P<message>.+)$"
        )
        
        # Schema B: API access logs
        # Format: "IP: 10.0.0.5 | GET /index | Status: 200"
        self.pattern_api = re.compile(
            r"^IP:\s*(?P<ip>\S+)\s*\|\s*"
            r"(?P<method>[A-Z]+)\s*(?P<path>\S+)\s*\|\s*"
            r"Status:\s*(?P<status>\d{3})$"
        )

    def parse_line(self, line):
        line = line.strip()
        
        # 1. Attempt to match Standard Schema
        match = self.pattern_standard.match(line)
        if match:
            data = match.groupdict()
            data["log_schema"] = "standard"
            return data
            
        # 2. Attempt to match API Schema
        match = self.pattern_api.match(line)
        if match:
            data = match.groupdict()
            data["log_schema"] = "api"
            return data
            
        # 3. If no pattern matches, return None (unrecognized log format)
        return None

# --- Testing the Multi-Parser ---
parser = MultiFormatLogParser()

line_a = "2026-06-27 18:10:00 [INFO] Service started."
line_b = "IP: 192.168.1.1 | GET /api/v1/metrics | Status: 200"
line_c = "Obsolete raw text debug output" # Should fail to parse

print(parser.parse_line(line_a))
print(parser.parse_line(line_b))
print(parser.parse_line(line_c))
```

### Output:
```python
{'timestamp': '2026-06-27 18:10:00', 'level': 'INFO', 'message': 'Service started.', 'log_schema': 'standard'}
{'ip': '192.168.1.1', 'method': 'GET', 'path': '/api/v1/metrics', 'status': '200', 'log_schema': 'api'}
None
```

---

## Summary

Regular expressions combined with named capture groups allow you to parse complex log formats into structured Python dictionaries. Compiling patterns with `re.compile()` optimizes parser performance. In the next section, we will use these regex patterns to write a parser for **Nginx log files**.

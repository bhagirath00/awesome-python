# Analyzing Nginx Logs: Compiling Access Stats and Traffic Patterns

Nginx is one of the most common web servers in the world. By default, it logs client HTTP requests using the **Combined Log Format**. Analyzing these logs allows you to track page hits, detect server errors, measure data egress, and identify malicious traffic (like automated vulnerability scanners).

In this guide, we will examine the Nginx Combined log structure, write a compiled regular expression to parse it, and build a Python script to compile web server statistics.

---

## Nginx Combined Log Format Layout

A typical Nginx log line looks like this:
```text
192.168.1.100 - - [27/Jun/2026:18:15:00 +0530] "GET /api/v1/users HTTP/1.1" 200 4522 "-" "Mozilla/5.0"
```

The fields are defined as:
* **`remote_addr`**: The client IP address (`192.168.1.100`).
* **`remote_user`**: The authenticated user (`-` if none).
* **`time_local`**: Local time when request was received (`[27/Jun/2026:18:15:00 +0530]`).
* **`request`**: The HTTP method, URI, and protocol version (`"GET /api/v1/users HTTP/1.1"`).
* **`status`**: The HTTP response status code (`200`).
* **`body_bytes_sent`**: The size of the response sent to client in bytes (`4522`).
* **`http_referer`**: The referral URL (`"-"` if none).
* **`http_user_agent`**: The client web browser or crawler signature (`"Mozilla/5.0"`).

---

## 1. Nginx Combined Regex Matcher

Here is the regular expression to parse the Combined Log Format:

```python
import re

# Regex matching the Nginx Combined layout
NGINX_COMBINED_PATTERN = re.compile(
    r'^(?P<ip>\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}) - (?P<user>\S+) '
    r'\[(?P<timestamp>[^\]]+)\] '
    r'"(?P<method>[A-Z]+) (?P<path>\S+) (?P<protocol>[^"]+)" '
    r'(?P<status>\d{3}) (?P<bytes>\d+) '
    r'"(?P<referer>[^"]*)" "(?P<agent>[^"]*)"'
)
```

---

## 2. Complete Nginx Access Log Analyzer Script

Here is a complete Python script that reads an Nginx access log file line-by-line using a generator, parses the entries, aggregates statistics (top IP addresses, response codes, bandwidth usage), and prints a formatted report:

```python
import os
import re
from collections import Counter

# (Requires the NGINX_COMBINED_PATTERN defined above)

def stream_lines(filepath):
    with open(filepath, "r", encoding="utf-8") as f:
        for line in f:
            yield line

def analyze_nginx_access_log(log_path):
    if not os.path.exists(log_path):
        print(f"Error: Access log file '{log_path}' not found.")
        return

    # Initialize aggregators
    client_ips = Counter()
    status_codes = Counter()
    endpoints = Counter()
    total_bytes_transferred = 0
    total_requests = 0
    failed_lines = 0

    print(f"Analyzing Nginx access log: {log_path}...")

    # Iterate line-by-line
    for line in stream_lines(log_path):
        match = NGINX_COMBINED_PATTERN.match(line.strip())
        if match:
            data = match.groupdict()
            total_requests += 1
            
            # Aggregate metrics
            client_ips[data["ip"]] += 1
            status_codes[data["status"]] += 1
            endpoints[data["path"]] += 1
            total_bytes_transferred += int(data["bytes"])
        else:
            failed_lines += 1

    # Print Compiled Statistics Report
    print("\n==================================================")
    print("📊 NGINX ACCESS LOG REPORT")
    print("==================================================")
    print(f"Total Requests Analyzed:  {total_requests}")
    print(f"Egress Data Transferred:   {total_bytes_transferred / (1024*1024):.2f} MB")
    print(f"Malformed Lines Skipped:   {failed_lines}\n")

    print("Top 5 Active Client IP Addresses:")
    for ip, count in client_ips.most_common(5):
        percent = (count / total_requests) * 100 if total_requests else 0
        print(f" - {ip:<15} : {count:<5} requests ({percent:.1f}%)")

    print("\nHTTP Response Code Distribution:")
    for status, count in sorted(status_codes.items()):
        print(f" - HTTP {status} : {count} occurrences")

    print("\nTop 5 Requested Endpoints:")
    for path, count in endpoints.most_common(5):
        print(f" - {path:<25} : {count} hits")
    print("==================================================")

# Example usage:
# analyze_nginx_access_log("/var/log/nginx/access.log")
```

---

## Summary

Analyzing web server access logs allows you to monitor traffic patterns and detect issues like spikes in error rates or automated vulnerability scans. In the next section, we will learn how to parse **application logs**, which often contain complex multi-line exception stack traces.

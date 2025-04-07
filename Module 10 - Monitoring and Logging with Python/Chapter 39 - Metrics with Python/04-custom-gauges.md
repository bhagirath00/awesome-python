# Custom Gauges: Tracking System State and Resources

A **Gauge** represents a single numerical value that can go up and down. While Counters only increase, Gauges are used to measure current values, such as memory usage, CPU temperature, active user sessions, queue size, or database connection pools.

In this guide, we will cover how to update gauges (set, increment, decrement), track processing time automatically using timers, and implement **Callback Gauges** to collect system metrics dynamically when Prometheus scrapes your endpoint.

---

## 1. Basic Gauge Operations: Set, Inc, and Dec

Gauges support direct assignment as well as relative modifications:

```python
from prometheus_client import Gauge

# 1. Track current queue depth
JOB_QUEUE_SIZE = Gauge("worker_queue_items_count", "Current items waiting in worker queue")

# Set the gauge to an absolute value
JOB_QUEUE_SIZE.set(45)

# 2. Track active worker threads
ACTIVE_WORKERS = Gauge("active_worker_threads", "Number of active threads processing jobs")

def start_worker():
    ACTIVE_WORKERS.inc() # Increment value by 1

def terminate_worker():
    ACTIVE_WORKERS.dec() # Decrement value by 1
```

---

## 2. Using Gauges as Timers

Gauges can track duration, such as how long a backup takes or the latency of a database query. The `prometheus-client` library allows you to use gauges as decorators or context managers to automatically measure and update execution time.

### Using as a Decorator
```python
from prometheus_client import Gauge

BACKUP_TIME = Gauge("backup_execution_duration_seconds", "Time spent running system backup")

@BACKUP_TIME.time()
def run_system_backup():
    # Perform backup calculations
    # The elapsed time is automatically recorded in the gauge once the function returns
    pass
```

### Using as a Context Manager
```python
from prometheus_client import Gauge
import time

API_RESPONSE_TIME = Gauge("external_api_call_seconds", "Latency of third-party API calls")

def fetch_weather_data():
    # Measure execution time for this specific block
    with API_RESPONSE_TIME.time():
        # Trigger network request
        time.sleep(1.5)
```

---

## 3. Dynamic Callback Gauges (Advanced Pattern)

Usually, developers write background threads that update metrics in a loop:
```python
# Wasteful pattern: runs a loop constantly consuming CPU cycles
while True:
    GAUGE.set(get_cpu_usage())
    time.sleep(5)
```
This is inefficient. A better pattern is to use **Callback Gauges**. Instead of updating the gauge in a background loop, you assign a callback function to the gauge. When Prometheus scrapes your `/metrics` endpoint, the client library automatically calls your function to fetch the latest value on the fly.

This reduces CPU overhead and ensures metrics are collected exactly when needed.

```python
import psutil
from prometheus_client import Gauge

# 1. Define the callback function to fetch active system details
def get_system_cpu_usage():
    # Returns a float representing CPU usage percentage
    return psutil.cpu_percent()

# 2. Create the Gauge and pass the callback function using 'set_function'
# Note: Do not call the function when passing it; pass the function reference itself
CPU_USAGE_METRIC = Gauge(
    "system_cpu_utilization_percent",
    "Current CPU utilization percentage, fetched dynamically on scrape"
)
CPU_USAGE_METRIC.set_function(get_system_cpu_usage)

# When Prometheus requests /metrics, get_system_cpu_usage() is automatically run.
```

---

## Summary

Gauges are essential for tracking fluctuating values like resource usage or queue depth. By using timers and implementing dynamic callback functions, you can monitor system states efficiently without wasting CPU cycles. In the next section, we will cover **Custom Histograms** in detail.

# Custom Counters: Measuring Volume and Error Rates

A **Counter** is a cumulative metric that only increases or resets to zero upon application restart. It represents a monotonically increasing value. 

In this guide, we will cover how to declare counters, increment values in batches, categorize events using labels, protect your monitoring database from **high-cardinality** crashes, and use counters to track application exceptions.

---

## The Counter Lifecycle

```mermaid
graph TD
    AppStart[Application Start] -->|Reset| ValZero[Value: 0]
    ValZero -->|Event Occurs| Inc1[inc()]
    Inc1 -->|Value: 1| ValOne[Value: 1]
    ValOne -->|Batch Process| IncN[inc(10)]
    IncN -->|Value: 11| Val11[Value: 11]
    Val11 -->|Application Crashes / Restarts| ValZero
```

---

## 1. Declaring and Incrementing Counters

To create a counter, import `Counter` and define its name and description.

```python
from prometheus_client import Counter

# Declare the counter
IMAGE_PROCESSED = Counter(
    "image_processing_total",
    "Total number of image uploads processed"
)

def process_upload():
    # Increment the counter by 1
    IMAGE_PROCESSED.inc()
    print("Image processed successfully.")
```

### Incrementing by Custom Values
If you process logs or transfer data, you need to increment the counter by a dynamic value (like bytes or elements processed) rather than a fixed step of `1`. Pass the value as an argument to `inc()`:

```python
NETWORK_EGRESS_BYTES = Counter(
    "network_egress_bytes_total", 
    "Total volume of data transferred out of the system in bytes"
)

def stream_response(data_bytes):
    # Process stream...
    bytes_count = len(data_bytes)
    
    # Increment by the length of the data
    NETWORK_EGRESS_BYTES.inc(bytes_count)
```

---

## 2. Categorizing Metrics with Labels

To organize metric data without creating hundreds of individual variables, use **labels**. For example, you can create a single counter called `http_requests_total` with `method` and `status` labels:

```python
from prometheus_client import Counter

# Declare a counter with labels (specified as a list of strings)
HTTP_REQUESTS = Counter(
    "http_requests_total",
    "Total count of HTTP requests",
    ["method", "status"] # Label keys
)

def handle_request(method, status):
    # Increment the metric for the specific combination of label values
    HTTP_REQUESTS.labels(method=method, status=status).inc()
```

### The Threat of High-Cardinality
> [!CAUTION]
> **High Cardinality** is a major issue in time-series monitoring. Every unique combination of label values creates a new, separate time series in Prometheus.

* **Safe Labels**: `method` (GET, POST), `status` (200, 404, 500), `region` (us-east-1, eu-west-1). These have a small, defined set of values.
* **Dangerous Labels**: `user_id`, `email`, `ip_address`, `timestamp`. These have an infinite or extremely large number of possible values. If you use `user_id` as a label, and have 1 million users, Prometheus will try to index 1 million separate time series, causing the database to run out of memory and crash.

---

## 3. Counting Exceptions using Decorators and Context Managers

Counters can be used as decorators or context managers to automatically count the frequency of exceptions raised by your code.

### Using as a Decorator
```python
from prometheus_client import Counter

DB_CONNECTION_ERRORS = Counter(
    "db_connection_failures_total",
    "Total count of database connection errors caught"
)

# Apply the decorator
# This will automatically call DB_CONNECTION_ERRORS.inc() 
# if the decorated function raises an exception
@DB_CONNECTION_ERRORS.count_exceptions()
def connect_to_database():
    # Attempt connection logic
    # If connection fails, counter increments
    pass
```

### Using as a Context Manager
If you only want to track exceptions within a specific block of code inside a function:

```python
from prometheus_client import Counter

SMS_FAILURES = Counter("sms_send_failures_total", "SMS transmission errors")

def send_alert_notification():
    print("Initiating SMS send...")
    
    # Track exceptions only inside this context block
    with SMS_FAILURES.count_exceptions():
        # Trigger SMS API
        pass
```

---

## Summary

Counters are the primary metric for tracking system volume and error rates. By using labels safely and avoiding high-cardinality variables, you can categorize metrics without degrading performance. In the next section, we will cover **Custom Gauges** in detail.

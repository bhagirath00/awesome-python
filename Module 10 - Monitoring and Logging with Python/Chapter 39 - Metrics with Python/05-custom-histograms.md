# Custom Histograms: Analyzing Latency and Distributions

Measuring average latency (e.g., "our average API response time is 100ms") can mask issues. If 99% of your users receive responses in 10ms, but 1% experience a 10-second delay due to database locks, the average is skewed, and you miss the critical errors.

A **Histogram** solves this by sampling observations (usually response times or payload sizes) and counting them in configurable range **buckets**. This allows you to analyze distributions and calculate quantiles (like the p95 or p99 latency) on the Prometheus server using PromQL.

In this guide, we will cover how to declare histograms, configure custom linear and exponential buckets, use `observe()`, and compare histograms with summaries.

---

## The Histogram Bucket Structure

When you define a histogram named `request_duration_seconds`, Prometheus creates multiple time-series:
* **`request_duration_seconds_bucket{le="0.1"}`**: Count of observations less than or equal to 0.1s.
* **`request_duration_seconds_bucket{le="0.5"}`**: Count of observations less than or equal to 0.5s.
* **`request_duration_seconds_bucket{le="+Inf"}`**: Total count of all observations.
* **`request_duration_seconds_count`**: Total number of observations (identical to `le="+Inf"`).
* **`request_duration_seconds_sum`**: Cumulative sum of all observed values (used to calculate average latencies).

---

## 1. Declaring a Histogram and Observing Values

To record an observation, use the `observe()` method:

```python
from prometheus_client import Histogram
import time

# Declare the histogram
REQUEST_LATENCY = Histogram(
    "api_response_duration_seconds",
    "Time spent processing API requests"
)

def process_api_call():
    start = time.time()
    
    # Process request (simulated delay)
    time.sleep(0.15)
    
    elapsed = time.time() - start
    
    # Record elapsed time in the histogram
    REQUEST_LATENCY.observe(elapsed)
```

---

## 2. Configuring Custom Buckets (Linear & Exponential)

By default, Prometheus client creates buckets at standard ranges: `.005`, `.01`, `.025`, `.05`, `.075`, `.1`, `.25`, `.5`, `.75`, `1.0`, `2.5`, `5.0`, `10.0`, and `+Inf`.

If your application has different performance expectations (e.g. standard queries take around 200ms-500ms, or you are measuring file sizes in Megabytes), you should define custom buckets.

### Creating Manual Buckets
Pass a list or tuple of numbers to the `buckets` argument.

```python
from prometheus_client import Histogram

# Define custom boundaries manually
custom_boundaries = (0.05, 0.1, 0.25, 0.5, 1.0, 2.5)

HTTP_LATENCY = Histogram(
    "http_response_latency_seconds",
    "HTTP Latency in seconds",
    buckets=custom_boundaries
)
```

### Generating Exponential Buckets
The `prometheus-client` library provides helper functions to generate bucket boundaries automatically. **`exponential_buckets(start, factor, count)`** is highly useful for measuring latency.
* `start`: The value of the first bucket.
* `factor`: The multiplier to calculate the next bucket boundary.
* `count`: The total number of buckets to generate.

```python
from prometheus_client import Histogram, utils

# Generates buckets starting at 0.01, multiplying by 2 for 10 buckets
# Buckets generated: [0.01, 0.02, 0.04, 0.08, 0.16, 0.32, 0.64, 1.28, 2.56, 5.12]
exponential_ranges = utils.exponential_buckets(start=0.01, factor=2, count=10)

API_LATENCY = Histogram(
    "api_call_duration_seconds",
    "Duration of API calls",
    buckets=exponential_ranges
)
```

---

## 3. Histogram vs. Summary

| Feature / Metric | Histogram | Summary |
| :--- | :--- | :--- |
| **Quantiles Calculation**| On the *Prometheus Server* (via PromQL) | On the *Client Application* |
| **Quantile Aggregation** | Yes (Can merge values from multiple nodes) | No (Cannot average client-side percentiles) |
| **Resource Overhead** | Low CPU on client, requires index storage | Higher CPU overhead on client |
| **Error Margin** | Variable (Depends on bucket boundaries) | Precise (Calculated from raw data stream) |

---

## Complete API Middleware Decorator Example

Here is a practical decorator that measures execution time and records it in a custom exponential histogram:

```python
import time
from prometheus_client import Histogram, utils

# Define exponential buckets for sub-second API latencies
latency_buckets = utils.exponential_buckets(start=0.005, factor=2, count=8)

# Configure the histogram with labels for method and endpoint
API_LATENCY_HISTOGRAM = Histogram(
    "api_request_duration_seconds",
    "HTTP request latency in seconds",
    ["method", "endpoint"],
    buckets=latency_buckets
)

def track_api_latency(method, endpoint):
    """
    Decorator to track API response latency.
    """
    def decorator(func):
        def wrapper(*args, **kwargs):
            start_time = time.time()
            try:
                return func(*args, **kwargs)
            finally:
                duration = time.time() - start_time
                # Record duration with labels
                API_LATENCY_HISTOGRAM.labels(method=method, endpoint=endpoint).observe(duration)
        return wrapper
    return decorator

# --- Test standard decorated function ---
@track_api_latency(method="POST", endpoint="/checkout")
def handle_checkout(amount):
    # Simulate processing delay
    time.sleep(0.12)
    print("Checkout complete.")

# handle_checkout(100.0)
```

---

## Summary

Histograms are the standard tool for monitoring latency and response time profiles. By configuring custom linear or exponential buckets, you can track performance distributions and calculate accurate percentiles. In the next section, we will cover the **Prometheus Pushgateway**.

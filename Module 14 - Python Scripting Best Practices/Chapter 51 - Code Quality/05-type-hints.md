# Type Hints in Python: Enforcing Static Type Declarations

Python is a dynamically typed language. Variable types are resolved at runtime, not compile time. While this makes prototyping fast, it can introduce bugs in large codebases:
* A function expecting an integer (e.g. `timeout`) could receive a string from an environment configuration, raising a type error at runtime.
* IDE autocompletion cannot predict available object methods because it does not know the variable's type.

PEP 484 introduced **Type Hints** to Python. Type hints allow you to annotate code with expected data types. 

While Python still runs dynamically (type hints are ignored by the runtime interpreter), external type checkers (like mypy) use these annotations to catch bugs before execution, and IDEs leverage them to provide autocomplete suggestions.

In this guide, we will cover primitive types, container annotations using Python 3.10+ syntax, function signatures, and write a type-hinted configuration parser.

---

## Dynamic Typing vs. Static Type Hints

### Dynamic Typing (Standard Python)
```python
def configure_instance(instance_id, tags):
    # What data type is 'tags'? A list? A dictionary?
    # What does this function return?
    pass
```

### Static Type Hints (PEP 484)
```python
def configure_instance(instance_id: str, tags: dict[str, str]) -> bool:
    # 'instance_id' must be a string.
    # 'tags' must be a dictionary mapping strings to strings.
    # The function returns a boolean.
    pass
```

---

## 1. Type Hinting Syntax

### Primitives
Annotate variables by appending a colon (`:`) followed by the data type:

```python
port: int = 8080
hostname: str = "localhost"
is_active: bool = True
latency: float = 0.045
```

### Container Types (Python 3.10+)
In Python 3.10+, you can type containers using built-in classes directly:

```python
# A list of strings
active_hosts: list[str] = ["10.0.0.1", "10.0.0.2"]

# A dictionary mapping strings to integers
port_mapping: dict[str, int] = {"http": 80, "https": 443}

# A tuple containing an IP and a port
connection_details: tuple[str, int] = ("127.0.0.1", 8080)
```

### Union and Optional Types
If a variable can accept multiple types, use the pipe operator (`|`). To specify that a variable can be a type or `None`, append `| None` (which replaces `Optional[T]`):

```python
# Variable can be an integer or a string
timeout: int | str = 30

# Variable can be a string or None
dns_server: str | None = None
```

---

## 2. Practical Example: A Type-Hinted System Metrics Collector

Here is a system status collection script that uses Python 3.10+ type hints:

```python
# Save this file as system_monitor.py
import time

class SystemMetricsCollector:
    def __init__(self, host: str, interval: int = 5) -> None:
        self.host: str = host
        self.interval: int = interval
        self.metrics_history: list[dict[str, float]] = []

    def record_metrics(self, cpu_util: float, mem_util: float) -> bool:
        """
        Appends metrics dict to history. Returns True on success.
        """
        if cpu_util < 0.0 or mem_util < 0.0:
            return False
            
        payload: dict[str, float] = {
            "timestamp": time.time(),
            "cpu": cpu_util,
            "memory": mem_util
        }
        
        self.metrics_history.append(payload)
        return True

    def get_average_cpu(self) -> float | None:
        """
        Calculates the average CPU utilization. Returns None if history is empty.
        """
        if not self.metrics_history:
            return None
            
        total_cpu: float = sum(metric["cpu"] for metric in self.metrics_history)
        return total_cpu / len(self.metrics_history)
```

---

## Summary

Type hints add clarity and structure to dynamically typed Python code. By annotating primitive types, container structures, and function signatures, you can catch type mismatches early and improve developer productivity. In the next section, we will cover **mypy type checking**.

# Introduction to the psutil Library

While Python's standard library provides basic operating system details, it lacks advanced tools for monitoring system resources in real time. If you want to check CPU core utilization, monitor RAM usage percentage, read network bandwidth, or list running processes on a server, you will need a third-party library.

The standard tool for this is [psutil](https://github.com/giampaolo/psutil). The name stands for **Process and System Utilities**. It is a cross-platform library that allows you to inspect hardware performance metrics and manage active OS processes.

In this guide, we will look at how `psutil` works, how to install it, and how to query basic hardware resources.

---

## Why Use the `psutil` Library?

Before `psutil`, DevOps developers had to run shell commands (like `ps`, `top`, or `df` on Linux) inside a subprocess and parse the returned strings manually. This was slow and fragile. `psutil` accesses system metrics directly using C system APIs, making it fast and robust.

Here is how `psutil` compares to native system commands:

| Resource Checked | Linux Command | Windows Command | `psutil` Equivalent |
| :--- | :--- | :--- | :--- |
| **CPU Usage** | `top` / `htop` | Task Manager | `psutil.cpu_percent()` |
| **Memory Info** | `free -m` | Task Manager | `psutil.virtual_memory()` |
| **Disk Operations** | `iostat` | Performance Monitor | `psutil.disk_io_counters()` |
| **Active Processes** | `ps -aux` | `tasklist` | `psutil.process_iter()` |
| **Network Interfaces** | `ifconfig` / `ip a` | `ipconfig` | `psutil.net_if_addrs()` |

---

## Installing psutil

Because `psutil` is an external package, you must install it before you can import it in your scripts. You can install it using `pip`:

```bash
pip install psutil
```

Once installed, it compiles and binds directly to the hosting operating system APIs.

---

## Retrieving Hardware Statistics

Let us write a simple hardware health monitor script that uses `psutil` to query CPU, Memory, and Disk stats in one go.

```python
import psutil

# 1. CPU Core count and utilization percentage
# interval=1 means psutil will sample CPU usage over 1 second
cpu_usage = psutil.cpu_percent(interval=1)
cpu_cores = psutil.cpu_count(logical=True)

# 2. Memory / RAM stats
ram_info = psutil.virtual_memory()
total_ram_gb = ram_info.total / (1024 ** 3)
available_ram_gb = ram_info.available / (1024 ** 3)
ram_percentage = ram_info.percent

# 3. Disk space status
disk_info = psutil.disk_usage("/")
disk_percentage = disk_info.percent

print("System Health Dashboard")
print("=======================")
print(f"CPU Cores: {cpu_cores} | Current Load: {cpu_usage}%")
print(f"Memory (RAM): {available_ram_gb:.2f} GB free of {total_ram_gb:.2f} GB ({ram_percentage}% used)")
print(f"Root Disk space: {disk_percentage}% capacity used")
```

### Expected Output

```text
System Health Dashboard
=======================
CPU Cores: 8 | Current Load: 12.5%
Memory (RAM): 6.80 GB free of 16.00 GB (57.5% used)
Root Disk space: 42.1% capacity used
```

---

## Metric Discovery flow

Here is how `psutil` acts as a unified layer between your code and the operating system:

```mermaid
graph TD
    A[Python Script] --> B[psutil Library]
    B --> C{Detect OS API}
    C -- Linux --> D[/proc filesystem]
    C -- Windows --> E[WMI / Win32 API]
    C -- macOS --> F[sysctl calls]
    D --> G[Standard Python Objects]
    E --> G
    F --> G
    G --> H[Developer accesses properties]
```

---

## Real-World DevOps Scenario: System Health Guard

Here is a script that checks system stats. If any resource (CPU, Memory, or Disk) exceeds 90% utilization, the script raises a warning flag. This represents a typical host monitoring daemon.

```python
import psutil
import sys

def audit_host_resources():
    print("Initiating system health check audit...")
    print("--------------------------------------")
    
    # 1. Query metrics
    # We sample CPU load quickly
    cpu = psutil.cpu_percent(interval=0.5)
    memory = psutil.virtual_memory().percent
    disk = psutil.disk_usage("/").percent
    
    print(f"CPU: {cpu}% | RAM: {memory}% | Storage: {disk}%")
    
    # 2. Check thresholds
    warnings = []
    
    if cpu > 90.0:
        warnings.append(f"High CPU utilization: {cpu}%")
    if memory > 90.0:
        warnings.append(f"Low memory availability: {memory}% RAM used")
    if disk > 90.0:
        warnings.append(f"Storage critical: {disk}% capacity utilized")
        
    # 3. Report
    if warnings:
        print("\n[ALERT] CRITICAL HEALTH STATE DETECTED:")
        for warning in warnings:
            print(f"- {warning}")
        return False
    else:
        print("\n[OK] All hardware metrics are within safe parameters.")
        return True

# Run the health audit
audit_host_resources()
```

### Expected Output

```text
Initiating system health check audit...
--------------------------------------
CPU: 4.5% | RAM: 58.2% | Storage: 42.1%

[OK] All hardware metrics are within safe parameters.
```

---

## Common Mistakes

Here are some typical mistakes people make with `psutil`:

### 1. Calling `cpu_percent()` without an interval or loop pause
If you write `cpu_percent()` with no interval (or `interval=0` or `None`), the function returns the CPU load since the last time it was called. If it is the first call in your script, it will return `0.0` or a garbage number. Always provide an interval (like `interval=1` or `interval=0.5`) if you want a real snapshot measurement.

### 2. Forgetting that `psutil` is an external module
Because `psutil` is not built into Python, your scripts will fail with an `ImportError` on new target servers unless you install it first. In automated environments (like Ansible or Docker), ensure `psutil` is listed in your `requirements.txt` file.

### 3. Assuming path separators match on Windows
When calling `psutil.disk_usage()`, passing a forward slash `"/"` works on Linux, but on Windows you must target a drive (like `"C:\\"`). Passing `"/"` on Windows can sometimes work but can also throw exceptions depending on active drive mounts. Always check the OS format.

---

## Summary

* The [psutil](https://github.com/giampaolo/psutil) library provides advanced system resource and process monitoring tools in Python.
* It is a third-party package; install it using `pip install psutil`.
* Use `psutil.cpu_percent(interval=N)` to measure CPU utilization over N seconds.
* Check physical memory capacity and utilization using `psutil.virtual_memory()`.
* Inspect disk space details using `psutil.disk_usage(path)`.
* `psutil` retrieves metrics directly using OS APIs, bypassing the need to parse command-line outputs.
* Build resource checks using thresholds to alert administrators when metrics cross 90%.
* Always specify an interval when measuring CPU percentages to get an accurate load measurement.
* Remember to include `psutil` in dependency installation lists when deploying scripts to new servers.

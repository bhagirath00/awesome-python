# Remote Automation Patterns

When your infrastructure grows from a single server to clusters of dozens or hundreds of hosts, running scripts in serial (one after the other) becomes a major bottleneck. A deployment that takes 30 seconds on one server would take almost an hour on 100 servers.

To scale up, you must adopt professional automation patterns:
1. **Parallel Execution**: Running tasks across multiple hosts concurrently.
2. **Idempotency**: Ensuring that running the same script twice does not corrupt configurations.
3. **SSH Agent Forwarding**: Sharing credentials securely with nested child servers without copy-pasting keys.

In this guide, we will explore patterns for executing remote tasks in parallel, and how to forward credentials securely.

---

## Execution Strategies Comparison

When executing tasks across a cluster, you can choose different execution models:

| Strategy | Performance | Complexity | Best For | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Serial Execution** | Low (slowest) | Low | Diagnostic checks, database migrations. | Runs on host A, waits until done, then runs on host B. |
| **Thread Pool (`concurrent.futures`)** | High | Medium | Bulk parallel commands, multi-host updates. | Spawns multiple threads, each managing one SSH socket. |
| **Fabric Group (`ThreadingGroup`)** | High | Low | Cluster deployments, config syncs. | High-level Fabric wrapper that executes actions in parallel. |

---

## SSH Agent Forwarding Pattern

SSH Agent Forwarding allows a remote server to access your local SSH keys (via your local ssh-agent socket) to authenticate to other servers (like pulling code from a private Git repository).

```mermaid
graph TD
    A[Local Laptop: ssh-agent active] -->|SSH connect with Agent Forwarding| B[Jump Server / Bastion]
    B -->|Need private git repo access| C[Private Git Host]
    B -->|Requests key signature| A
    A -->|Signs challenge| B
    B -->|Authenticates successfully| C
    Note over B: No private keys are ever stored on Jump Server!
```

This prevents you from having to copy your private keys onto intermediate servers (like bastion hosts), which is a severe security risk.

---

## Parallel Command Execution with ThreadPoolExecutor

Let us write a script that runs a command on multiple hosts in parallel using Python's built-in `concurrent.futures.ThreadPoolExecutor` and Fabric connection objects.

```python
from concurrent.futures import ThreadPoolExecutor
from fabric import Connection

def run_remote_check(host):
    # Establish connection
    conn = Connection(host=host, user="admin")
    try:
        # Run command with a 3-second timeout limit
        result = conn.run("uptime", hide=True, timeout=3.0)
        return host, True, result.stdout.strip()
    except Exception as err:
        return host, False, str(err)
    finally:
        conn.close()

# List of target hosts in our fleet
servers = ["192.168.1.10", "192.168.1.11", "127.0.0.1"]

print("Starting parallel fleet diagnostics check...")
print("--------------------------------------------")

# Spawns a pool of up to 5 parallel threads
with ThreadPoolExecutor(max_workers=5) as executor:
    # Map the run function to all servers in our fleet
    results = executor.map(run_remote_check, servers)

for host, success, output in results:
    if success:
        print(f"[OK]      Host {host:<15} | Output: {output}")
    else:
        print(f"[FAILED]  Host {host:<15} | Error:  {output}")
```

### Expected Output

```text
Starting parallel fleet diagnostics check...
--------------------------------------------
[FAILED]  Host 192.168.1.10   | Error:  Unable to connect to 192.168.1.10:22
[FAILED]  Host 192.168.1.11   | Error:  Unable to connect to 192.168.1.11:22
[FAILED]  Host 127.0.0.1      | Error:  Unable to connect to 127.0.0.1:22
```

---

## Designing Idempotent Automation Scripts

In automated pipelines, scripts run repeatedly. If your script appends a configuration line to a file (like `echo "port=80" >> app.conf`), running the script ten times will append the line ten times, corrupting the configuration.

Ensure your remote scripts check state before making changes:

```python
# INSECURE - Appends duplicate entries
command_bad = "echo 'settings=enabled' >> /etc/app.conf"

# SECURE / IDEMPOTENT - Only appends if the setting does not exist
command_good = "grep -qF 'settings=enabled' /etc/app.conf || echo 'settings=enabled' >> /etc/app.conf"
```

---

## Common Mistakes

Here are some pitfalls when executing remote orchestrations:

### 1. Copy-pasting private keys onto Bastion servers
Never copy your private keys onto server file systems (like Jump boxes or Bastion servers) to authenticate nested SSH or Git calls. If that bastion box is compromised, attackers gain access to all systems. Use SSH Agent Forwarding instead.

### 2. Not handling individual thread failures
If you run tasks in parallel and one server fails (e.g. DNS timeout), your coordinator script should not crash. Ensure you wrap each thread task in a try-except block to catch failures individually so that other servers continue updating.

### 3. Setting Thread Worker counts too high
If you have 1,000 servers and spawn 1,000 threads simultaneously, your local script will exhaust file descriptors and local CPU cores, crashing your laptop. Limit parallel workers (e.g. `max_workers=20` or `30`) to batch the execution safely.

---

## Summary

* Parallel execution patterns scale remote command execution to multiple hosts.
* Use `concurrent.futures.ThreadPoolExecutor` to run tasks concurrently across threads.
* SSH Agent Forwarding allows remote servers to verify credentials using your local ssh-agent.
* Never copy or save private key files onto intermediate jump or bastion servers.
* Write idempotent command lines (like using `grep` check guards) to prevent duplicate runs from corrupting files.
* Wrap concurrent runner functions in individual try-except blocks to contain failures.
* Set thread pool worker limits (`max_workers`) to protect local system resources.
* Reconnect and run diagnostics in parallel to prevent cluster deploy delays.
* Set timeouts on all parallel client connections to prevent one dead host from hanging the coordinator.

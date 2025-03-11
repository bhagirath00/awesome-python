# Container Logs

When something goes wrong inside a container, logs are your first tool to figure out what happened. Whether a web server is returning 500 errors, a worker process is crashing silently, or a database is emitting warnings, the answers are in the container logs. Python's Docker SDK gives you both a one-shot log dump and a real-time streaming log tail.

In this guide, we will learn how to fetch container logs, stream them in real time, and filter them by time.

---

## Log Retrieval Modes Comparison

The Docker SDK supports two ways to read container logs. Each suits a different use case:

| Mode | How to use | Returns | Best For |
| :--- | :--- | :--- | :--- |
| **Snapshot** | `container.logs()` | Bytes (all at once) | Fetching historical logs after container exits |
| **Streaming** | `container.logs(stream=True)` | Generator (line by line) | Watching a running container in real time |
| **Timed** | `container.logs(since=..., until=...)` | Bytes for a time window | Debugging specific incidents |
| **With timestamps** | `container.logs(timestamps=True)` | Bytes with ISO timestamps prefixed | Correlating logs with external events |

---

## How Log Fetching Works

```mermaid
graph LR
    A[Container Process] -->|stdout/stderr| B[Docker Daemon Log Driver]
    B --> C{Log storage}
    C -->|json-file driver| D[/var/lib/docker/containers/ID/ID-json.log]
    D -->|GET /containers/ID/logs| E[Docker SDK]
    E -->|bytes or generator| F[Your Python Script]
```

Docker captures everything a container writes to `stdout` and `stderr`. By default, it uses the `json-file` log driver to save these on disk. The SDK's `container.logs()` method reads from that store.

---

## Fetching All Logs at Once

The simplest usage: grab everything the container has logged and print it.

```python
import docker
from docker.errors import NotFound, DockerException

def get_container_logs(container_name):
    try:
        client = docker.from_env()
        container = client.containers.get(container_name)
        
        print(f"Fetching logs for '{container_name}' (status: {container.status})...")
        
        # logs() returns bytes — decode to read as text
        raw_logs = container.logs(stdout=True, stderr=True)
        log_text = raw_logs.decode('utf-8').strip()
        
        if not log_text:
            print("No logs found for this container.")
        else:
            print(f"\n--- Logs for {container_name} ---")
            print(log_text)
            print("--- End of logs ---")
        
        client.close()
        return log_text

    except NotFound:
        print(f"Container '{container_name}' not found.")
        return None
    except DockerException as e:
        print(f"Error: {e}")
        return None

get_container_logs("dev-nginx")
```

### Expected Output
```text
Fetching logs for 'dev-nginx' (status: running)...

--- Logs for dev-nginx ---
2026/06/25 04:00:01 [notice] 1#1: nginx/1.25.3
2026/06/25 04:00:01 [notice] 1#1: built by gcc 12.2.1 (Alpine)
2026/06/25 04:00:01 [notice] 1#1: start worker processes
--- End of logs ---
```

---

## Fetching the Last N Lines (Tail)

When a container has been running for days and its log file is huge, you only want recent lines. Use the `tail` parameter.

```python
import docker
from docker.errors import NotFound, DockerException

def tail_logs(container_name, lines=50):
    try:
        client = docker.from_env()
        container = client.containers.get(container_name)
        
        # tail="all" fetches everything; tail=N fetches the last N lines
        raw_logs = container.logs(
            stdout=True,
            stderr=True,
            tail=lines,
            timestamps=True    # Prefix each line with an ISO timestamp
        )
        
        log_text = raw_logs.decode('utf-8').strip()
        print(f"Last {lines} lines from '{container_name}':")
        print(log_text)
        
        client.close()
        return log_text

    except NotFound:
        print(f"Container '{container_name}' not found.")
        return None
    except DockerException as e:
        print(f"Error: {e}")
        return None

tail_logs("redis-cache", lines=10)
```

### Expected Output
```text
Last 10 lines from 'redis-cache':
2026-06-25T04:05:01.123456789Z * Ready to accept connections tcp
2026-06-25T04:05:01.124000000Z * Server initialized
2026-06-25T04:05:03.999000000Z * DB loaded from disk: 0.001 seconds
```

---

## Real-World Script: Stream Logs in Real Time

For monitoring a container that is currently doing work (like a migration, a build, or a batch job), you want to watch logs as they come in, not wait for the job to finish.

```python
import docker
from docker.errors import NotFound, DockerException

def stream_logs(container_name):
    try:
        client = docker.from_env()
        container = client.containers.get(container_name)
        
        print(f"Streaming logs from '{container_name}' (Ctrl+C to stop)...\n")
        
        # stream=True returns a generator that yields log bytes as they arrive
        log_stream = container.logs(
            stdout=True,
            stderr=True,
            stream=True,
            timestamps=True
        )
        
        for log_chunk in log_stream:
            line = log_chunk.decode('utf-8').rstrip('\n')
            print(line)
        
        print(f"\nContainer '{container_name}' stopped producing logs.")
        client.close()

    except KeyboardInterrupt:
        print("\nLog streaming interrupted by user.")
    except NotFound:
        print(f"Container '{container_name}' not found.")
    except DockerException as e:
        print(f"Error: {e}")

stream_logs("db-migration-job")
```

### Expected Output
```text
Streaming logs from 'db-migration-job' (Ctrl+C to stop)...

2026-06-25T04:10:00.001Z Flyway 9.22.0 by Redgate
2026-06-25T04:10:00.120Z Database: jdbc:postgresql://db:5432/appdb
2026-06-25T04:10:01.400Z Successfully validated 12 migrations
2026-06-25T04:10:01.500Z Current version of schema: 1.0.011
2026-06-25T04:10:02.800Z Migrating schema to version 1.0.012
2026-06-25T04:10:03.200Z Successfully applied 1 migration.

Container 'db-migration-job' stopped producing logs.
```

---

## Common Mistakes

### 1. Forgetting to Decode Bytes
`container.logs()` always returns `bytes`, not a string. If you try to print or split it without `.decode('utf-8')`, you will see ugly byte string output like `b'...'`. Always decode immediately.

### 2. Streaming Logs From an Exited Container
`stream=True` on an exited container will not error, but the generator will yield all historical logs and then immediately stop — it will not wait for new output. This is actually useful to iterate all past logs from a one-shot container.

### 3. Not Setting Both `stdout=True` and `stderr=True`
By default, both are True, but if you explicitly set only one, you might miss logs. Many applications write errors to stderr and normal output to stdout. Capture both to get the complete picture.

### 4. Fetching Huge Logs Without a Tail Limit
On a container that has been running for weeks, `container.logs()` without `tail` will try to load the entire log file into memory. For long-running containers, always use `tail=N` or a time-based filter (`since=`, `until=`) to avoid memory issues.

---

## Summary

* `container.logs()` returns all container output as a single `bytes` object — decode with `.decode('utf-8')`.
* Use `tail=N` to limit output to the last N lines, which is essential for long-running containers.
* Use `timestamps=True` to prefix each line with an ISO 8601 timestamp for incident correlation.
* Use `stream=True` to get a generator that yields log chunks as they arrive — great for live monitoring.
* Set both `stdout=True` and `stderr=True` to capture all output types.
* You can stream logs from already-exited containers — the generator yields historical logs and finishes.
* Use `since` and `until` with Unix timestamps or `datetime` objects to fetch logs from a specific time window.
* Always wrap streaming in a `try/except KeyboardInterrupt` block so Ctrl+C exits cleanly.
* Log streaming runs synchronously in the main thread — use `threading` if you want non-blocking log watching.
* Docker's log size limits are configured via the `json-file` log driver options (`max-size`, `max-file`).

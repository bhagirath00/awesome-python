# Rotating Log Files: Size and Time-Based Log Rollouts

If an application runs continuously, its log files will continuously grow. Over time, a single log file can grow to several gigabytes, consuming all available disk space, crashing the server, and making search operations extremely slow.

To prevent this, production services must use **log rotation**. Log rotation splits logs into manageable chunks, renames old files (e.g., `app.log.1`, `app.log.2`), retains a history of recent files, and automatically deletes files older than a configured limit.

In this guide, we will cover the inner workings of size-based rotation (`RotatingFileHandler`), configure time-based daily rotations (`TimedRotatingFileHandler`), examine the file-locking limitations on Windows, and write a dual-rotator setup script.

---

## Size-Based Rotation Sequence

When a log file reaches the configured `maxBytes` threshold, the `RotatingFileHandler` triggers a rotation chain:

```text
Step 1: app.log.5 is deleted (reaches backupCount limit).
Step 2: app.log.4 is renamed to app.log.5
Step 3: app.log.3 is renamed to app.log.4
Step 4: app.log.2 is renamed to app.log.3
Step 5: app.log.1 is renamed to app.log.2
Step 6: The active app.log is closed and renamed to app.log.1
Step 7: A new, empty app.log is created for active logging.
```

This ensures that you never use more than: `maxBytes * (backupCount + 1)` bytes of disk space.

---

## 1. Implementing Size-Based Rotation (`RotatingFileHandler`)

Configure size-based rotation when you want to set strict limits on disk space consumption regardless of how long the program has been running.

```python
import logging
from logging.handlers import RotatingFileHandler

logger = logging.getLogger("size_backup")
logger.setLevel(logging.INFO)

# Configure size-based handler
# maxBytes=1048576 (1 Megabyte)
# backupCount=5 (retains app.log and app.log.1 through app.log.5)
handler = RotatingFileHandler(
    filename="app.log",
    maxBytes=1024 * 1024, # 1 MB
    backupCount=5,
    encoding="utf-8"
)

formatter = logging.Formatter("%(asctime)s - %(levelname)s - %(message)s")
handler.setFormatter(formatter)
logger.addHandler(handler)

# Test the rotation by writing multiple lines
for i in range(20000):
    logger.info(f"Test log line: {i} - monitoring size limits.")
```

---

## 2. Implementing Time-Based Rotation (`TimedRotatingFileHandler`)

Use time-based rotation when you want logs organized by date (e.g., having a clean file for every day of the week).

The `when` parameter accepts these time interval directives:
* `'S'`: Seconds
* `'M'`: Minutes
* `'H'`: Hours
* `'D'`: Days
* `'midnight'`: Rolls over exactly at midnight
* `'W0'` to `'W6'`: Weekday rollover (0 = Monday)

```python
import logging
from logging.handlers import TimedRotatingFileHandler

logger = logging.getLogger("daily_audit")
logger.setLevel(logging.WARNING)

# Rotate every day at midnight (when='midnight', interval=1)
# Keep up to 30 days of historical logs (backupCount=30)
# Files created look like: audit.log, audit.log.2026-06-26, audit.log.2026-06-27...
handler = TimedRotatingFileHandler(
    filename="audit.log",
    when="midnight",
    interval=1,
    backupCount=30,
    encoding="utf-8"
)

formatter = logging.Formatter("%(asctime)s [%(levelname)s] - %(message)s")
handler.setFormatter(formatter)
logger.addHandler(handler)

logger.warning("Daily timed audit log session initiated.")
```

---

## The Windows File-Locking Issue

> [!CAUTION]
> If you run log rotation on Windows, you may encounter a **`PermissionError`** when a rollover event occurs. 

On Windows, if a file is open by a process, the OS locks the file, preventing other operations from renaming or deleting it. If multiple threads, workers, or sub-processes share the same log file handler, the rollover rename command will fail.

### How to resolve this:
1. **Use `QueueHandler` / `QueueListener`**: (as covered in Section 03). Since only a single background listener thread has the log file open, file-lock conflicts are minimized.
2. **Use Syslog**: On Windows, direct logs to a syslog server or a database, avoiding file management completely.
3. **Use the `ConcurrentRotatingFileHandler` library**: A third-party drop-in replacement that uses file locks to safely coordinate multithreaded writes on Windows:
   ```bash
   pip install concurrent-log-handler
   ```

---

## Summary

Log rotation is a production requirement for long-running applications. Size-based rotators (`RotatingFileHandler`) protect against disk capacity issues, while time-based rotators (`TimedRotatingFileHandler`) help keep logs organized by date. In the next section, we will cover **Structured Logging**, which formats these logs as JSON for parsing.

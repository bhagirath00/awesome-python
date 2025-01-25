# Writing Files

In system automation, you will frequently need to save data back to disk. This includes writing test execution results, updating system settings, logging errors, or appending metrics to a dashboard file. Python provides simple methods to write new files or append data to existing ones.

In this guide, we will look at how to write files in Python. We will compare write and append modes, write multiple lines at once, and create a system log writer.

## Write Mode vs Append Mode

When you open a file for writing, you must tell Python how to handle existing files:
1. **Write Mode (`"w"`)**: Opens the file for writing. If the file already exists, Python **deletes all its content** (truncates it) before writing new data. If the file does not exist, Python creates it.
2. **Append Mode (`"a"`)**: Opens the file for writing. If the file exists, Python places the write cursor at the **end of the file**, preserving existing data. If the file does not exist, Python creates it.

Let's compare these two modes.

| Mode | Symbol | If File Exists? | If File Doesn't Exist? | DevOps Use Case |
| :--- | :--- | :--- | :--- | :--- |
| **Write** | `"w"` | Overwrites all content (Truncates) | Creates a new file | Creating fresh reports, overwriting configs |
| **Append** | `"a"` | Appends new text to the end | Creates a new file | Writing running log logs, audit entries |

Here is a diagram showing the difference in how these modes affect an existing file:

```mermaid
graph TD
    subgraph Write Mode 'w'
    A["Existing File: [line1]"] -->|open with 'w'| B["File is emptied: []"]
    B -->|write 'line2'| C["Final File: [line2]"]
    end
    
    subgraph Append Mode 'a'
    D["Existing File: [line1]"] -->|open with 'a'| E["Cursor placed at end"]
    E -->|write 'line2'| F["Final File: [line1, line2]"]
    end
```

## Writing to a New File

Let's look at how to create a new file and write text to it. Note that `.write()` does not add a newline character automatically; you must write `\n` yourself.

Create a file named `write_basics.py` and write the code below.

```python
# Open in write mode ('w')
file = open("report.txt", "w")

# Write text
file.write("=== System Setup Report ===\n")
file.write("Environment: Production\n")
file.write("Status:      Operational\n")

file.close()
print("Report file written successfully.")
```

Run this file in your terminal.

```bash
python3 write_basics.py
```

Expected output:
```text
Report file written successfully.
```

Let's read the report file to see what it wrote:
```bash
cat report.txt
```

Expected output:
```text
=== System Setup Report ===
Environment: Production
Status:      Operational
```

## Appending to an Existing File

Now, let's append a log line to our report file using append mode (`"a"`).

```python
# Open in append mode ('a')
file = open("report.txt", "a")

# Append a status check
file.write("Last Check:  Just Now\n")

file.close()
print("Append operation completed.")
```

Run the script.

```bash
python3 append_basics.py
```

Expected output:
```text
Append operation completed.
```

Let's verify the updated file:
```bash
cat report.txt
```

Expected output:
```text
=== System Setup Report ===
Environment: Production
Status:      Operational
Last Check:  Just Now
```

## Real-World Example: Automated System Log Writer

Let's write a script that acts as an activity logger. The function will accept a log level and a message. It will format the message with a date timestamp and append it to `system_activity.log`.

Create a file named `activity_logger.py` and write the code below.

```python
from datetime import datetime

log_filepath = "system_activity.log"

def log_event(level, message):
    # 1. Format timestamp
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    
    # 2. Build log line
    log_line = f"[{timestamp}] {level.upper()}: {message}\n"
    
    # 3. Open in append mode to preserve history
    file = open(log_filepath, "a")
    file.write(log_line)
    file.close()
    
    # Print to console for visibility
    print("Logged: " + log_line.strip())

# Simulate system events
log_event("info", "Web server gateway started.")
log_event("warning", "High connection load on proxy node.")
log_event("error", "Database sync failed.")
```

Run the script.

```bash
python3 activity_logger.py
```

Expected output:
```text
Logged: [2026-06-25 08:50:12] INFO: Web server gateway started.
Logged: [2026-06-25 08:50:12] WARNING: High connection load on proxy node.
Logged: [2026-06-25 08:50:12] ERROR: Database sync failed.
```

## Common Mistakes

### Unintentionally Overwriting Configuration Files
If you open an existing configuration file in write mode (`"w"`) when you meant to append, Python will delete the entire file contents instantly.

* **The Bad Way:**
```python
# This deletes all existing logs before writing!
file = open("application.log", "w")
file.write("New event log\n")
```

* **The Good Way:**
Use append mode `"a"` to add messages to the end of a log file safely.
```python
file = open("application.log", "a")
file.write("New event log\n")
```

### Writing Non-String Data Types directly
The `.write()` method only accepts strings. If you try to pass an integer or list directly, Python will crash with a `TypeError`.

* **The Bad Way:**
```python
port = 8080
file = open("settings.txt", "w")
# This raises a TypeError: write() argument must be str, not int
file.write(port)
```

* **The Good Way:**
```python
port = 8080
file = open("settings.txt", "w")
# Convert to string first
file.write(str(port) + "\n")
```

## Summary

* Open files for writing using `open(path, mode)`.
* Write mode (`"w"`) empties the file before writing new data.
* Append mode (`"a"`) adds new text to the end of the file, preserving history.
* The `.write()` method does not append a newline character `\n` automatically.
* Use `.writelines(list)` to write a list of strings to a file at once.
* Always close files using `.close()` to release the lock on the disk.
* Opening a file in `"w"` mode on an existing file deletes its content immediately.
* Convert numbers to strings before trying to write them to files.
* Writing logs with date timestamps helps troubleshoot system updates.
* Automatically logging execution details keeps your deployment history traceable.

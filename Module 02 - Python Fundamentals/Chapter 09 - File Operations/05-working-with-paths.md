# Working with File Paths

When writing automation scripts that run on different systems, you must handle file paths carefully. Windows uses backslashes (`\`) for folder paths, while Linux and Mac use forward slashes (`/`). If you hardcode paths using slashes, your scripts will crash when run on a different operating system. Python provides built-in modules to handle paths safely across all platforms.

In this guide, we will look at how to manage file paths in Python. We will learn about the legacy `os.path` module, joining paths safely, and checking if directories exist.

## The os.path Module

The built-in `os` module contains a sub-module called `os.path` that handles path calculations. It automatically detects the active operating system and formats paths accordingly.

Let's look at a comparison of common path functions.

| Function | What it does | Example | Output on Linux | Output on Windows |
| :--- | :--- | :--- | :--- | :--- |
| **`os.path.join()`** | Joins folders safely | `os.path.join("var", "log")` | `"var/log"` | `"var\log"` |
| **`os.path.exists()`**| Checks if path exists | `os.path.exists("app.log")`| `True` or `False` | `True` or `False` |
| **`os.path.isfile()`**| Checks if target is a file| `os.path.isfile("logs")` | `True` or `False` | `True` or `False` |
| **`os.path.isdir()`** | Checks if target is a folder| `os.path.isdir("logs")` | `True` or `False` | `True` or `False` |
| **`os.path.basename()`**| Gets filename from path | `os.path.basename("a/b.txt")`| `"b.txt"` | `"b.txt"` |
| **`os.path.dirname()`** | Gets folder name of file | `os.path.dirname("a/b.txt")` | `"a"` | `"a"` |

Here is a diagram showing how `os.path.join` formats paths dynamically:

```mermaid
graph TD
    A[Arguments: 'var', 'log', 'app.log'] --> B{os.path.join}
    B -->|Checks active OS| C{Is OS Windows?}
    C -- Yes --> D["var\log\app.log"]
    C -- No --> E["var/log/app.log"]
```

## Joining and Resolving Paths

Let's write a script that joins folders together and resolves absolute paths.

Create a file named `path_basics.py` and write the code below.

```python
import os

folder = "logs"
filename = "server.log"

# 1. Join paths safely
log_path = os.path.join(folder, filename)
print("Joined Path:   " + log_path)

# 2. Get the absolute path
abs_path = os.path.abspath(log_path)
print("Absolute Path: " + abs_path)

# 3. Split directory and file names
dir_name = os.path.dirname(log_path)
file_name = os.path.basename(log_path)
print("Folder:        " + dir_name)
print("Filename:      " + file_name)
```

Run this file in your terminal.

```bash
python3 path_basics.py
```

Expected output (on Linux/Mac):
```text
Joined Path:   logs/server.log
Absolute Path: /home/user/project/logs/server.log
Folder:        logs
Filename:      server.log
```

Expected output (on Windows):
```text
Joined Path:   logs\server.log
Absolute Path: C:\Users\patel\Github\devops\devops\logs\server.log
Folder:        logs
Filename:      server.log
```

## Auditing Directory Existence

Before you read or write files, you should verify if their directories exist.

```python
import os

target_dir = "backups"
target_file = os.path.join(target_dir, "db.tar.gz")

# Check if directory exists
if not os.path.exists(target_dir):
    print(f"Folder '{target_dir}' is missing. Creating directory...")
    os.makedirs(target_dir)

# Verify if target_file is a file
print("Is it a file? " + str(os.path.isfile(target_file)))
```

Run the script.

```bash
python3 check_paths.py
```

Expected output:
```text
Folder 'backups' is missing. Creating directory...
Is it a file? False
```

## Real-World Example: Safe Backup Path Generator

Let's write a script that generates a safe backup target path. The script resolves the folder containing our active script, creates a nested `backups/db/` directory relative to that folder, and outputs the absolute path where the backup archive should be saved.

Create a file named `backup_path_gen.py` and write the code below.

```python
import os

print("=== Resolving Backup Paths ===")

# 1. Get the directory containing the active script file
# '__file__' holds the path to the running script
script_dir = os.path.dirname(os.path.abspath(__file__))
print("Script directory: " + script_dir)

# 2. Define the backups sub-folder path
backup_target_dir = os.path.join(script_dir, "backups", "db")

# 3. Create folders if missing
if not os.path.exists(backup_target_dir):
    os.makedirs(backup_target_dir)
    print("Created nested backup folders.")
    
# 4. Generate final target filepath
backup_file = os.path.join(backup_target_dir, "backup-db-latest.tar.gz")
print("✅ Resolved Target Filepath:")
print(backup_file)
```

Run the script.

```bash
python3 backup_path_gen.py
```

Expected output:
```text
=== Resolving Backup Paths ===
Script directory: C:\Users\patel\Github\devops\devops
Created nested backup folders.
✅ Resolved Target Filepath:
C:\Users\patel\Github\devops\devops\backups\db\backup-db-latest.tar.gz
```

## Common Mistakes

### Joining Paths Using String Concatenation
Many beginners manually concatenate strings using slashes. This works on the system they wrote it on, but breaks on other operating systems.

* **The Bad Way:**
```python
# This breaks on Windows
filepath = "var" + "/" + "log" + "/" + "app.log"
```

* **The Good Way:**
Use `os.path.join()` which automatically loads the system-specific separator.
```python
import os
filepath = os.path.join("var", "log", "app.log")
```

### Using Relative Paths that Depend on Current Directory
If you use relative paths (like `"./config.json"`), the script searches for the file relative to where you are standing in the terminal when you run it, not relative to where the script is located. This will cause file missing errors in pipelines.

* **The Bad Way:**
```python
# This fails if you run the script from a different folder!
with open("config.json", "r") as file:
    pass
```

* **The Good Way:**
Resolve paths relative to the script directory using `__file__`.
```python
import os
script_dir = os.path.dirname(os.path.abspath(__file__))
config_path = os.path.join(script_dir, "config.json")
with open(config_path, "r") as file:
    pass
```

## Summary

* Windows uses backslashes (`\`) and Unix uses forward slashes (`/`) for directory paths.
* Hardcoding path slashes breaks script compatibility across operating systems.
* Use `os.path.join()` to merge folders together safely using system-specific slashes.
* Use `os.path.abspath()` to resolve a relative path to its absolute path on disk.
* Use `os.path.exists()` to check if a folder or file exists.
* Use `os.path.isfile()` and `os.path.isdir()` to identify the type of a path.
* Resolve paths relative to the script's actual folder using `__file__`.
* Avoid string concatenation for file paths.
* Check directory existence and create folders with `os.makedirs()` before writing files.
* Cross-platform path management is essential for write scripts in shared projects.

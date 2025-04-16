# Ansible-Runner Library: Programmatic Execution and Event Streaming

While calling `ansible-playbook` via Python's `subprocess` module is simple, it requires you to parse raw console log lines using regular expressions to determine execution states. For complex workflows, you need a cleaner programmatic interface.

**`ansible-runner`** is the official helper library maintained by Red Hat. It provides a stable Python API to run Ansible playbooks, modules, and roles. 

Instead of parsing raw console logs, `ansible-runner` exposes direct Python attributes to check execution statuses, retrieve host stats, and stream task events (such as task starting, passing, or failing) as Python dictionaries in real time.

In this guide, we will cover the directory structure required by `ansible-runner`, establish programmatic executions, and write an event listener script to monitor tasks in real time.

---

## The Private Data Directory Layout

`ansible-runner` isolates run configurations inside a structured directory called the **Private Data Directory**. This prevents configuration leakage and ensures runs are repeatable.

Before running a script, organize your project files in this layout:

```text
private_data_dir/
├── env/
│   ├── envvars (environment variables)
│   └── extravars (JSON/YAML file containing extra variables)
├── inventory/
│   └── hosts (Ansible host inventory file)
└── project/
    ├── playbooks/
    │   └── deploy_app.yml
    └── roles/
```

---

## 1. Installation

Install the library using pip:

```bash
pip install ansible-runner
```

Confirm the import is working in your environment:
```bash
python -c "import ansible_runner; print(ansible_runner.__file__)"
```

---

## 2. Programmatic Execution with `ansible_runner.run()`

To execute a playbook, call `ansible_runner.run()`, specifying the path to the private data directory and the target playbook filename.

```python
import ansible_runner

# Define paths relative to the execution script
private_data_directory = "./ansible_data"
playbook_file = "playbooks/deploy_app.yml"

print("Starting playbook execution via ansible-runner...")

# Execute the run command (synchronous, blocks until complete)
# To run asynchronously, use 'ansible_runner.run_async()'
r = ansible_runner.run(
    private_data_dir=private_data_directory,
    playbook=playbook_file,
    extravars={"app_version": "v1.2.0"}
)

# Audit the execution status
print("\n==================================================")
print("📋 EXECUTION RESULTS")
print("==================================================")
print(f"Status:      {r.status}") # 'successful', 'failed', or 'unreachable'
print(f"Return Code: {r.rc}")     # 0 for success, non-zero for failure
print(f"Stats:       {r.stats}")  # Recap metrics dict
print("==================================================")
```

The `r.stats` dictionary returns metrics directly, eliminating the need to parse stdout:
```json
{
  "processed": {"192.168.1.50": 1},
  "failures": {"192.168.1.50": 0},
  "ok": {"192.168.1.50": 4},
  "changed": {"192.168.1.50": 1},
  "skipped": {"192.168.1.50": 0},
  "unreachable": {"192.168.1.50": 0}
}
```

---

## 3. Streaming Real-Time Task Events

If you are building a custom CI/CD log viewer or web dashboard, you want to display task progress in real time (e.g. "Task 'Install Nginx' is running... passed!"). 

`ansible-runner` allows you to inspect the event logs as they occur.

```python
import ansible_runner
import time

private_data_directory = "./ansible_data"
playbook_file = "playbooks/deploy_app.yml"

print("Starting asynchronous execution with event monitoring...")

# 1. Start execution asynchronously (returns immediately)
thread, runner = ansible_runner.run_async(
    private_data_dir=private_data_directory,
    playbook=playbook_file
)

# 2. Poll the runner object and read events from the generator
# runner.events yields task execution dictionaries
seen_events = set()

while thread.is_alive():
    # Retrieve all events generated so far
    for event in runner.events:
        event_id = event.get("uuid")
        
        # Prevent repeating print outputs
        if event_id not in seen_events:
            seen_events.add(event_id)
            
            # Filter to display task starts and results
            event_name = event.get("event")
            
            if event_name == "runner_on_start":
                task_name = event.get("event_data", {}).get("task")
                print(f"▶️ Starting Task: {task_name}")
                
            elif event_name == "runner_on_ok":
                task_name = event.get("event_data", {}).get("task")
                host = event.get("event_data", {}).get("host")
                print(f"✅ Passed: {task_name} on {host}")
                
            elif event_name == "runner_on_failed":
                task_name = event.get("event_data", {}).get("task")
                host = event.get("event_data", {}).get("host")
                res = event.get("event_data", {}).get("res", {})
                msg = res.get("msg", "Unknown error")
                print(f"❌ Failed: {task_name} on {host}! Error: {msg}")
                
    time.sleep(1)

# Ensure thread completes
thread.join()
print(f"\nFinal status: {runner.status}")
```

---

## Summary

The `ansible-runner` library provides a stable Python API for executing configuration management tasks. By using the private data directory structure and reading the output metrics and event logs directly, you can build integrated deployment engines. In the next section, we will cover **Dynamic Inventory** generation using Python.

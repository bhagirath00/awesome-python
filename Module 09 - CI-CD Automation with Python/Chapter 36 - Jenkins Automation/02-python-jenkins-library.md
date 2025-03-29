# Python-Jenkins Library: Connecting and Diagnosing Jenkins Servers

While interacting with the Jenkins REST API using raw HTTP requests is fully functional, managing authorization headers, session states, URL encoding, and CSRF crumb updates manually is tedious.

To simplify these tasks, we use the **python-jenkins** library. This package wraps all Jenkins REST endpoints in a clean, pythonic class structure. It handles connection pooling, authentication, CSRF tokens, and JSON decoding automatically, letting you manage your build infrastructure with clean Python code.

In this guide, we will cover how to install python-jenkins, establish connection sessions, query server states, list jobs, and audit build nodes and executors.

---

## Installation

Install the library using pip inside your active virtual environment:

```bash
pip install python-jenkins
```

To confirm the library is installed and verify its version:

```bash
python -c "import jenkins; print(jenkins.__version__)"
```

---

## Connecting and Session Configurations

To connect to a Jenkins server, instantiate a `jenkins.Jenkins` object. You specify the server URL, username, and API Token.

```python
import jenkins

# Jenkins Connection Details
JENKINS_URL = "http://localhost:8080"
USERNAME = "admin"
API_TOKEN = "11a5d6f6e09398f481c0090c238b71d6f7"

# Initialize the server connection
# Under the hood, python-jenkins establishes connection protocols 
# and automatically fetches crumbs for future POST requests.
server = jenkins.Jenkins(JENKINS_URL, username=USERNAME, password=API_TOKEN)

try:
    # Get server version to verify connection
    version = server.get_version()
    print("✅ Successfully connected to Jenkins!")
    print(f"Server Version: {version}")
except jenkins.JenkinsException as e:
    print(f"❌ Connection failed: {e}")
except Exception as e:
    print(f"An unexpected error occurred: {e}")
```

---

## Inspecting Jobs and Understanding Status Colors

Jenkins does not use terms like "SUCCESS" or "FAILURE" to indicate job status in its global list. Instead, it uses **color codes**.

### Jenkins Job Color Coding Map:

| Color Code | Meaning | Equivalent Pipeline State |
| :--- | :--- | :--- |
| **`blue`** (or `anime`) | The last build completed successfully. | `SUCCESS` |
| **`red`** | The last build failed. | `FAILURE` |
| **`yellow`** | The last build completed with test failures. | `UNSTABLE` |
| **`grey`** | The job has never been run yet. | `NOT_BUILT` |
| **`disabled`** | The job is currently deactivated. | `DISABLED` |
| **`aborted`** | The last build was manually canceled. | `ABORTED` |
| **`anime` suffix** | The job is currently building (e.g. `red_anime`). | `BUILDING` |

Here is a Python script that retrieves all jobs and filters them by color status to find failing builds:

```python
import jenkins

server = jenkins.Jenkins("http://localhost:8080", username="admin", password="token")

try:
    # Retrieve list of all configured jobs
    # Returns a list of dictionaries: [{'name': 'job1', 'url': '...', 'color': 'blue'}]
    jobs = server.get_jobs()
    
    print(f"Total Configured Jobs: {len(jobs)}")
    print("-" * 40)
    
    failing_jobs = []
    building_jobs = []
    
    for job in jobs:
        name = job["name"]
        color = job["color"]
        
        # Check if the job is failing
        if color == "red":
            failing_jobs.append(name)
        # Anime indicates that the job is currently building
        elif "anime" in color:
            building_jobs.append(name)
            
    print(f"✅ Healthy / Passing Jobs: {len(jobs) - len(failing_jobs)}")
    print(f"🚨 Failing Jobs ({len(failing_jobs)}):")
    for job in failing_jobs:
        print(f" - {job}")
        
    print(f"🚀 Currently Active Builds ({len(building_jobs)}):")
    for job in building_jobs:
        print(f" - {job}")
        
except Exception as e:
    print(f"Failed to query jobs: {e}")
```

---

## Auditing Build Nodes and Executor Pools

A Jenkins server uses a master-agent architecture. The master coordinates jobs, while agents (nodes) run the builds. If all nodes are offline, jobs accumulate in the build queue, stalling your deployment pipeline.

You can inspect the status of your nodes and executors programmatically:

```python
import jenkins

server = jenkins.Jenkins("http://localhost:8080", username="admin", password="token")

try:
    # Get details about build nodes (runners)
    # Returns a list of dictionaries containing node descriptions
    nodes = server.get_nodes()
    
    print("--- Jenkins Build Nodes Audit ---")
    for node in nodes:
        name = node["name"]
        
        # 'master' is named '(master)' inside the node list
        node_name = "Master Node" if name == "(master)" else name
        
        # Fetch detailed status about this node
        # Returns a dict with offline, idle, and executor count keys
        info = server.get_node_info(name)
        
        is_offline = info.get("offline", False)
        is_idle = info.get("idle", False)
        num_executors = info.get("numExecutors", 0)
        
        status = "🔴 OFFLINE" if is_offline else "🟢 ONLINE"
        idle_status = "💤 Idle" if is_idle else "🛠️ Active / Processing"
        
        print(f"\nNode Name: {node_name}")
        print(f"Status:    {status}")
        print(f"Activity:  {idle_status}")
        print(f"Executors: {num_executors}")
        
except Exception as e:
    print(f"Failed to audit nodes: {e}")
```

---

## Summary

The `python-jenkins` library maps Jenkins resources to clean Python classes. By auditing jobs and build node states, you can track system health and build custom monitoring dashboards. In the next section, we will cover how to **trigger builds** programmatically.

# How Python Fits with Other Tools

When you start using Python, you will notice that it does not run in a vacuum. Instead, you will use Python to control, configure, and connect other tools in your infrastructure. Python acts as the common language that links various tools together.

In this guide, we will look at how Python works with popular tools like Ansible, Docker, and AWS. We will write real code to see how these integrations work, look at a diagram of the workflow, and cover common mistakes.

## Python and the Modern Toolchain

Let's look at a table to see how Python connects with some of the most popular tools used to manage servers and clouds.

| Tool | What it Does | How it Uses Python |
| :--- | :--- | :--- |
| **Ansible** | Configures servers | Runs Python scripts on remote hosts |
| **Boto3 (AWS)** | Manages AWS cloud | Official library for writing AWS automation scripts |
| **Pulumi** | Deploys cloud setups | Allows you to write infrastructure configuration in Python |
| **Docker SDK** | Manages containers | Python library to start, stop, and build containers |
| **Kubernetes Client** | Manages container clusters | Python library to control application deployments |

Here is a diagram showing how Ansible uses Python under the hood to configure a remote server over SSH:

```mermaid
graph TD
    A[Your Computer] -->|1. Run Ansible Playbook| B[Ansible Engine]
    B -->|2. Pack task as Python script| C[SSH Connection]
    C -->|3. Copy script to server| D[Remote Linux Server]
    D -->|4. Run script using remote Python| E[Applied Changes]
    E -->|5. Return status back| A
```

## Running External Commands

Sometimes you need your Python script to interact with tools that are already installed on your server, like Docker or Git. You can use Python's built-in `subprocess` module to run these command-line tools and read their results.

Let's write a script that runs the `docker --version` command on your system to see if Docker is running.

```python
import subprocess

print("Checking if Docker is installed...")

try:
    # Run the shell command and grab the output
    result = subprocess.run(
        ["docker", "--version"], 
        stdout=subprocess.PIPE, 
        stderr=subprocess.PIPE, 
        text=True, 
        timeout=5
    )
    
    if result.returncode == 0:
        print("Success: Docker is installed!")
        print("Output details: " + result.stdout.strip())
    else:
        print("Error: Docker command failed with exit code: " + str(result.returncode))
except FileNotFoundError:
    print("Warning: Docker was not found on your system path.")
except Exception as error:
    print("An unexpected error occurred: " + str(error))
```

Run this script in your terminal.

```bash
python check_docker.py
```

Expected output (if Docker is installed):
```text
Checking if Docker is installed...
Success: Docker is installed!
Output details: Docker version 24.0.7, build afdd53b
```

Expected output (if Docker is not installed):
```text
Checking if Docker is installed...
Warning: Docker was not found on your system path.
```

## Talking to Cloud Services

Most companies store their files and databases in the cloud. Amazon Web Services (AWS) provides an official library called Boto3. This library lets you write Python code to create and manage S3 storage buckets, EC2 instances, and RDS databases.

Let's write a script that uses a simulator class to show how Boto3 lists files in a cloud bucket.

```python
# A simple mock class to show how Boto3 behaves without needing cloud keys
class MockS3Client:
    def list_objects(self, bucket_name):
        print("Connecting to cloud S3 bucket: " + bucket_name)
        return {
            "Contents": [
                {"Key": "app-logs-2026-06-24.txt", "Size": 1024},
                {"Key": "app-logs-2026-06-25.txt", "Size": 2048}
            ]
        }

# Using our mock client
client = MockS3Client()
response = client.list_objects("company-production-logs")

for file_info in response["Contents"]:
    print("File: " + file_info["Key"] + " (Size: " + str(file_info["Size"]) + " bytes)")
```

Run the script.

```bash
python list_s3.py
```

Expected output:
```text
Connecting to cloud S3 bucket: company-production-logs
File: app-logs-2026-06-24.txt (Size: 1024 bytes)
File: app-logs-2026-06-25.txt (Size: 2048 bytes)
```

## Reading Local Settings

Configuration management tools like Ansible require you to keep track of server settings. Often, these settings are stored in simple files. Python makes it easy to read these files and use them to decide which servers to ping.

Let's write a script that reads a simple host configuration file and lists active web servers.

```python
# Create a test server list file
with open("hosts.txt", "w") as file:
    file.write("web-server-01:active\n")
    file.write("web-server-02:inactive\n")
    file.write("db-server-01:active\n")

print("Reading server settings...")
with open("hosts.txt", "r") as file:
    for line in file:
        parts = line.strip().split(":")
        server_name = parts[0]
        status = parts[1]
        
        if status == "active":
            print("Found active server: " + server_name)
```

Run this script.

```bash
python read_hosts.py
```

Expected output:
```text
Reading server settings...
Found active server: web-server-01
Found active server: db-server-01
```

## Common Mistakes

### Forgetting Subprocess Timeouts
When you run external commands using `subprocess.run`, the script will wait forever if the command gets stuck. Always specify a timeout limit in your code so your script will eventually exit if there is a problem.

* **The Bad Way:**
```python
import subprocess
# If git gets stuck waiting for a password, this hangs forever
subprocess.run(["git", "pull"])
```

* **The Good Way:**
```python
import subprocess
try:
    # Exit and raise an error if the command takes longer than 10 seconds
    subprocess.run(["git", "pull"], timeout=10)
except subprocess.TimeoutExpired:
    print("Error: Git command timed out.")
```

### Assuming Cloud Credentials Are Safe in Code
When using Boto3, do not write your AWS Access Key and Secret Key in your script. If you push the code to a Git repository, scanners will find your credentials and steal them. Always use the AWS credential file or environment variables instead.

* **The Bad Way:**
```python
# Writing keys directly makes them public to anyone reading the script
s3 = boto3.client('s3', aws_access_key_id='AKIAIOSFODNN7EXAMPLE')
```

* **The Good Way:**
```python
import boto3
# Boto3 automatically searches environment variables and credentials files
s3 = boto3.client('s3')
```

### Mixing Up System Python and Project Python
If you run a script that uses third-party libraries, you might get an error saying the module is missing. This happens when you install a package in a virtual environment but run the script using the system's global Python.

* **The Bad Way:**
```bash
# This uses the global python, which doesn't have your packages
python my_script.py
```

* **The Good Way:**
```bash
# Activate your virtual environment first
source myenv/bin/activate
python my_script.py
```

## Summary

* Python is the foundation for Ansible, which uses it to execute automation commands on remote hosts.
* AWS provides the Boto3 library so you can manage your cloud infrastructure using clean Python scripts.
* Pulumi lets you write your Infrastructure as Code in Python instead of learning custom languages.
* You can run existing shell commands from Python using the built-in `subprocess` module.
* Always define a timeout when running external commands to stop your scripts from hanging.
* Avoid hardcoding cloud credentials inside your scripts to prevent security leaks.
* Docker and Kubernetes both provide official Python libraries to manage containers and clusters.
* Using virtual environments helps prevent package conflicts between different infrastructure scripts.
* Python acts as the glue that coordinates server setup, cloud platforms, and container deployments.
* Writing small, clean scripts is the best way to coordinate multiple systems in your infrastructure.

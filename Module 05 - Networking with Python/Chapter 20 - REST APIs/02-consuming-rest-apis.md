# Consuming REST APIs with Python

Consuming a REST API means writing code that makes HTTP requests to a remote service, reads the returned data, and uses it inside your application. In DevOps, this is how you read server metrics, check deployment status on a cloud provider, or poll task states from a CI/CD platform.

Most modern REST APIs communicate using **JSON (JavaScript Object Notation)**. The `requests` library provides built-in tools to serialize Python dictionaries into JSON payloads when sending requests, and deserialize JSON strings back into Python dictionaries when receiving responses.

In this guide, we will look at how to consume REST APIs, parse JSON responses, and handle data lookups safely.

---

## Request Types and Data Passing

When consuming REST APIs, how you pass data depends on the HTTP method:

| HTTP Method | Data Parameter | Transport Location | Common Content-Type | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **`GET`** | `params=dict` | Encoded inside the URL. | None | Filtering resources (e.g. `?status=active`). |
| **`POST`** | `json=dict` | Encoded inside the HTTP Body. | `application/json` | Creating resources with detailed structures. |
| **`DELETE`** | `params=dict` / `None` | URL or Body. | None | Specifying which resource to delete. |

---

## JSON Deserialization Flow

Here is how Python parses a JSON API response:

```mermaid
graph TD
    A[Send HTTP GET Request] --> B[Receive Response Bytes]
    B --> C{Verify Content-Type Header?}
    C -->|application/json| D[Call response.json]
    C -->|text/html| E[Log Error & Skip Parsing]
    D --> F[Convert JSON array to Python list]
    D --> G[Convert JSON object to Python dict]
    F & G --> H[Access values: data['key']]
```

---

## Basic REST API Consumer Example

Let us write a script that queries a mock REST endpoint, reads the JSON response, and extracts specific fields. We will use the public `httpbin` service to inspect our request payload format.

```python
import requests

url = "https://httpbin.org/get"

# 1. Define query filters
filters = {
    "status": "terminated",
    "region": "us-east-1"
}

try:
    print(f"Querying instances from: {url}...")
    response = requests.get(url, params=filters, timeout=3.0)
    
    # 2. Check for HTTP errors
    response.raise_for_status()
    
    # 3. Deserialize JSON response
    # This automatically converts the JSON string to a Python dictionary
    data = response.json()
    
    # Extract query params returned by httpbin
    received_args = data.get("args", {})
    
    print("\nAPI Response Details:")
    print("---------------------")
    print(f"Returned Region: {received_args.get('region')}")
    print(f"Returned Status: {received_args.get('status')}")
    
except requests.exceptions.RequestException as err:
    print(f"API Connection failed: {err}")
```

### Expected Output

```text
Querying instances from: https://httpbin.org/get...

API Response Details:
---------------------
Returned Region: us-east-1
Returned Status: terminated
```

---

## Real-World DevOps Scenario: Fetching GitHub Repository Details

Let us write a script that queries GitHub's REST API to fetch metadata about a specific public repository (like the number of open issues or repository description), showing how to navigate nested JSON keys safely.

```python
import requests

def fetch_repository_metadata(owner, repo):
    url = f"https://api.github.com/repos/{owner}/{repo}"
    headers = {
        "Accept": "application/vnd.github.v3+json",
        "User-Agent": "DevOps-API-Consumer-v1"
    }
    
    try:
        print(f"Fetching metadata for: {owner}/{repo}...")
        response = requests.get(url, headers=headers, timeout=5.0)
        
        # Will raise HTTPError if repository is private or does not exist
        response.raise_for_status()
        
        # Parse JSON
        repo_data = response.json()
        
        # Accessing nested and root fields safely
        repo_name = repo_data.get("full_name")
        stars = repo_data.get("stargazers_count", 0)
        open_issues = repo_data.get("open_issues_count", 0)
        license_info = repo_data.get("license") or {} # license can be None
        license_name = license_info.get("name", "No License")
        
        print("\nRepository Specifications:")
        print("--------------------------")
        print(f"Name:        {repo_name}")
        print(f"Stars:       {stars}")
        print(f"Open Issues: {open_issues}")
        print(f"License:     {license_name}")
        
    except requests.exceptions.HTTPError as err:
        print(f"Failed to query repository: {err}")
    except requests.exceptions.RequestException as err:
        print(f"Network error: {err}")

# Run metadata audit
fetch_repository_metadata("ansible", "ansible")
```

### Expected Output

```text
Fetching metadata for: ansible/ansible...

Repository Specifications:
--------------------------
Name:        ansible/ansible
Stars:       58200
Open Issues: 2540
License:     GNU General Public License v3.0 or later
```

---

## Common Mistakes

Here are some typical gotchas when consuming REST APIs:

### 1. Hardcoding dictionary keys lookups
If you write `val = data['license']['name']`, and the license field is `None` or missing, your script will crash with a `KeyError` or `TypeError`. Always use the `.get()` method to provide default values when parsing API payloads:

```python
# WRONG - can raise KeyError
license_name = data["license"]["name"]

# CORRECT - safe navigation
license_name = data.get("license", {}).get("name", "None")
```

### 2. Calling `response.json()` without validating the response type
If an API gateway fails (returning a 502 Bad Gateway or 504 Gateway Timeout), it will often return an HTML error page. If you try to run `response.json()` on HTML text, the parser will raise a `requests.exceptions.JSONDecodeError`. Always call `response.raise_for_status()` first, which filters out error statuses before you parse.

### 3. Assuming JSON floats/ints are strings
JSON supports numbers natively. When Python decodes JSON, it translates numbers to Python integers or floats. If you try to concatenate it with a string (like `print("Stars: " + data['stargazers_count'])`), it will fail with a `TypeError`. Cast variables to strings using `str()` or use f-strings.

---

## Summary

* Consuming REST APIs involves sending HTTP calls and parsing returned payload data.
* Most modern REST APIs communicate using JSON formatting.
* Use `response.json()` to deserialize JSON strings into Python dictionaries automatically.
* Pass parameters using `params` for GET calls, and use `json` to send data in POST payloads.
* Check if a response contains JSON by inspecting the `Content-Type` header.
* Call `response.raise_for_status()` to catch HTTP errors before trying to parse body content.
* Protect against parser crashes by querying nested keys using the `.get()` method with default values.
* Cast numeric values returned in API payloads to strings before doing string concatenations.
* Set timeouts on all consumer requests to prevent blocking background automation.

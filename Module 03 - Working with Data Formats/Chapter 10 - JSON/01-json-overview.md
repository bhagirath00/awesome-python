# Understanding JSON

When you write scripts to automate infrastructure, you will constantly need to send data between different systems. For example, you might query a cloud provider API to get a list of servers, or write configuration settings for a web application. The industry-standard format for sharing this data is JSON (JavaScript Object Notation). 

In this guide, we will learn how JSON works. We will compare JSON and Python data structures, understand formatting rules, and write a simple script to parse JSON text.

## What is JSON?

JSON is a lightweight, text-based format for storing and exchanging data. It is easy for humans to read and write, and easy for computers to parse and generate. JSON is completely language-independent, meaning you can generate JSON in Python and read it in Go, Java, or JavaScript without any translation tools.

Let's compare JSON data types to their Python equivalents.

| JSON Data Type | Example JSON Value | Python Equivalent | Example Python Value |
| :--- | :--- | :--- | :--- |
| **Object** | `{"host": "web-01"}` | `dict` | `{"host": "web-01"}` |
| **Array** | `["web-01", "db-01"]`| `list` | `["web-01", "db-01"]` |
| **String** | `"active"` | `str` | `"active"` |
| **Number** | `8080` or `99.5` | `int` or `float` | `8080` or `99.5` |
| **Boolean** | `true` or `false` | `bool` | `True` or `False` |
| **Null** | `null` | `NoneType` | `None` |

Here is a diagram showing how JSON data maps to Python data structures when parsed:

```mermaid
graph TD
    subgraph Raw JSON String
    A['{ "port": 80, "active": true, "tags": null }']
    end
    
    subgraph Parsed Python Dictionary
    B["dict: { 'port': 80, 'active': True, 'tags': None }"]
    end
    
    A -->|json.loads| B
```

## Basic JSON Rules

JSON has strict syntax rules that you must follow, or else parsers will fail to read your data:
1. **Double Quotes Only**: All string values and key names must be wrapped in double quotes (`"`). Single quotes (`'`) are invalid in JSON.
2. **Boolean Cases**: Booleans must be written in lowercase: `true` or `false`.
3. **No Trailing Commas**: You cannot leave a comma after the last item in an object or array.
4. **Null Value**: Missing values must be written as `null` in lowercase.

Create a file named `json_basics.py` and write the code below.

```python
import json

# A valid JSON string
# Note the double quotes inside the single-quoted Python string
raw_json = '{"server_name": "prod-app-01", "port": 80, "is_active": true}'

print("Loading JSON string...")
# Convert JSON text to a Python dictionary
server_info = json.loads(raw_json)

print("Parsed data type: " + type(server_info).__name__)
print("Server Name:      " + server_info["server_name"])
print("Port Number:      " + str(server_info["port"]))
print("Is Active?        " + str(server_info["is_active"]))
```

Run this file in your terminal.

```bash
python3 json_basics.py
```

Expected output:
```text
Loading JSON string...
Parsed data type: dict
Server Name:      prod-app-01
Port Number:      80
Is Active?        True
```

## Real-World Example: Server Profile Metadata

Let's look at a structured JSON payload representing a server profile. We will parse this metadata, inspect values, and verify if the node matches our compliance guidelines.

Create a file named `audit_json.py` and write the code below.

```python
import json

# Simulated JSON payload from an API response
api_response = """
{
    "cluster": "us-east-production",
    "nodes_count": 2,
    "firewall_rules": ["allow-ssh", "allow-http"],
    "settings": {
        "auto_scale": false,
        "max_load": 85.0
    }
}
"""

def audit_cluster_payload(raw_text):
    print("=== Auditing JSON Configurations ===")
    
    try:
        # Parse the JSON string
        data = json.loads(raw_text)
        
        cluster_name = data["cluster"]
        nodes = data["nodes_count"]
        scale_flag = data["settings"]["auto_scale"]
        
        print(f"Cluster:        {cluster_name}")
        print(f"Active Nodes:   {nodes}")
        print(f"Auto-scale:     {scale_flag}")
        
        # Check rule parameters
        if nodes < 3 and not scale_flag:
            print(" ⚠️  COMPLIANCE WARNING: Small cluster has auto-scale disabled!")
        else:
            print(" ✅ Compliance Check Passed.")
            
    except json.JSONDecodeError as err:
        print(" ❌ Error: Invalid JSON syntax detected! Details: " + str(err))

audit_cluster_payload(api_response)
```

Run the script.

```bash
python3 audit_json.py
```

Expected output:
```text
=== Auditing JSON Configurations ===
Cluster:        us-east-production
Active Nodes:   2
Auto-scale:     False
 ⚠️  COMPLIANCE WARNING: Small cluster has auto-scale disabled!
```

## Common Mistakes

### Using Single Quotes in JSON Strings
A common mistake is writing key names or values using single quotes. While this is valid in Python, it is invalid in JSON and will crash the parser.

* **The Bad Way:**
```python
import json
# Invalid JSON syntax
bad_json = "{'host': 'localhost'}"
# This raises a JSONDecodeError
json.loads(bad_json)
```

* **The Good Way:**
```python
import json
# Correct double quotes
good_json = '{"host": "localhost"}'
json.loads(good_json)
```

### Leaving Trailing Commas
If you leave a trailing comma after the last key-value pair or array item, JSON parser tools will throw a decode error.

* **The Bad Way:**
```json
// Invalid JSON
{
    "ip": "10.0.0.1",
    "port": 80, // Error: Trailing comma not allowed here
}
```

* **The Good Way:**
```json
// Valid JSON
{
    "ip": "10.0.0.1",
    "port": 80
}
```

### Capitalizing Booleans or Null Values
In JSON, booleans and null placeholders must be written in lowercase. Writing `True`, `False`, or `None` (which are valid in Python) will cause parsing failures.

* **The Bad Way:**
```python
import json
# Raises a JSONDecodeError: Expecting value
bad_json = '{"active": True, "details": None}'
json.loads(bad_json)
```

* **The Good Way:**
```python
import json
# Lowercase true and null
good_json = '{"active": true, "details": null}'
json.loads(good_json)
```

## Summary

* JSON (JavaScript Object Notation) is a lightweight, text-based data format used by web APIs and configs.
* JSON is language-independent, allowing different programs to exchange data.
* JSON objects map directly to Python dictionaries, and JSON arrays map to Python lists.
* All string values and keys in JSON must be wrapped in double quotes (`"`).
* JSON booleans (`true`, `false`) and nulls (`null`) must be lowercase.
* Trailing commas are strictly forbidden in JSON arrays and objects.
* Use `json.loads(text)` to convert a JSON string into a Python dictionary.
* The `JSONDecodeError` exception is raised when parsing invalid JSON text.
* Catching syntax issues prevents scripts from crashing when reading corrupt API responses.
* Parsing JSON configurations dynamically helps automate infrastructure audits.

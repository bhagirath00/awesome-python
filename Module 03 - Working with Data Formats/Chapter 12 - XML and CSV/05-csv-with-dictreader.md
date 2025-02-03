# CSV Operations with DictReader and DictWriter

When working with CSV files in DevOps, referencing columns by their numeric index (like `row[0]` or `row[2]`) can quickly lead to bugs. If someone edits the CSV template and moves or adds a new column, your indexes will break, causing your automation scripts to fail.

To prevent this, Python provides the [csv.DictReader](https://docs.python.org/3/library/csv.html#csv.DictReader) and [csv.DictWriter](https://docs.python.org/3/library/csv.html#csv.DictWriter) classes. These tools let you read and write CSV rows as dictionaries instead of lists, mapping data directly to header names.

---

## Why Use Dictionary-Based Readers and Writers?

* **Flexibility**: If columns are reordered, your code still works.
* **Readability**: Writing `row['ip_address']` is much clearer to a teammate than `row[4]`.
* **Safety**: You explicitly list your fields, making debugging easier.

---

## How DictReader Maps CSV Data

Here is a visual map showing how a CSV line is turned into a dictionary:

```mermaid
graph TD
    A[CSV Row: web-prod-01, 10.0.0.1, active] --> B{csv.DictReader}
    B --> C["{'hostname': 'web-prod-01', 'ip': '10.0.0.1', 'status': 'active'}"]
    C --> D[Access via row['hostname']]
```

---

## Reading CSVs with DictReader

Imagine we have a CSV file named `users.csv` with these contents:

```text
username,uid,home_dir
alice,1001,/home/alice
bob,1002,/home/bob
charlie,1003,/home/charlie
```

We can use [csv.DictReader](https://docs.python.org/3/library/csv.html#csv.DictReader) to read these values as dictionary items. Python automatically reads the first line of the CSV as the dictionary keys.

```python
import csv

with open("users.csv", mode="r", encoding="utf-8", newline="") as file:
    reader = csv.DictReader(file)
    
    # We do not need to call next() to skip the header. 
    # DictReader uses the header line to set the keys automatically.
    for row in reader:
        # Access each column by its header name
        username = row["username"]
        uid = row["uid"]
        home = row["home_dir"]
        
        print(f"User: {username} has UID {uid} and home directory at {home}")
```

### Expected Output

```text
User: alice has UID 1001 and home directory at /home/alice
User: bob has UID 1002 and home directory at /home/bob
User: charlie has UID 1003 and home directory at /home/charlie
```

---

## Writing CSVs with DictWriter

To write a CSV from a list of dictionaries, use [csv.DictWriter](https://docs.python.org/3/library/csv.html#csv.DictWriter). When you create a `DictWriter` object, you must specify the column headers using the `fieldnames` parameter.

Here is how to create a file called `cloud_inventory.csv`:

```python
import csv

# Define the columns in the exact order you want them in the file
headers = ["instance_id", "instance_type", "region", "status"]

# Prepare our list of dictionaries
instances = [
    {"instance_id": "i-09f1a23b", "instance_type": "t3.micro", "region": "us-east-1", "status": "running"},
    {"instance_id": "i-08c4e56d", "instance_type": "m5.large", "region": "eu-west-1", "status": "stopped"},
    {"instance_id": "i-07d8e90f", "instance_type": "t3.small", "region": "us-west-2", "status": "running"}
]

with open("cloud_inventory.csv", mode="w", encoding="utf-8", newline="") as file:
    # Pass the fieldnames list to the writer
    writer = csv.DictWriter(file, fieldnames=headers)
    
    # You must call writeheader() to write the field names as the first row
    writer.writeheader()
    
    # Write all the dictionaries
    writer.writerows(instances)

print("Inventory CSV created successfully!")
```

### Expected Output File (`cloud_inventory.csv`)

```text
instance_id,instance_type,region,status
i-09f1a23b,t3.micro,us-east-1,running
i-08c4e56d,m5.large,eu-west-1,stopped
i-07d8e90f,t3.small,us-west-2,running
```

---

## Comparison: List-Based vs. Dictionary-Based

Let us compare the two ways of working with CSVs in Python:

| Feature | `csv.reader` / `csv.writer` | `csv.DictReader` / `csv.DictWriter` |
| :--- | :--- | :--- |
| **Data representation** | List of strings | Dictionary (keys mapped to headers) |
| **Access method** | Numeric index: `row[1]` | Dictionary key: `row['ip_address']` |
| **Resilience to column changes** | Low (breaks if columns are added/moved) | High (handles column reordering easily) |
| **Header configuration** | Manual skipping or writing | Automatic based on file or parameters |
| **Speed** | Slightly faster | Slightly slower (due to dictionary creation) |

---

## Real-World DevOps Scenario: Filtering Cloud Resources

Imagine you have a large inventory file called `all_servers.csv` containing server details. You need to read this file, filter out only the servers that are offline, and write them to a new cleanup report file called `cleanup_report.csv`.

Here is the source `all_servers.csv`:

```text
id,name,environment,state
1,web-01,production,running
2,test-01,development,terminated
3,db-01,production,running
4,stage-01,staging,terminated
```

Here is the Python script to filter and write the terminated servers:

```python
import csv

input_file = "all_servers.csv"
output_file = "cleanup_report.csv"

# We only want to save specific fields to our cleanup report
output_fields = ["id", "name", "state"]
terminated_servers = []

# Read and filter
with open(input_file, mode="r", encoding="utf-8", newline="") as file:
    reader = csv.DictReader(file)
    for row in reader:
        if row["state"] == "terminated":
            # Extract only the keys we need
            filtered_row = {
                "id": row["id"],
                "name": row["name"],
                "state": row["state"]
            }
            terminated_servers.append(filtered_row)

# Write output file
with open(output_file, mode="w", encoding="utf-8", newline="") as file:
    writer = csv.DictWriter(file, fieldnames=output_fields)
    writer.writeheader()
    writer.writerows(terminated_servers)

print(f"Cleanup report saved! Found {len(terminated_servers)} servers to clean up.")
```

### Expected Output File (`cleanup_report.csv`)

```text
id,name,state
2,test-01,terminated
4,stage-01,terminated
```

---

## Common Mistakes

Here are some errors to watch out for:

### 1. Dictionary keys are case-sensitive
If your CSV header has `IP_Address` and you write `row['ip_address']`, Python will throw a `KeyError`. Always double-check capitalization.

### 2. Forgetting `writeheader()`
If you do not call `writer.writeheader()` inside a `DictWriter` block, Python will write the rows of data, but there will be no header row. The output file will look like raw data, making it difficult for other tools to parse.

### 3. Extra keys in your dictionary
If your dictionary contains keys that are not listed in the `fieldnames` parameter when using `DictWriter`, Python will raise a `ValueError` (specifically "dict contains fields not in fieldnames").

If you want to ignore extra keys without raising an error, you can set the `extrasaction` parameter to `"ignore"`:

```python
# This ignores extra keys in dictionaries instead of crashing
writer = csv.DictWriter(file, fieldnames=headers, extrasaction="ignore")
```

---

## Summary

* `DictReader` maps CSV rows to dictionaries, using header row values as keys.
* `DictWriter` writes dictionary rows to a CSV file.
* Use dictionary-based CSV tools to write scripts that do not break when column positions change.
* DictReader does not need explicit header skipping; it parses the header row automatically.
* You must define the `fieldnames` list when creating a `DictWriter` object.
* Always call `writer.writeheader()` to write the header row when using DictWriter.
* Remember that header dictionary keys are case-sensitive.
* Use `extrasaction='ignore'` if your input dictionaries have more keys than your CSV headers.
* Set `newline=""` when opening files for writing or reading to preserve line-endings.

# Sending Logs to Elasticsearch: Direct API Ingestion and Bulk Uploads

**Elasticsearch** is a distributed, JSON-based search and analytics engine designed to store, index, and query large volumes of log data. It forms the core of many log aggregation stacks (such as the ELK or OpenSearch stacks).

While log shippers (like Filebeat or Fluentd) are typically used to collect local log files, you can use Python to send log documents directly to Elasticsearch from your code, or build custom log ingestion pipelines.

In this guide, we will connect to an Elasticsearch cluster using the official client, index individual log documents, and perform high-performance **Bulk Indexing** to batch-upload thousands of records in a single request.

---

## direct Ingestion Architecture

```mermaid
graph LR
    LogParser[Python Stateful Parser] -->|Extract JSON Records| BulkShipper[Bulk Ingestion Script]
    BulkShipper -->|HTTP POST _bulk| ES[Elasticsearch Cluster]
    ES -->|Index & Store| Kibana[Kibana Dashboards]
```

---

## 1. Connecting with Authentication and SSL

To interact with Elasticsearch, install the official client library:

```bash
pip install elasticsearch
```

When connecting to production Elasticsearch instances, you must authenticate using basic credentials (username/password) or an API Key. You should also enable SSL verification.

```python
from elasticsearch import Elasticsearch

# Initialize the client
# If security is enabled, pass basic auth or api_key parameters
es = Elasticsearch(
    "https://localhost:9200",
    basic_auth=("elastic", "super_secret_admin_password"),
    verify_certs=True,                # Enforce SSL verification in production
    ssl_show_warn=False
)

# Test the connection
try:
    cluster_info = es.info()
    print("✅ Connected to Elasticsearch cluster successfully!")
    print(f"Cluster Name: {cluster_info['cluster_name']}")
    print(f"ES Version: {cluster_info['version']['number']}")
except Exception as e:
    print(f"❌ Connection failed: {e}")
```

---

## 2. Indexing a Single Log Document

To index (save) a log entry, convert it to a dictionary and pass it to the `index` method, specifying the target index name (which acts like a table name).

```python
from datetime import datetime
from elasticsearch import Elasticsearch

es = Elasticsearch("https://localhost:9200", basic_auth=("elastic", "pass"), verify_certs=False)

# Define the log payload document
log_document = {
    "timestamp": datetime.utcnow().isoformat() + "Z",
    "level": "WARNING",
    "service": "checkout-api",
    "message": "Payment gateway response delayed",
    "metadata": {
        "gateway": "stripe",
        "response_time_ms": 2500,
        "environment": "production"
    }
}

try:
    # Index the document
    response = es.index(index="application-logs-prod", document=log_document)
    print(f"✅ Indexed document. Result: {response['result']}, ID: {response['_id']}")
except Exception as e:
    print(f"Failed to index document: {e}")
```

---

## 3. High-Performance Bulk Ingestion

Sending logs one-by-one is slow because of network round-trip times. To ingest thousands of log entries efficiently, use the `helpers.bulk()` utility to batch-upload documents in a single HTTP request.

We will use a generator to yield bulk action dictionaries. This keeps memory usage low by streaming logs from disk instead of loading all objects into a list before uploading.

```python
from datetime import datetime
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk

# Initialize client
es = Elasticsearch("https://localhost:9200", basic_auth=("elastic", "pass"), verify_certs=False)

def log_document_generator(filepath):
    """
    Reads local parsed logs and yields bulk action dictionaries.
    """
    # Sample file parser loop
    with open(filepath, "r", encoding="utf-8") as f:
        for idx, line in enumerate(f):
            # Parse line (Simulated simple splitting)
            parts = line.strip().split(" | ")
            if len(parts) < 3:
                continue
                
            # Yield the bulk ingestion action structure
            yield {
                "_index": "application-logs-prod",
                "_source": {
                    "timestamp": datetime.utcnow().isoformat() + "Z",
                    "level": parts[0],
                    "service": "worker-pool",
                    "message": parts[1],
                    "line_index": idx
                }
            }

def run_bulk_shipper(log_file):
    try:
        print(f"Starting bulk ingestion from {log_file}...")
        
        # bulk takes the client and an iterable of actions
        success_count, errors = bulk(es, log_document_generator(log_file))
        
        print(f"✅ Successfully indexed {success_count} documents.")
        if errors:
            print(f"⚠️ Encountered errors on some records: {errors}")
            
    except Exception as e:
        print(f"Bulk ingestion failed: {e}")

# Example:
# run_bulk_shipper("./app_events.txt")
```

---

## Summary

Ingesting structured logs into Elasticsearch allows you to search and analyze logs using Kibana. By utilizing bulk indexing and generator patterns, you can build fast, resource-efficient ingestion pipelines. This concludes **Module 10 - Monitoring and Logging with Python**.

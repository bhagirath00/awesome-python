# Fixtures in Pytest: Dependency Injection and Scope Lifecycles

In testing, **Fixtures** provide a stable, consistent environment for your test functions. They allow you to initialize database connections, prepare temporary configurations, or instantiate client helper classes.

Pytest handles fixtures using a **Dependency Injection** model: test functions declare the fixtures they need by listing them as input arguments, and Pytest resolves and injects them at runtime.

In this guide, we will cover fixture scopes, configure cleanup using `yield`, implement parameterized fixtures to test multiple inputs, and write a test suite for a database client.

---

## Fixture Scope Lifecycle

Pytest manages fixture lifetimes using the `scope` parameter:

```text
               ▲
              / \
             /   \      session - Runs once per test run (e.g. S3 clients, Vault connections).
            / sess\
           /-------\
          /  mod-   \   module - Runs once per test file (e.g. database schema setups).
         /   ule     \
        /-------------\
       /    class      \  class - Runs once per test class.
      /-----------------\
     /     function      \  function (default) - Runs before every test method (e.g. temp directories).
    /---------------------\
```

---

## 1. Cleanup Actions with `yield`

If your fixture needs to perform cleanup tasks after a test completes (like closing a socket connection), use `yield` instead of `return`. Pytest executes code before the `yield` statement before the test runs, and runs the code after `yield` after the test completes.

```python
import pytest

@pytest.fixture
def resource_client():
    # 1. SETUP: Initialize resource
    client = connect_to_system()
    
    # Yield the resource object to the test function
    yield client
    
    # 2. TEARDOWN: Clean up resource
    client.close()
```

---

## 2. Testing Multiple Inputs with Parameterized Fixtures

If you need to test your code against multiple configurations (e.g. verifying a script works on SQLite, PostgreSQL, and MySQL connections), use **parameterized fixtures** (`params=[...]`).

Pytest executes each test function multiple times—once for each parameter in the fixture.

---

## 3. Practical Example: Testing a Database Helper Client

Here is a database helper class, followed by a Pytest test suite that uses parameterized fixtures and scopes to verify its logic:

```python
# Save this file as db_helper.py
import sqlite3

class SQLiteDatabaseHelper:
    def __init__(self, db_path):
        self.db_path = db_path
        self.conn = None

    def connect(self):
        self.conn = sqlite3.connect(self.db_path)
        # Create a mock table
        self.conn.execute("CREATE TABLE IF NOT EXISTS config (key TEXT, val TEXT)")
        self.conn.commit()

    def set_config(self, key, val):
        self.conn.execute("INSERT INTO config VALUES (?, ?)", (key, val))
        self.conn.commit()

    def get_config(self, key):
        cursor = self.conn.cursor()
        cursor.execute("SELECT val FROM config WHERE key = ?", (key,))
        row = cursor.fetchone()
        return row[0] if row else None

    def close(self):
        if self.conn:
            self.conn.close()
```

### The Pytest Test Suite:
```python
# Save this file as test_db_helper.py
import pytest
from db_helper import SQLiteDatabaseHelper

# 1. Parameterized fixture specifying two different database paths
# SQLite supports memory databases (':memory:') and disk files
@pytest.fixture(params=[":memory:", "temp_test_db.sqlite"])
def db_client(request, tmp_path):
    """
    Fixture that initializes and yields a database helper client,
    testing both in-memory and disk-based SQLite configurations.
    """
    db_target = request.param
    
    # If the parameter is a disk file, locate it inside pytest's tmp_path directory
    if db_target != ":memory:":
        db_target = str(tmp_path / db_target)
        
    # SETUP: Initialize and connect
    client = SQLiteDatabaseHelper(db_target)
    client.connect()
    
    yield client
    
    # TEARDOWN: Close connection and clean up resources
    client.close()

# 2. Test functions automatically execute twice: once for each parameter in db_client
def test_database_insert_and_retrieve(db_client):
    # Write config
    db_client.set_config("timeout", "30")
    
    # Retrieve config
    value = db_client.get_config("timeout")
    
    assert value == "30"

def test_missing_key_returns_none(db_client):
    value = db_client.get_config("non_existent_key")
    assert value is None
```

---

## Summary

Pytest fixtures provide a clean, modular way to manage test resources. By configuring lifetimes scopes, handling cleanup with `yield`, and leveraging parameterization, you can write thorough test suites that verify your code under different configurations. This completes **Chapter 49 - Testing DevOps Code**. In the next chapter, we will explore **Infrastructure Testing**.

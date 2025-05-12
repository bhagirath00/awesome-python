# PEP 8 Style Guide: Standard Code Conventions in Python

When working in engineering teams, code readability is critical. Code is read far more often than it is written. If every developer uses their own styling conventions—such as mixing tabs and spaces, using inconsistent variable naming styles, or writing long lines—the codebase becomes difficult to maintain.

**PEP 8** is the official style guide for Python code. It provides guidelines for writing clean, consistent, and readable Python code.

In this guide, we will cover the core formatting rules defined in PEP 8, examine imports grouping conventions, analyze naming standards, and compare non-compliant code with PEP 8 compliant examples.

---

## Core Formatting Guidelines

### 1. Indentation
* **Spaces vs. Tabs**: Always use **4 spaces** per indentation level. Never use tabs.
* Mixing tabs and spaces in Python 3 will raise a `TabError` at runtime.

### 2. Maximum Line Length
* Limit all lines to a maximum of **79 characters**.
* For docstrings or comments, limit the line length to **72 characters**.
* Long lines can be wrapped using Python's implicit line joining inside parentheses, brackets, and braces.

```python
# RECOMMENDED: Implicit line wrapping inside parentheses
def configure_network_interface(
        interface_name, ip_address, gateway_address, dns_servers):
    pass
```

### 3. Blank Lines
* Surround top-level function and class definitions with **two blank lines**.
* Surround method definitions inside a class with **a single blank line**.

### 4. Imports Formatting
Imports should always be written at the top of the file, right after any module comments and docstrings, and before module globals. Group imports in this order, with a blank line separating each group:
1. **Standard library** imports.
2. Related **third-party** imports.
3. **Local application/library** specific imports.

```python
# 1. Standard Library
import os
import sys

# 2. Third-Party Libraries
from cryptography.fernet import Fernet
import requests

# 3. Local Application Modules
from my_app import database_client
```

---

## Naming Conventions

Naming conventions vary depending on the code element:

| Code Element | Naming Convention | Example |
| :--- | :--- | :--- |
| **Variables & Functions** | lowercase with words separated by underscores (`snake_case`). | `db_connection`, `get_user_by_id()` |
| **Classes** | Capitalized words without underscores (`PascalCase`). | `DatabaseConnector`, `BillingManager` |
| **Constants** | All capital letters with underscores (`UPPER_CASE_SNAKE`). | `MAX_RETRIES`, `API_ENDPOINT_URL` |
| **Internal Attributes** | Start with a single leading underscore to denote protected scope. | `_internal_token` |

---

## Non-Compliant vs. PEP 8 Compliant Code

Let's look at a badly styled Python script, followed by its PEP 8 compliant equivalent:

### Non-Compliant Implementation:
```python
# Bad styling: mixed imports, poor naming, spacing issues, tabs mixed in
import requests,os
class my_connector:
	def __init__(self,Url):
		self.Url=Url
	def fetch_data(self):
         # Bad: spacing around operator, tabs used
         response=requests.get(self.Url)
         return response.json()
def helper(x, y): return x+y # Bad: single line function, no spaces around operator
```

### PEP 8 Compliant Implementation:
```python
# Clean styling: grouped imports, clear naming, proper spacing, and blank lines
import os

import requests


class MyConnector:
    def __init__(self, url):
        self.url = url

    def fetch_data(self):
        # Spacing around assignment operator
        response = requests.get(self.url)
        return response.json()


def helper(x, y):
    # Spaces around arithmetic operator
    return x + y
```

---

## Summary

Adhering to the PEP 8 style guide ensures that your Python code is consistent and readable. By using standard indentation, limiting line lengths, grouping imports, and following naming conventions, you can write maintainable code. In the next section, we will cover the **Black formatter**.

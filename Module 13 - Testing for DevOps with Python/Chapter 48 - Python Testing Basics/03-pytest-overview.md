# Pytest Overview: A Modern Testing Framework for Python

While Python's built-in `unittest` module is powerful, it inherits Java-style object-oriented syntax. Writing classes, importing modules, and using specific assertion names (like `self.assertAlmostEqual`) can make test code verbose and difficult to maintain.

**`pytest`** is the standard third-party testing framework in the Python community. It allows you to write tests using simple functions, standard Python `assert` statements, and reusable **Fixtures** (which replace rigid class-based setup/teardown methods).

In this guide, we will compare `pytest` and `unittest`, install the package, examine the test discovery rules, and rewrite a test suite using Pytest syntax.

---

## Pytest vs. Unittest Comparison

| Feature / Capability | Python `unittest` (Built-In) | Python `pytest` (Third-Party) |
| :--- | :--- | :--- |
| **Testing Paradigm** | Object-Oriented (requires classes) | Functional (simple functions) |
| **Assertion Style** | Custom methods (`self.assertEqual`) | Standard Python `assert` statements |
| **Setup & Teardown** | Class hooks (`setUp`/`tearDown`) | Reusable dependency-injection **Fixtures** |
| **Parameterization** | Verbose (requires helper libraries) | Native decorator (`@pytest.mark.parametrize`) |
| **Plugin Ecosystem** | Small | Large (over 800 community plugins) |

---

## 1. Installation

Install the package using pip:

```bash
pip install pytest
```

Verify the CLI tool is available:
```bash
pytest --version
```

---

## 2. Test Discovery Rules

When you execute the `pytest` command, the engine automatically searches for files and functions matching these rules:
* Files named **`test_*.py`** or **`*_test.py`**.
* Functions defined outside classes starting with the prefix **`test_`**.
* Classes starting with the prefix **`Test`** (no inheritance required).

---

## 3. Simplified Assertions and Fixtures

Instead of using class-based setups, Pytest uses **Fixtures** (marked with the `@pytest.fixture` decorator) to provide test resources. A fixture can run cleanup code automatically after a test completes by replacing `return` with **`yield`**.

Here is our configuration parser test suite rewritten in Pytest syntax:

```python
# Save this file as test_config_pytest.py
import os
import pytest
from config_parser import ConfigurationParser

# 1. Define a reusable fixture to handle setup and teardown
@pytest.fixture
def temp_config_file():
    """
    Creates a temporary configuration file, yields it to the test,
    and deletes it once the test completes.
    """
    filename = "temp_test_config.conf"
    
    # SETUP: Create file
    with open(filename, "w", encoding="utf-8") as f:
        f.write("# Pytest mock config\n")
        f.write("port = 8080\n")
        f.write("host = 127.0.0.1\n")
        
    # Yield the filepath to the test function
    yield filename
    
    # TEARDOWN: Delete file
    if os.path.exists(filename):
        os.remove(filename)

# 2. Write test functions using standard assert statements
# To use a fixture, pass its name as an argument to the test function

def test_successful_parsing(temp_config_file):
    """Verify that a valid configuration file is parsed correctly."""
    parser = ConfigurationParser(temp_config_file)
    data = parser.parse()
    
    # Standard Python assert statements
    assert data["port"] == "8080"
    assert data["host"] == "127.0.0.1"

def test_missing_file_throws_error():
    """Verify that passing a missing filepath raises FileNotFoundError."""
    parser = ConfigurationParser("missing_file.conf")
    
    # Use pytest.raises to verify exceptions
    with pytest.raises(FileNotFoundError):
        parser.parse()

def test_invalid_syntax_throws_error(temp_config_file):
    """Verify that a config file with invalid syntax raises ValueError."""
    # Modify the temporary file to introduce a syntax error
    with open(temp_config_file, "w", encoding="utf-8") as f:
        f.write("invalid_line_without_equals_sign\n")
        
    parser = ConfigurationParser(temp_config_file)
    with pytest.raises(ValueError):
        parser.parse()
```

---

## 4. Running the Tests

To run all tests in the current directory and its subdirectories, execute:

```bash
pytest
```

To run a specific test file:
```bash
pytest test_config_pytest.py
```

To run tests and display stdout logs (which are captured and hidden by default):
```bash
pytest -s
```

---

## Summary

The `pytest` framework provides a modern syntax for writing Python tests. By utilizing standard `assert` statements and reusable fixtures, you can write cleaner, more maintainable test suites. In the next section, we will cover **test structures**.

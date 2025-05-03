# Test Structures: Directory Layouts and Conftest Configurations

When building test suites for your automation scripts, organizing your files is key to keeping the codebase maintainable. 

If you scatter test files across your source directories, locating code and running specific test suites becomes difficult. A better approach is to use a centralized directory structure, separate unit tests from integration tests, and leverage **`conftest.py`** to share fixtures globally.

In this guide, we will cover the directory layout for Python projects, configure the `pytest.ini` settings file, and write a global `conftest.py` file to share test resources.

---

## Directory Layout for Python Projects

For Python applications and DevOps repositories, the standard layout separates source code from tests:

```text
my-devops-project/
├── pytest.ini            # Global Pytest configuration file
├── src/                  # Application source code
│   ├── config_loader.py
│   └── deploy_tool.py
└── tests/                # Test suites directory
    ├── conftest.py       # Global fixtures configuration file
    ├── integration/      # Integration test files
    │   └── test_deploy_integration.py
    └── unit/             # Unit test files
        ├── test_config_loader.py
        └── test_parsing_logic.py
```

*Note: Separating `/unit` and `/integration` subfolders allows you to run fast unit tests separately from slow network-based integration tests.*

---

## 1. Global Configurations using `pytest.ini`

To define global options for Pytest (such as default command-line flags or custom markers), create a `pytest.ini` configuration file in the project root:

```ini
[pytest]
# Default CLI options:
# -v: verbose output
# --tb=short: compact stack traces on failures
addopts = -v --tb=short

# Define search directories
testpaths = tests

# Register custom test category markers
markers =
    unit: Fast logic unit tests.
    integration: Slow integration tests that connect to endpoints.
```

Using markers allows you to run specific groups of tests:
```bash
# Run unit tests only
pytest -m unit
```

---

## 2. Sharing Fixtures Globally with `conftest.py`

If you write fixtures inside a test file (e.g. `test_config.py`), those fixtures are only available within that file. To share fixtures across multiple test files without importing them, define them in a file named **`conftest.py`** inside your `tests/` directory.

Pytest automatically loads `conftest.py` files it finds in the directory tree.

Here is a `conftest.py` file that defines a shared mock environment fixture:

```python
# Save this file as tests/conftest.py
import os
import pytest

@pytest.fixture(scope="session", autouse=True)
def configure_test_environment():
    """
    Session-scoped fixture that runs once before any tests start.
    Sets up mock environment variables for the test run.
    """
    print("\n[conftest] Setting up global test environment...")
    
    # Save original values to restore later
    original_env = dict(os.environ)
    
    # Inject mock credentials
    os.environ["AWS_ACCESS_KEY_ID"] = "mock_key_id"
    os.environ["AWS_SECRET_ACCESS_KEY"] = "mock_secret_key"
    os.environ["DATABASE_URL"] = "sqlite:///:memory:"
    
    yield
    
    print("\n[conftest] Restoring original environment...")
    os.environ.clear()
    os.environ.update(original_env)

@pytest.fixture
def mock_temp_directory(tmp_path):
    """
    Function-scoped fixture that returns a temporary directory path.
    Uses pytest's built-in 'tmp_path' fixture.
    """
    temp_dir = tmp_path / "mock_configs"
    temp_dir.mkdir()
    return temp_dir
```

---

## 3. Consuming Shared Fixtures in Test Files

Because the fixtures are defined in `conftest.py`, you can use them in any test file under the `tests/` directory by passing their name as an argument to your test functions:

```python
# Save this file as tests/unit/test_config_loader.py

def test_config_retrieves_env_vars(mock_temp_directory):
    # Retrieve dynamic variables
    import os
    db_url = os.environ.get("DATABASE_URL")
    
    # Verify environment values match our conftest settings
    assert db_url == "sqlite:///:memory:"
    
    # Verify temp path is writable
    temp_file = mock_temp_directory / "app.conf"
    temp_file.write_text("port=80")
    assert temp_file.read_text() == "port=80"
```

---

## Summary

Organizing your test suites into structured directories and using configuration files like `pytest.ini` and `conftest.py` makes your test suite easier to maintain. In the next section, we will cover **assertions** in Pytest.

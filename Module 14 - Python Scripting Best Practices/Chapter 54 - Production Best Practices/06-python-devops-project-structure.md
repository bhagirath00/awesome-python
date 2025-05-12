# Python DevOps Project Structure: Production Directory Layouts

As your automation codebase grows, keeping scripts in a flat structure (e.g. putting all `.py` files in the root folder) becomes unmanageable. It results in name collisions, circular import dependencies, and makes testing difficult.

For production-grade DevOps repositories, use a standardized directory layout. 

The industry standard is the **`src/` layout**. By placing all source code inside a `src/` directory, you separate development files from packaging configurations and tests, ensuring Python imports your modules consistently.

In this guide, we will analyze a production directory layout, explain the purpose of each directory, and examine configuration file placement.

---

## Production Directory Layout

Here is a standard project structure for a DevOps repository that contains Python automation code, Terraform modules, and Ansible playbooks:

```text
devops-automation-repo/
├── .github/
│   └── workflows/
│       └── pipeline.yml       # CI/CD pipeline configuration
├── configs/
│   └── default_config.yaml    # Application settings
├── infrastructure/
│   ├── terraform/             # IaC provisioning modules
│   │   ├── main.tf
│   │   └── variables.tf
│   └── ansible/               # Configuration management playbooks
│       ├── playbook.yml
│       └── hosts.ini
├── src/                       # Source code directory
│   └── devops_tools/
│       ├── __init__.py
│       ├── cli.py             # CLI parser (Click / Argparse)
│       ├── s3_helper.py       # S3 interaction module
│       └── utils.py           # Shared helper functions
├── tests/                     # Test suites directory
│   ├── conftest.py
│   ├── integration/
│   │   └── test_s3_integration.py
│   └── unit/
│       └── test_utils.py
├── .coveragerc                # Coverage exclusions config
├── .gitignore                 # Git ignore file list
├── .pre-commit-config.yaml    # Pre-commit hooks config
├── pyproject.toml             # Packaging metadata and CLI commands
├── requirements.txt           # Dependency pin lists with hashes
└── README.md                  # Project documentation
```

---

## Directory Descriptions

| Directory / File | Purpose | Description |
| :--- | :--- | :--- |
| **`.github/workflows/`** | CI/CD | Mapped pipeline actions that trigger tests on every code commit. |
| **`configs/`** | Configurations | Contains static configuration files (YAML, JSON) representing application defaults. |
| **`infrastructure/`** | Infrastructure | Stores Terraform and Ansible files to manage resources. |
| **`src/devops_tools/`** | Source Code | Contains the main Python modules. Keeping code inside `src/` prevents Python from importing local development files during test runs. |
| **`tests/`** | Testing | Holds unit and integration test files, separated to allow running tests selectively. |
| **`pyproject.toml`** | Packaging | Stores packaging metadata, dependencies, and command line entry points. |

---

## The Value of the `src/` Layout

A common issue in Python projects is that when you run tests, the interpreter imports the local source files in your working directory instead of the installed package:

```text
# Without 'src/' layout:
Importing 'my_module' loads the local file, bypassing package validation.

# With 'src/' layout:
Importing 'my_module' fails unless the package has been installed,
ensuring you are testing the built package.
```

Using the `src/` layout forces you to install the package (e.g. `pip install -e .`) before running tests, ensuring that your test runs align with production executions.

---

## Summary

Adopting a structured directory layout like the `src/` layout is essential for keeping your scripting repositories clean and maintainable. By separating configurations, source code, and tests, you can build reliable automation pipelines. In the next section, we will cover the **production checklist**.

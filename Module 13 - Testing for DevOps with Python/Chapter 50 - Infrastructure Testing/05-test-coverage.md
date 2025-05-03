# Test Coverage with Python: Line, Branch, and Pytest-Cov Integration

When building a test suite, you need to know how much of your code is actually executed by your tests. **Test Coverage** measures the percentage of your source code lines that are run during a test execution.

Measuring coverage helps you identify untested execution paths, such as error-handling blocks or fallback procedures, which are common sources of production bugs.

In this guide, we will cover the difference between line and branch coverage, configure the `pytest-cov` library, set up exclusions in a `.coveragerc` file, and analyze test coverage reports.

---

## Line vs. Branch Coverage

Consider this Python function:

```python
def process_data(status):
    if status == "ACTIVE":
        run_active_process()
    else:
        run_fallback_process()
```

If your tests only verify the "ACTIVE" status:
* **Line Coverage**: Can report high coverage (e.g., 80%) because most lines were executed.
* **Branch Coverage**: Reports only 50% because the `else` branch was never executed.

> [!TIP]
> Always enable **Branch Coverage** in your testing configuration to ensure all conditional paths and logical branches are verified.

---

## 1. Installation

Install the coverage library and the Pytest integration plugin:

```bash
pip install coverage pytest-cov
```

Verify that the coverage command is available:
```bash
coverage --version
```

---

## 2. Using Pytest-Cov in the CLI

To run your tests and output a coverage report to the console, pass the `--cov` flag followed by the source directory path:

```bash
pytest --cov=src tests/
```

To include branch coverage analysis, add the `--cov-branch` flag:
```bash
pytest --cov=src --cov-branch tests/
```

This command outputs a coverage table:
```text
Name                    Stmts   Miss Branch BrPart  Cover
---------------------------------------------------------
src/config_loader.py       24      2      4      1    89%
src/deploy_tool.py         45      8     10      2    81%
---------------------------------------------------------
TOTAL                      69     10     14      3    83%
```

---

## 3. Excluding Code blocks using `.coveragerc`

DevOps scripts often contain code blocks that do not need to be tested, such as manual CLI entry points (`if __name__ == "__main__":`) or log statement blocks. 

To exclude these blocks from your coverage metrics, create a **`.coveragerc`** configuration file in your project root:

```ini
[run]
branch = True
source = src

[report]
# Regex patterns to exclude specific code lines from metrics
exclude_lines =
    # Exclude standard main block
    if __name__ == .__main__.:
    
    # Exclude debug logs
    logger\.debug
    
    # Exclude abstract method definitions
    raise NotImplementedError
    
    # Exclude exception safeguards
    pass

# Ignore files completely (e.g. mock scripts)
omit =
    src/mocks/*
    tests/*
```

---

## 4. Generating HTML Coverage Dashboards

Reviewing coverage details in a terminal can be difficult for large projects. You can generate an interactive HTML coverage report using the `--cov-report=html` flag:

```bash
pytest --cov=src --cov-report=html tests/
```

This command creates a `htmlcov/` directory containing a web dashboard. Open `htmlcov/index.html` in a browser to view a color-coded breakdown showing exactly which lines of your code were executed (green) and which were missed (red).

---

## Summary

Measuring test coverage helps you identify untested execution paths in your automation code. By using `pytest-cov` to track branch coverage and configuring exclusions in a `.coveragerc` file, you can build reliable test suites. This completes **Module 13: Testing for DevOps with Python**. In the next module, we will explore **Python Scripting Best Practices**.

# Pylint: Comprehensive Static Code Analysis

While Flake8 provides fast linting check rules, it does not perform deep semantic code analysis. For large-scale codebases, you need a tool that can check for code duplication, identify overly complex code blocks, verify naming conventions, check docstrings, and grade your code.

**Pylint** is a comprehensive static code analyzer for Python. It checks your code files against strict compliance rules and provides a final code quality score out of 10.

In this guide, we will compare Pylint and Flake8, examine Pylint's warning categories, configure settings in a `.pylintrc` file, and write a script containing code that needs refactoring to analyze Pylint's output.

---

## Pylint vs. Flake8

| Comparison Metric | Flake8 Linter | Pylint Static Analyzer |
| :--- | :--- | :--- |
| **Analysis Depth** | Fast, syntax and basic style checks. | Deep semantic analysis, type inferences, dependency maps. |
| **Code Grading** | No grading. | Assigns a numeric quality score (e.g., `8.5/10`). |
| **Refactoring Checks** | Limited McCabe checks. | Flags duplicate code blocks, unused parameters, excessive arguments. |
| **Docstrings Checks** | None. | Verifies all modules, classes, and public functions have docstrings. |
| **Speed** | Fast. | Slow (performs deep code graph traversal). |

---

## Pylint Warning Categories

Pylint categorizes its findings using classification letters:

* **`C` (Convention)**: Style violations (e.g. invalid naming styles, missing docstrings).
* **`R` (Refactor)**: Code smells that require refactoring (e.g. duplicate blocks, too many parameters in a function).
* **`W` (Warning)**: Code bugs like unused variables, shadowing parameters, or deprecated methods.
* **`E` (Error)**: Syntax errors or invalid calls that will raise exceptions at runtime.

---

## 1. Installation

Install Pylint using pip:

```bash
pip install pylint
```

Verify the CLI tool:
```bash
pylint --version
```

---

## 2. Running Pylint via the CLI

To scan a file:
```bash
pylint app.py
```

### The Output Report
Pylint outputs a list of issues followed by a summary report and code grade:

```text
************* Module app
app.py:1:0: C0114: Missing module docstring (missing-module-docstring)
app.py:5:4: R0913: Too many arguments in function (6/5) (too-many-arguments)

------------------------------------------------------------------
Your code has been rated at 7.50/10 (previous run: 7.50/10, +0.00)
```

---

## 3. Configuring Settings in `.pylintrc`

Pylint can be strict by default. You can configure and disable specific warning codes in a `.pylintrc` file.

To generate a default configuration file, run:
```bash
pylint --generate-rcfile > .pylintrc
```

Here is a `.pylintrc` configuration template:

```ini
[MASTER]
# Exclude specific folders
ignore=venv,tests

[MESSAGES CONTROL]
# Disable specific warning codes
# C0111: missing-docstring
# C0103: invalid-name
disable=C0111, C0103

[REPORTS]
# Show the numeric grade rating
reports=yes

[BASIC]
# Maximum length of lines
max-line-length=88
```

---

## 4. Code Refactoring Example

Let's write a script containing code smells to analyze Pylint's output:

### Code with Refactoring Issues (`refactor_demo.py`):
```python
def configure_user_account(user_id, email, password, role, region, billing_code):
    """
    Function containing too many arguments (R0913).
    """
    total_connections = 10
    
    # Unused variable (W0612)
    unused_var = 5
    
    # Shadowing variable from outer scope (W0621)
    for total_connections in range(3):
        print(total_connections)
        
    return True
```

### Pylint Output:
```text
refactor_demo.py:1:0: R0913: Too many arguments (6/5) (too-many-arguments)
refactor_demo.py:7:4: W0612: Unused variable 'unused_var' (unused-variable)
refactor_demo.py:10:8: W0621: Redefining name 'total_connections' from outer scope (line 5) (redefined-outer-name)
```

---

## Summary

Pylint performs deep static code analysis to help you identify code smells and logical bugs. By configuring exclusions and rules overrides in a `.pylintrc` file, you can enforce code quality checks in your project. In the next section, we will cover **type hints**.

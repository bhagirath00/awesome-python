# Working with INI Files

Before modern formats like JSON, YAML, and TOML became popular, the standard configuration format on Windows and many Linux utilities was the INI file. INI is short for **initialization**. 

Even though it is an older format, you will still run into INI configurations every day in DevOps. Tools like Git (`.gitconfig`), Ansible (`ansible.cfg`), and pip (`pip.conf`) use INI or INI-style syntax for their configuration.

In this guide, we will learn about the structure of INI files, their rules, and how they compare to newer alternatives.

---

## Structure of an INI File

An INI file is a plain-text file structured into sections, properties, and values. Here is what a simple config looks like:

```ini
; This is a comment using a semicolon
# This is also a comment using a hash symbol

[webserver]
host = 192.168.1.10
port = 80
enable_ssl = yes

[database]
host = 192.168.1.20
port = 3306
```

---

## Core Elements of INI Files

### 1. Sections
Sections are headers enclosed in square brackets: `[section_name]`. They group key-value pairs together. A section continues until the next section header begins or the file ends.

### 2. Keys and Values
Keys and values are separated by an equals sign (`=`) or a colon (`:`). White space around the separator is ignored.

```ini
key = value
another_key : another_value
```

### 3. Comments
Comments start with a semicolon (`;`) or a hash mark (`#`) at the beginning of a line. Some systems allow inline comments, but this depends on the specific parser tool you are using.

---

## Comparing INI with TOML and YAML

Although INI looks similar to TOML, it lacks many features that make modern configuration languages reliable.

| Feature | INI | TOML | YAML |
| :--- | :--- | :--- | :--- |
| **Data Types** | None (everything is treated as a string) | Strong (numbers, booleans, dates) | Strong (numbers, booleans, lists) |
| **Nesting** | Flat (one level of sections only) | Nested tables (dot notation) | Infinite nesting via indentation |
| **Arrays / Lists** | No direct syntax (must parse commas manually) | Yes (using brackets) | Yes (using dashes) |
| **Standardization** | No official spec (parsers behave differently) | Strict official specification | Strict official specification |

---

## How INI Files Parse

Because INI does not have strict data types, the parser reads all values as strings. Here is how a Python configuration parser handles an INI section:

```mermaid
graph TD
    A[INI File] --> B[Parser reads Section: database]
    B --> C[Reads port = 3306]
    C --> D["Stores database['port'] = '3306' as String"]
    D --> E[Developer must manually run int('3306')]
```

---

## Real-World Example: Ansible Configuration File

Here is an example of an `ansible.cfg` file which controls how Ansible runs playbooks on your remote hosts:

```ini
# ansible.cfg - Settings for remote infrastructure management

[defaults]
inventory = ./hosts
remote_user = deployer
ask_pass = false
roles_path = /etc/ansible/roles

[privilege_escalation]
become = true
become_method = sudo
become_user = root
become_ask_pass = false

[ssh_connection]
pipelining = true
timeout = 30
```

---

## Common Mistakes

Here are some errors to watch out for when creating or editing INI files:

### 1. Expecting Automatic Type Conversion
Unlike JSON or TOML, if you write `port = 8080` or `debug = false`, your parser will read them as the string `"8080"` and the string `"false"`. If you write `if config['debug']:`, it will evaluate to `True` because non-empty strings are always truthy in Python. You must convert these manually.

### 2. Creating Nested Sections
INI files do not support nested sections natively. Attempting to do the following will lead to mixed results depending on the parser:

```ini
# INVALID INI NESTING
[server]
host = localhost

[server.database] # The parser will treat this as a flat section name string 'server.database'
port = 5432
```

### 3. Duplicate Key Names within a Section
If you specify the same key twice in the same section, most parser engines will simply overwrite the first value with the second one without warning you.

```ini
[web]
port = 80
port = 8080 # Overwrites port 80 in most parsers
```

---

## Summary

* INI files are a classic plain-text configuration format used by tools like Git and Ansible.
* They are organized into sections defined by brackets (e.g. `[defaults]`).
* Key-value pairs are defined using either `=` or `:` as separators.
* Comments start with a semicolon (`;`) or a hash symbol (`#`).
* INI does not have native data types; every value is parsed as a string.
* You must manually convert values to integers or booleans inside your code.
* Nesting sections is not supported; section layouts are strictly flat.
* Duplicate keys inside a section will overwrite earlier values without warnings.
* Always check tool documentation, as many tools use slightly custom INI parser engines.

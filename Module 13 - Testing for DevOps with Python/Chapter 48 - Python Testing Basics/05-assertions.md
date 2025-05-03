# Assertions in Pytest: Introspection and Exception Verification

Assertions are the core of any test. They verify that the actual output of your code matches the expected result. 

While Java-style frameworks use custom assertion methods (e.g. `self.assertListEqual`), Pytest uses the standard Python **`assert`** statement. Pytest intercepts these assertions at runtime and provides **Rich Introspection**—if an assertion fails, Pytest prints a detailed diff showing exactly where strings, lists, or dictionaries differ.

In this guide, we will cover how Pytest handles assertions, use helper functions like `pytest.approx()` for floating-point calculations, assert exceptions using `pytest.raises`, and write a test suite for a validation utility.

---

## Pytest Assert Introspection Example

If you assert list equality in Pytest:
```python
assert [1, 2, 3] == [1, 2, 4]
```
Pytest does not simply print "AssertionError". It outputs a detailed comparison:
```text
At index 2 diff: 3 != 4
Use -v to get the full diff
```

This introspection applies to large dictionaries, multi-line strings, and custom objects as well.

---

## 1. Comparing Floats with `pytest.approx`

Because of binary floating-point representation limitations, calculations in computers can yield small rounding discrepancies:

```python
# Standard Python evaluation fails: 0.1 + 0.2 evaluates to 0.30000000000000004
assert 0.1 + 0.2 == 0.3 # Fails with AssertionError!
```

To prevent tests from failing due to minor rounding errors, wrap the expected value in **`pytest.approx()`**:

```python
import pytest

# Passes successfully
assert 0.1 + 0.2 == pytest.approx(0.3)
```

---

## 2. Asserting Exceptions and Error Messages

To verify that your code raises the correct exceptions under error conditions, use the **`pytest.raises`** context manager. You can capture the exception details to verify the error message content.

```python
import pytest

def divide_numbers(a, b):
    if b == 0:
        raise ZeroDivisionError("Cannot divide by zero!")
    return a / b

def test_division_exception():
    # Verify exception is raised
    with pytest.raises(ZeroDivisionError) as exc_info:
        divide_numbers(10, 0)
        
    # Verify the specific error message text
    assert "divide by zero" in str(exc_info.value)
```

---

## 3. Practical Example: Testing an Input Validator

Here is an input validation utility, followed by a Pytest test suite that uses assertions to verify its logic:

```python
# Save this file as validator.py
import re

class UserInputValidator:
    def __init__(self):
        self.email_regex = re.compile(r"^[a-zA-Z0-9_.+-]+@[a-zA-Z0-9-]+\.[a-zA-Z0-9-.]+$")

    def validate_user_details(self, details_dict):
        """
        Validates email structure, password strength, and age limits.
        """
        errors = []
        
        # 1. Validate email
        email = details_dict.get("email", "")
        if not self.email_regex.match(email):
            errors.append("Invalid email address format.")
            
        # 2. Validate password strength
        password = details_dict.get("password", "")
        if len(password) < 8:
            errors.append("Password must be at least 8 characters long.")
        if not any(char.isdigit() for char in password):
            errors.append("Password must contain at least one digit.")
            
        # 3. Validate age limits
        try:
            age = int(details_dict.get("age", 0))
            if age < 18 or age > 120:
                errors.append("Age must be between 18 and 120.")
        except (ValueError, TypeError):
            errors.append("Age must be a valid integer.")

        if errors:
            raise ValueError("; ".join(errors))
            
        return True
```

### The Pytest Test Suite:
```python
# Save this file as test_validator.py
import pytest
from validator import UserInputValidator

@pytest.fixture
def validator():
    return UserInputValidator()

def test_successful_validation(validator):
    valid_data = {
        "email": "user@example.com",
        "password": "SecurePassword1",
        "age": 25
    }
    # Should return True without raising exceptions
    assert validator.validate_user_details(valid_data) is True

def test_invalid_email_format(validator):
    bad_data = {
        "email": "bad_email_format",
        "password": "SecurePassword1",
        "age": 25
    }
    with pytest.raises(ValueError) as exc_info:
        validator.validate_user_details(bad_data)
        
    assert "Invalid email address format." in str(exc_info.value)

def test_weak_password(validator):
    bad_data = {
        "email": "user@example.com",
        "password": "short", # Too short, no digits
        "age": 25
    }
    with pytest.raises(ValueError) as exc_info:
        validator.validate_user_details(bad_data)
        
    error_msg = str(exc_info.value)
    assert "Password must be at least 8 characters long." in error_msg
    assert "Password must contain at least one digit." in error_msg

def test_invalid_age(validator):
    bad_data = {
        "email": "user@example.com",
        "password": "SecurePassword1",
        "age": 15 # Too young
    }
    with pytest.raises(ValueError) as exc_info:
        validator.validate_user_details(bad_data)
        
    assert "Age must be between 18 and 120." in str(exc_info.value)
```

---

## Summary

Pytest simplifies testing by using standard Python `assert` statements and providing detailed feedback on failures. By using `pytest.approx()` for floating-point values and `pytest.raises` to verify exceptions, you can build reliable test suites for your code. This completes **Chapter 48 - Python Testing Basics**. In the next chapter, we will explore **Testing DevOps Code**.

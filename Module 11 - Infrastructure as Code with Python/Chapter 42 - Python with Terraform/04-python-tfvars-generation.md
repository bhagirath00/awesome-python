# Python tfvars Generation: Dynamic Variables and JSON Configurations

To customize infrastructure deployments, Terraform configurations use input variables (`variables.tf`). In manual workflows, developers supply these values using command-line arguments (e.g. `-var="instance_type=t3.micro"`) or by writing variable definition files (`terraform.tfvars`).

In automated pipelines, variable values are resolved at runtime (e.g., retrieving database passwords from a secret vault, fetching developer emails from git details, or scaling cluster sizes based on current load). 

While you can write standard HCL (HashiCorp Configuration Language) files, writing them programmatically requires custom formatters. A better pattern is to use **JSON Variable Files (`.tfvars.json`)**. Terraform natively supports JSON variable files, allowing you to use Python's built-in `json` module to serialize dictionaries into structured variable inputs.

In this guide, we will compare HCL vs. JSON variable structures, cover secrets security, and write an automation script that generates dynamic variable files and cleans them up post-deploy.

---

## HCL vs. JSON Variables

### HCL Variable File (`terraform.tfvars`)
```hcl
environment   = "production"
instance_count = 5
billing_tags  = {
  Department = "Engineering"
  Project    = "Billing API"
}
```
*Note: Writing this programmatically requires string concatenation or a custom template library (like Jinja2).*

### JSON Variable File (`terraform.tfvars.json`)
```json
{
  "environment": "production",
  "instance_count": 5,
  "billing_tags": {
    "Department": "Engineering",
    "Project": "Billing API"
  }
}
```
*Note: Writing this is trivial in Python: `json.dump(dictionary, file)`.*

---

## 1. Security Safeguards for Generated Variables

When generating variable files programmatically, you are often writing sensitive details (like database passwords or API keys) to the server disk. To prevent leaking these secrets:

1. **Add to `.gitignore`**: Make sure `*.tfvars.json` and `*.tfvars` are explicitly ignored in your repository's `.gitignore` file.
2. **Use Temp Files**: Create temporary variable files, and delete them immediately using a `finally` block once the Terraform command completes.
3. **Avoid Console Prints**: Never log the contents of your generated variable files to standard output consoles.

---

## 2. Dynamic Variable Generator Implementation

Here is a Python script that loads credentials, compiles configuration details, generates a `terraform.tfvars.json` file, and executes a Terraform apply command safely.

```python
import os
import json
import subprocess

class TerraformVariableGenerator:
    def __init__(self, tf_dir):
        self.tf_dir = tf_dir
        self.tfvars_path = os.path.join(tf_dir, "terraform.tfvars.json")

    def create_variable_file(self, variables_dict):
        """
        Serializes a Python dictionary into a JSON tfvars file.
        """
        try:
            print("Generating dynamic variables file...")
            with open(self.tfvars_path, "w", encoding="utf-8") as f:
                json.dump(variables_dict, f, indent=4)
            print(f"✅ Variables file written to: {self.tfvars_path}")
            return True
        except Exception as e:
            print(f"❌ Failed to write variables file: {e}")
            return False

    def clean_variable_file(self):
        """
        Deletes the generated variables file to prevent credential leaks.
        """
        if os.path.exists(self.tfvars_path):
            try:
                os.remove(self.tfvars_path)
                print("🧹 Temporary variables file deleted successfully.")
            except Exception as e:
                print(f"Warning: Failed to clean up variables file: {e}")

    def execute_apply(self):
        """
        Runs terraform apply. It will automatically load 'terraform.tfvars.json' if present.
        """
        try:
            print("Applying configuration using generated variable file...")
            result = subprocess.run(
                ["terraform", "apply", "-auto-approve"],
                cwd=self.tf_dir,
                capture_output=True,
                text=True,
                check=True
            )
            print("✅ Terraform apply completed successfully.")
            return True
        except subprocess.CalledProcessError as e:
            print(f"❌ Terraform apply failed:\n{e.stderr}")
            return False
```

---

## 3. Orchestration Script: Injecting Secret Vault Details

Here is the integration wrapper showing how you load dynamic database secrets, build tags, generate the JSON variables, run the deploy, and clean up the environment:

```python
# Save this file as scripts/run_dynamic_deploy.py
# (Requires the TerraformVariableGenerator class defined above)

def run_dynamic_deploy():
    tf_directory = "./infrastructure"
    generator = TerraformVariableGenerator(tf_directory)
    
    # 1. Retrieve dynamic credentials
    # In production, load these from AWS Secrets Manager or Vault
    db_pass = os.environ.get("PRODUCTION_DB_PASSWORD", "fallback_pass")
    
    # 2. Build the variable dictionary
    variables = {
        "environment": "production",
        "instance_type": "t3.medium",
        "database_password": db_pass,
        "resource_tags": {
            "Owner": "devops-team",
            "CostCenter": "CC-401",
            "AutoDestroy": "false"
        }
    }
    
    # 3. Execute deploy with cleanup safeguards
    try:
        # Create the variables file
        if generator.create_variable_file(variables):
            # Run apply
            generator.execute_apply()
    finally:
        # Guarantee cleanup runs even if apply throws an exception
        generator.clean_variable_file()

# if __name__ == "__main__":
#     run_dynamic_deploy()
```

---

## Summary

Generating variable files programmatically allows you to customize Terraform infrastructure deployments dynamically. By utilizing `.tfvars.json` structures, you can use Python dictionaries to format values and clean them up immediately after execution to prevent credential leaks. This completes **Chapter 42 - Python with Terraform**. In the next chapter, we will explore **Python with Ansible**.

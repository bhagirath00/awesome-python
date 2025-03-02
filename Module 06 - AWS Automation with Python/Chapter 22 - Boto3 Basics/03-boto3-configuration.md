# Boto3 Credentials and Configuration

Before your Python scripts can create or manage resources on AWS, they must authenticate. AWS uses access keys (Access Key ID and Secret Access Key) to verify the identity of your scripts and authorize their requests.

Boto3 has a built-in search sequence to locate these credentials automatically, allowing you to configure authentication once on your system without modifying your scripts.

In this guide, we will look at Boto3's credentials resolution order, how configuration files are structured, and how to query your credentials in Python.

---

## Credentials Resolution Order

When you initialize a Boto3 session, the SDK searches for credentials in a specific sequence. It stops searching as soon as it finds valid keys.

```mermaid
graph TD
    A[Start Boto3 Session] --> B{1. Keys passed in constructor?}
    B -- Yes --> C[Use constructor keys]
    B -- No --> D{2. Environment Variables set?}
    D -- Yes --> E[Use AWS_ACCESS_KEY_ID & AWS_SECRET_ACCESS_KEY]
    D -- No --> F{3. Shared Credentials File exists?}
    F -- Yes --> G[Use ~/.aws/credentials values]
    F -- No --> H{4. Shared Config File exists?}
    H -- Yes --> I[Use ~/.aws/config values]
    H -- No --> J{5. IAM Instance Profile Role?}
    J -- Yes --> K[Query IMDS metadata service]
    J -- No --> L[Raise NoCredentialsError]
```

---

## Configuration Methods Comparison

Let us compare the two most common ways to provide credentials to Boto3 on your host systems:

| Method | Source | Lifecycle | Best For | Description |
| :--- | :--- | :--- | :--- | :--- |
| **Environment Variables** | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | Temporary (lives in shell session). | Container systems (Docker), CI/CD runner pipelines. | Passed directly to the running process in RAM. Safe and easy to override. |
| **Credentials File** | `~/.aws/credentials` | Persistent (stored on host disk). | Local developer laptops, administrative workstations. | Managed via `aws configure` CLI command. Supports multiple profiles. |

---

## File Structures for Credentials and Config

On developer laptops, the AWS CLI and Boto3 share two files located in the `~/.ssh` cousin folder, `.aws/`.

### 1. The Credentials File (`~/.aws/credentials`)
This file holds your secret access keys. You can define multiple profiles:

```ini
[default]
aws_access_key_id = AKIAIOSFODNN7EXAMPLE
aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[staging]
aws_access_key_id = AKIAI44QHXI73EXAMPLE
aws_secret_access_key = je7MtGbClwBF/2ZpXoX7EXAMPLEKEY
```

### 2. The Config File (`~/.aws/config`)
This file holds configuration settings (like default regions and output styles):

```ini
[default]
region = us-east-1
output = json

[profile staging]
region = us-west-2
output = text
```

---

## Querying Active Configuration Details

Let us write a script that queries Boto3 to inspect which profile, region, and credentials it has loaded.

```python
import boto3

# 1. Create a session using a specific profile (e.g. 'default')
# If you run in production, Boto3 will use default or env profiles automatically
try:
    session = boto3.Session(profile_name="default")
    
    # 2. Get credentials object
    credentials = session.get_credentials()
    
    print("Resolved AWS Configuration:")
    print("---------------------------")
    print(f"Active Profile: {session.profile_name}")
    print(f"Target Region:  {session.region_name}")
    
    if credentials:
        # Resolve values (which might have come from files or env variables)
        access_key = credentials.access_key
        print(f"Access Key ID:  {access_key[:8]}... (Resolved via: {credentials.method})")
    else:
        print("No active credentials loaded.")
        
except Exception as err:
    print(f"Configuration query failed: {err}")
```

### Expected Output

```text
Resolved AWS Configuration:
---------------------------
Active Profile: default
Target Region:  us-east-1
Access Key ID:  AKIAIOSF... (Resolved via: shared-credentials-file)
```

---

## Common Mistakes

Here are some typical gotchas when configuring Boto3:

### 1. Hardcoding Keys in the `boto3.client()` constructor
Avoid passing access keys directly into your client initialization:
`boto3.client('s3', aws_access_key_id=key)`
Doing this bypasses Boto3's credentials resolution order, making it impossible to rotate keys without updating your code. It also increases the risk of checking credentials into Git. Always configure keys using environment variables or credentials files.

### 2. Mismatching Profile Names
If your script specifies `boto3.Session(profile_name='staging')`, but your `~/.aws/credentials` file only contains a `[default]` block, the script will crash with a `ProfileNotFound` exception. Always confirm profile names in your host files.

### 3. Placing `profile` prefix in the credentials file
In the config file (`~/.aws/config`), custom profiles require the `profile` prefix (e.g. `[profile staging]`). However, in the credentials file (`~/.aws/credentials`), they must not include the prefix (e.g. `[staging]`). If you add `profile` in the credentials file, Boto3 will fail to load the keys for that profile.

---

## Summary

* Boto3 requires AWS access keys to authenticate to AWS APIs.
* Understand Boto3's credentials resolution order (Constructor -> Env -> Credentials File -> Instance Role).
* Storing credentials in `~/.aws/credentials` is the standard for local development.
* Manage regional settings and format preferences inside `~/.aws/config`.
* Retrieve current session parameters by calling `session.get_credentials()`.
* Access specific credentials properties like `credentials.access_key`.
* Load specific configurations by passing `profile_name` to the Session constructor.
* Avoid passing raw keys to `boto3.client()` constructors to maintain security.
* Remember that custom profiles in the config file require the `profile` prefix, while the credentials file does not.

# Access Key Rotation

AWS IAM Access Keys (the Access Key ID and Secret Access Key) are long-term credentials. Unlike passwords, they do not expire automatically. If a developer accidentally uploads an active access key to a public Git repository, an attacker can find it in minutes and use it to compromise your account.

Rotating these keys regularly really matters. Security best practices (and compliance standards) say you should rotate access keys at least every 90 days. Doing this manually for every developer is exhausting and error-prone. Writing a script to audit, disable, and clean up old access keys makes it automatic and safe.

In this guide, we will write Python scripts using Boto3 to audit access key age and safely rotate them.

---

## Access Key States Comparison

An IAM user can have a maximum of **two** active access keys at any given time. Let us look at the states an access key can be in:

| Key State | Can call APIs? | Limit Per User | Best For | Risk Level |
| :--- | :--- | :--- | :--- | :--- |
| **Active** | Yes | Max 2 | Active applications and CLI usage | High (active credentials) |
| **Inactive** | No | Max 2 (shared with Active) | Testing if key is still in use before deletion | Low (cannot be used unless reactivated) |
| **Deleted** | No | None (purged from AWS) | Cleaning up obsolete credentials | Zero (no longer exists) |

---

## Safe Rotation Workflow

Never delete an old key immediately! If you delete a key that an application is currently using, that application will instantly crash. You must use a phased workflow:

```mermaid
graph TD
    A[Start: User has Key A active] --> B[Create Key B]
    B --> C[Deploy Key B to application/servers]
    C --> D[Verify application works with Key B]
    D --> E[Change Key A state to Inactive]
    E --> F[Monitor logs to ensure no systems use Key A]
    F --> G{Errors or calls on Key A?}
    G -->|Yes| H[Reactivate Key A & troubleshoot]
    H --> E
    G -->|No| I[Delete Key A]
    I --> J[End: Rotation complete]
```

---

## Auditing Access Key Age

Before rotating keys, we need a script to find out which keys are old. Let us write a script that checks all keys for a user and calculates their age in days.

```python
import boto3
from datetime import datetime, timezone
from botocore.exceptions import ClientError

def audit_user_keys(username):
    iam_client = boto3.client('iam')
    
    try:
        print(f"Auditing access keys for user: {username}")
        response = iam_client.list_access_keys(UserName=username)
        
        metadata_list = response['AccessKeyMetadata']
        now = datetime.now(timezone.utc)
        
        for key in metadata_list:
            key_id = key['AccessKeyId']
            status = key['Status']
            create_date = key['CreateDate']
            
            # Calculate the age in days
            age_days = (now - create_date).days
            
            print(f"Key ID: {key_id}")
            print(f"  Status:   {status}")
            print(f"  Age:      {age_days} days old")
            
            # Warning threshold: 90 days
            if age_days > 90 and status == 'Active':
                print("  WARNING: This key is older than 90 days and should be rotated!")
            print("-" * 40)
            
    except ClientError as e:
        print(f"Audit failed: {e}")

# Run the auditor on a sample user
audit_user_keys('junior-dev-alice')
```

### Expected Output
```text
Auditing access keys for user: junior-dev-alice
Key ID: AKIAIOSFODNN7EXAMPLE
  Status:   Active
  Age:      124 days old
  WARNING: This key is older than 90 days and should be rotated!
----------------------------------------
Key ID: AKIAIOLHP7NN7EXAMPLE
  Status:   Inactive
  Age:      10 days old
----------------------------------------
```

---

## Real-World Script: Automated Key Rotation Helper

Let us build a complete rotation workflow helper. This script will:
1. Inspect active keys.
2. If there is only one active key, generate a second key.
3. Show how to disable (deactivate) an old key.
4. Show how to delete a disabled key.

```python
import boto3
from botocore.exceptions import ClientError

def rotate_user_keys(username, old_key_id):
    iam_client = boto3.client('iam')
    
    # 1. Fetch current keys to ensure we do not exceed the 2-key limit
    try:
        keys_response = iam_client.list_access_keys(UserName=username)
        existing_keys = keys_response['AccessKeyMetadata']
        print(f"User '{username}' currently has {len(existing_keys)} keys.")
        
        if len(existing_keys) >= 2:
            print("CRITICAL: User already has 2 keys. You must delete one before creating a new one.")
            return False
    except ClientError as e:
        print(f"Failed listing keys: {e}")
        return False

    # 2. Create the new key
    new_key_id = None
    try:
        print("Creating new access key...")
        create_response = iam_client.create_access_key(UserName=username)
        new_key = create_response['AccessKey']
        new_key_id = new_key['AccessKeyId']
        
        print("New key created successfully!")
        print(f"New Key ID:     {new_key_id}")
        print(f"New Secret Key: {new_key['SecretAccessKey']}")
        print("ACTION REQUIRED: Update your configuration file with these new keys.")
    except ClientError as e:
        print(f"Failed to create new key: {e}")
        return False

    # 3. Deactivate the old key
    try:
        print(f"Deactivating old key '{old_key_id}'...")
        iam_client.update_access_key(
            UserName=username,
            AccessKeyId=old_key_id,
            Status='Inactive'
        )
        print(f"Key '{old_key_id}' is now Inactive. Applications cannot use it.")
    except ClientError as e:
        print(f"Failed to deactivate old key: {e}")
        # Note: If deactivation fails, you should investigate before deleting!

    return True

# Example execution
# Let us rotate the old key we found during our audit
target_user = 'junior-dev-alice'
old_key = 'AKIAIOSFODNN7EXAMPLE'
rotate_user_keys(target_user, old_key)
```

### Expected Output
```text
User 'junior-dev-alice' currently has 1 keys.
Creating new access key...
New key created successfully!
New Key ID:     AKIAIOLHP7NN7EXAMPLE
New Secret Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
ACTION REQUIRED: Update your configuration file with these new keys.
Deactivating old key 'AKIAIOSFODNN7EXAMPLE'...
Key 'AKIAIOSFODNN7EXAMPLE' is now Inactive. Applications cannot use it.
```

---

## Deleting Obsolete Deactivated Keys

Once you have verified that the new key is fully deployed and the old inactive key is no longer receiving any traffic, you must delete it to clean up the user profile.

Let us write a helper script that finds any `Inactive` keys for a user and purges them:

```python
import boto3
from botocore.exceptions import ClientError

def purge_inactive_keys(username):
    iam_client = boto3.client('iam')
    
    try:
        print(f"Checking for inactive keys to delete for user: {username}")
        response = iam_client.list_access_keys(UserName=username)
        metadata_list = response['AccessKeyMetadata']
        
        purged_count = 0
        for key in metadata_list:
            key_id = key['AccessKeyId']
            status = key['Status']
            
            if status == 'Inactive':
                print(f"Found inactive key: {key_id}. Deleting...")
                iam_client.delete_access_key(
                    UserName=username,
                    AccessKeyId=key_id
                )
                print("Key successfully deleted.")
                purged_count += 1
                
        print(f"Purge process complete. Removed {purged_count} key(s).")
        return True
        
    except ClientError as e:
        print(f"Failed to purge keys: {e}")
        return False

# Run the purging operation
purge_inactive_keys('junior-dev-alice')
```

### Expected Output
```text
Checking for inactive keys to delete for user: junior-dev-alice
Found inactive key: AKIAIOSFODNN7EXAMPLE. Deleting...
Key successfully deleted.
Purge process complete. Removed 1 key(s).
```

---

## Line-by-Line Code Breakdown

Let us look at how the Boto3 APIs interact in our rotation script:

1. `iam_client.list_access_keys(UserName=username)`: Returns metadata about all keys assigned to the user. It contains the key IDs, status, and creation dates, but does *not* contain the secret key.
2. `iam_client.create_access_key(...)`: Generates a new access key pair. This is the only API call that returns the secret key.
3. `iam_client.update_access_key(..., Status='Inactive')`: Toggles the state of a key. Making a key `Inactive` is a safe midway step; if an application breaks, you can instantly turn it back to `Active` using the same command.
4. `iam_client.delete_access_key(..., AccessKeyId=...)`: Permanently removes the key from the AWS account database. This operation is irreversible.

---

## Common Mistakes

Here are some pitfalls when automating access key rotation:

### 1. Hard-deleting Keys Immediately
Never delete the old key immediately after creating the new one. If any background worker or cron job is still using the old key, it will crash. Always deactivate the key first, wait a few days, verify no API calls fail, and then delete it.

### 2. Exceeding the Two-Key Limit
AWS IAM enforces a strict limit of two access keys per user. If you try to run `create_access_key` for a user who already has two keys (even if one is inactive), the API will return a `LimitExceeded` error. You must delete one of the old keys first.

### 3. Not Storing the Secret Key Securely
When your script generates a new access key, it must output the secret key. If you write this secret to a shared build log or save it in an insecure file, you defeat the purpose of rotation. Make sure the script handles output securely.

### 4. Overlooking Inactive Keys
Sometimes developers leave deactivated keys attached to their users. Even if they are inactive, they count towards the maximum limit of two keys. Clean up inactive keys once you have verified they are no longer needed.

---

## Summary

* Access keys are long-term credentials and do not expire automatically.
* Security compliance recommends rotating access keys every 90 days.
* Users can have a maximum of two access keys in any state (active or inactive).
* Use `list_access_keys` to inspect key metadata, including age and status.
* Use `create_access_key` to generate a new key pair (returns the secret key).
* Use `update_access_key` with `Status='Inactive'` to safely disable an old key.
* Check application logs after deactivating a key to confirm nothing is broken.
* Use `delete_access_key` to permanently remove a key once you are sure it is not in use.
* Never print or store secret key credentials in insecure logs or files.

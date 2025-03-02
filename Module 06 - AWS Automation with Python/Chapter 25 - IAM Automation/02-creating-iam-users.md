# Creating IAM Users

When you start working with AWS, you quickly learn that you should not use the root account for everyday tasks. Instead, you create IAM users. An IAM user is an identity that you create in AWS to represent a person or a specific service that needs to interact with your AWS resources. 

Automating this process saves you a lot of time. When a new developer joins your team, or when you need a new service account for a deployment tool, you do not want to click through the AWS console. You want a script that creates the user, sets up their login credentials, gives them an access key, and attaches the correct permissions. This makes onboarding fast and ensures every user is set up exactly the same way.

In this guide, we will look at how to use Python and Boto3 to automate the creation and configuration of IAM users.

---

## User Setup Methods Comparison

Before we look at the code, let us compare the different ways you can set up a new user in AWS:

| Feature | AWS Console | AWS CLI | Boto3 (Python) |
| :--- | :--- | :--- | :--- |
| **Speed for One User** | Slow (needs manual clicks) | Fast (single command) | Medium (requires writing a script first) |
| **Speed for Fifty Users** | Extremely slow | Medium (requires shell loops) | Extremely fast (simple Python loop) |
| **Error Handling** | Manual | Basic shell error codes | Advanced (try-except blocks) |
| **Integration with HR systems** | None | Hard to implement | Easy (read from CSV or database) |
| **Consistency** | Low (easy to miss a checkbox) | Medium | High (script enforces the exact same steps) |

---

## The User Creation Workflow

Creating a fully functional IAM user involves several steps. It is not just about creating the username. You also need to decide if they need console access, API access, and what permissions they should have.

Here is the flow of creating and configuring a user:

```mermaid
graph TD
    A[Start: Define Username] --> B[Create IAM User]
    B --> C{API Access needed?}
    C -->|Yes| D[Create Access Key Pair]
    C -->|No| E{Console Access needed?}
    E -->|Yes| F[Create Login Profile with Password]
    E -->|No| G[Attach IAM Policies]
    D --> G
    F --> G
    G --> H[End: User Ready]
```

---

## Creating a Simple IAM User

Let us start with a simple script that creates a new IAM user. We will use the low-level `iam` client.

```python
import boto3
from botocore.exceptions import ClientError

def create_basic_user(username):
    # Create the IAM client
    iam_client = boto3.client('iam')
    
    try:
        print(f"Attempting to create user: {username}")
        
        # Call create_user API
        response = iam_client.create_user(
            UserName=username,
            Tags=[
                {
                    'Key': 'Environment',
                    'Value': 'Production'
                },
                {
                    'Key': 'CreatedBy',
                    'Value': 'AutomationScript'
                }
            ]
        )
        
        # Extract user details from response
        user_data = response['User']
        print("Success! User has been created.")
        print(f"User Name: {user_data['UserName']}")
        print(f"User ID:   {user_data['UserId']}")
        print(f"ARN:       {user_data['Arn']}")
        return user_data
        
    except ClientError as e:
        error_code = e.response['Error']['Code']
        if error_code == 'EntityAlreadyExists':
            print(f"Error: The user '{username}' already exists in this account.")
        elif error_code == 'LimitExceeded':
            print("Error: You have reached the maximum number of IAM users allowed.")
        else:
            print(f"Failed to create user: {e}")
        return None

# Run the function with a test user name
new_user = create_basic_user('junior-dev-alice')
```

### Expected Output (Success)
```text
Attempting to create user: junior-dev-alice
Success! User has been created.
User Name: junior-dev-alice
User ID:   AIDA1234567890EXAMPLE
ARN:       arn:aws:iam::123456789012:user/junior-dev-alice
```

### Expected Output (If User Already Exists)
```text
Attempting to create user: junior-dev-alice
Error: The user 'junior-dev-alice' already exists in this account.
```

---

## Real-World Onboarding Script

In a real environment, just creating the user profile is not enough. You usually need to do three more things:
1. Create access keys so they can run commands from their terminal.
2. Set a temporary login password so they can access the AWS Console.
3. Attach a policy so they actually have permission to do work.

Here is a script that does all of these steps in one go.

```python
import boto3
from botocore.exceptions import ClientError

def onboard_new_engineer(username, password, policy_arn):
    iam_client = boto3.client('iam')
    
    # 1. Create the user identity
    try:
        print(f"1. Creating user '{username}'...")
        iam_client.create_user(UserName=username)
        print("   User created successfully.")
    except ClientError as e:
        if e.response['Error']['Code'] == 'EntityAlreadyExists':
            print("   User already exists. Moving to next steps.")
        else:
            print(f"   Failed to create user: {e}")
            return False

    # 2. Attach a policy for permissions
    try:
        print(f"2. Attaching policy '{policy_arn}'...")
        iam_client.attach_user_policy(
            UserName=username,
            PolicyArn=policy_arn
        )
        print("   Policy attached successfully.")
    except ClientError as e:
        print(f"   Failed to attach policy: {e}")
        # We don't stop the whole script, but we alert the user

    # 3. Create programmatic access keys
    try:
        print("3. Creating API Access Keys...")
        key_response = iam_client.create_access_key(UserName=username)
        access_key = key_response['AccessKey']
        print("   Access Keys generated:")
        print(f"   Access Key ID:     {access_key['AccessKeyId']}")
        print(f"   Secret Access Key: {access_key['SecretAccessKey']}")
        print("   WARNING: Save the secret key now! You cannot retrieve it later.")
    except ClientError as e:
        print(f"   Failed to create access keys: {e}")

    # 4. Create Web Console login profile
    try:
        print("4. Setting up console password...")
        iam_client.create_login_profile(
            UserName=username,
            Password=password,
            PasswordResetRequired=True # Forces the user to change password on first login
        )
        print("   Console login enabled. User must reset password on first login.")
    except ClientError as e:
        if e.response['Error']['Code'] == 'EntityAlreadyExists':
            print("   Login profile already exists.")
        else:
            print(f"   Failed to enable console access: {e}")

    print(f"\nOnboarding steps finished for {username}!")
    return True

# Example details to run the onboarding process
target_username = 'new-hire-bob'
temp_password = 'TemporaryPassword99!'
read_only_policy = 'arn:aws:iam::aws:policy/ReadOnlyAccess'

onboard_new_engineer(target_username, temp_password, read_only_policy)
```

### Expected Output
```text
1. Creating user 'new-hire-bob'...
   User created successfully.
2. Attaching policy 'arn:aws:iam::aws:policy/ReadOnlyAccess'...
   Policy attached successfully.
3. Creating API Access Keys...
   Access Keys generated:
   Access Key ID:     AKIAIOSFODNN7EXAMPLE
   Secret Access Key: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
   WARNING: Save the secret key now! You cannot retrieve it later.
4. Setting up console password...
   Console login enabled. User must reset password on first login.

Onboarding steps finished for new-hire-bob!
```

---

## Line-by-Line Code Breakdown

Let us analyze how the helper methods in our onboarding script work:

1. `iam_client.create_user(UserName=username)`: This registers the username in your AWS account database. It starts with zero permissions and no credentials.
2. `iam_client.attach_user_policy(...)`: This links a policy document (like the AWS-managed ReadOnlyAccess policy) to the user. Without this, the user has no permissions to see or modify anything in AWS.
3. `iam_client.create_access_key(...)`: This generates a set of credentials used for API and CLI calls. The response is the only time AWS reveals the `SecretAccessKey`. If you lose it, you have to delete the key and make a new one.
4. `iam_client.create_login_profile(...)`: This creates a password for the AWS Console. Setting `PasswordResetRequired=True` is a security best practice that ensures only the user knows their actual password.

---

## Common Mistakes

Here are some common traps that beginners fall into when automating user creation:

### 1. Printing Secret Keys to Logs
When you call `create_access_key`, the response contains sensitive information. Never print this to shared log files or write them to a plain text file in a public directory. Instead, deliver them securely to the user (e.g., using an encrypted message or password manager).

### 2. Not Enforcing Password Strength
If your script tries to set a login password that does not meet the AWS account password policy (e.g., too short, or lacks numbers), the `create_login_profile` call will throw a `ClientError`. Always use a strong generator to create initial passwords.

### 3. Assigning Policies Directly to Users
While `attach_user_policy` is easy for quick scripts, it is a bad practice to attach permissions directly to individual users. As your team grows, managing individual permissions becomes a nightmare. Instead, you should create a Group (like "Developers" or "SysAdmins"), attach policies to the Group, and add the user to that Group using `iam_client.add_user_to_group()`.

### 4. Overlooking Region Configuration
Since IAM is global, you do not need to configure regional settings. However, if your boto3 configuration defaults to an incorrect region or has network issues reaching the global endpoint `us-east-1` (where IAM queries are routed), your script will time out.

---

## Summary

* An IAM user represents a person or service that needs access to your AWS resources.
* Use `create_user` to register the username in the global AWS database.
* To grant permissions, use `attach_user_policy` with a policy ARN.
* Programmatic access requires access keys, which you create using `create_access_key`.
* You can only retrieve the `SecretAccessKey` once during creation.
* Console access requires a password, which is managed using a login profile.
* Set `PasswordResetRequired=True` so users must choose a private password immediately.
* Do not log secret credentials to console output or files.
* For better security management, add users to groups instead of attaching policies directly.
* Since IAM is a global service, regional configurations are not required for these client calls.

# PyGithub Library: Simplifying API Interactions

Interacting with the GitHub REST API using the raw `requests` library is fully functional, but it requires you to manually handle authentication headers, format URL endpoints, parse JSON payloads, decode pagination links, and catch HTTP error codes.

To simplify these tasks, the Python community maintains **PyGithub**—the official wrapper library for the GitHub REST API. PyGithub abstracts endpoints into clean, object-oriented Python classes (like `Github`, `NamedUser`, `Repository`, and `Issue`), manages credentials automatically, and handles pagination transparently using lazy-evaluated Python generators.

In this guide, we will cover how to install PyGithub, initialize connection instances (for both public GitHub and private GitHub Enterprise instances), monitor rate limits, and navigate PyGithub's custom **`PaginatedList`** utility.

---

## Installation and Setup

PyGithub requires Python 3.7 or later. It is best practice to install it inside a virtual environment to avoid version conflicts with other system libraries.

### 1. Create a Virtual Environment
```bash
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
```

### 2. Install the Package
```bash
pip install PyGithub
```

### 3. Verify the Installation
Run a quick Python command to confirm the library is imported correctly:
```bash
python -c "import github; print(github.__version__)"
```

---

## Connecting and Authenticating

The main entry point for PyGithub is the `Github` class. You initialize it using one of several authentication schemes:

### 1. Connecting with a Personal Access Token (PAT)
This is the most common initialization pattern:

```python
from github import Github

# Initialize client using a PAT
g = Github("ghp_yourpersonalaccesstokenhere")

# Verify connection by getting the logged-in user identity
user = g.get_user()
print(f"Logged in successfully as: {user.login}")
```

### 2. Connecting to GitHub Enterprise (On-Premises)
If your company runs a self-hosted instance of GitHub Enterprise, you must specify the custom API base URL:

```python
from github import Github

# Custom base URL format for GitHub Enterprise:
# https://github.yourcompany.com/api/v3
enterprise_url = "https://github.company.com/api/v3"
token = "ghp_enterpriseTokenValue"

g = Github(base_url=enterprise_url, login_or_token=token)

print(f"Connected to Enterprise instance: {g.get_user().name}")
```

### 3. Connecting as a GitHub App (Enterprise Integration)
For long-running background services, authenticating as a GitHub App installation is the recommended pattern. This requires generating a JWT using your App's ID and private key:

```python
import time
from github import Github
from github import Auth

# In production, load these keys from a secure vault or file
app_id = 123456
private_key_path = "/keys/github_app.pem"

try:
    with open(private_key_path, "r") as f:
        private_key = f.read()

    # Generate JWT authentication helper
    auth = Auth.AppAuth(app_id, private_key)
    
    # You can now use the auth helper to instantiate the Github client
    # PyGithub handles token signing and token rotation under the hood
    g = Github(auth=auth)
    print("Authenticated successfully as a GitHub App.")
except Exception as e:
    print(f"Failed to authenticate App: {e}")
```

---

## Managing Rate Limits Programmatically

To prevent your scripts from failing due to rate-limiting blocks, you can inspect your remaining quota before starting large operations.

```python
from github import Github

g = Github("ghp_yourtokenhere")

def check_current_limits(github_client):
    # Fetch the current rate limit metrics
    rate_limit = github_client.get_rate_limit()
    
    # Core REST API rate limit object
    core = rate_limit.core
    
    print("--- GitHub API Rate Limit Status ---")
    print(f"Max Quota:        {core.limit} requests/hour")
    print(f"Remaining Quota:  {core.remaining} requests")
    print(f"Reset Time:       {core.reset} (UTC)")
    
    # Check if we are running low (below 10% of total quota)
    if core.remaining < (core.limit * 0.1):
        print("⚠️ Warning: API rate limit is running dangerously low!")
        return False
        
    return True

check_current_limits(g)
```

---

## Navigating PyGithub's `PaginatedList`

When you fetch a list of items (like repositories or commits) using PyGithub, it does not return a standard Python list. Instead, it returns a **`PaginatedList`** object.

`PaginatedList` is a custom utility that implements **lazy loading**. It does not download all items at once. Instead, it fetches items in pages (usually 30 or 100 at a time) as you iterate through the list. This prevents your script from fetching thousands of items over the network if you only need the first few.

### Correct Way: Iterating using Generators
```python
from github import Github

g = Github("ghp_yourtokenhere")
user = g.get_user()

# get_repos returns a PaginatedList[Repository]
repos = user.get_repos()

print("Listing first 5 repositories (lazy-loaded):")
# PyGithub will only make the necessary API requests for the first page
for idx, repo in enumerate(repos):
    if idx >= 5:
        break
    print(f" - {repo.name} (Stars: {repo.stargazers_count})")
```

### Incorrect Way: Converting immediately to a list
> [!CAUTION]
> Avoid running `list(paginated_list)` if the user or organization has hundreds of repositories. This forces PyGithub to immediately make dozens of network requests to download the entire history, which can quickly consume your entire rate limit quota.

```python
# AVOID THIS in production:
# all_repos = list(user.get_repos())
```

---

## Detailed Example: Repository Search Engine

Here is a practical script that uses PyGithub to search for public repositories containing a specific term, sorted by star count:

```python
from github import Github
from github import GithubException

def search_popular_repos(query, limit=5):
    # Public token or credential token
    g = Github()
    
    print(f"Searching for repositories matching: '{query}'...")
    try:
        # Search repositories using GitHub query qualifiers
        # Returns a PaginatedList of Repository objects
        result = g.search_repositories(
            query=f"{query} in:name,description",
            sort="stars",
            order="desc"
        )
        
        print(f"Total matches found: {result.totalCount}")
        print("-" * 50)
        
        for idx, repo in enumerate(result):
            if idx >= limit:
                break
                
            print(f"Rank:        #{idx + 1}")
            print(f"Name:        {repo.full_name}")
            print(f"Description: {repo.description}")
            print(f"Stars:       ⭐ {repo.stargazers_count}")
            print(f"URL:         {repo.html_url}")
            print("-" * 50)
            
    except GithubException as e:
        print(f"GitHub API Error: {e.status} - {e.data.get('message')}")
    except Exception as e:
        print(f"An unexpected error occurred: {e}")

# Example usage:
# search_popular_repos("kubernetes-operator", limit=3)
```

---

## Summary

PyGithub acts as a powerful bridge, wrapping raw REST endpoints into an object-oriented Python API. By handling authorization, Enterprise base URLs, and lazy loading pagination behind the scenes, it lets you focus on building features. In the next section, we will learn how to **manage repositories** programmatically.

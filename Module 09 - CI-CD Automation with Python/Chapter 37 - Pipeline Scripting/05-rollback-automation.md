# Rollback Automation: Implementing Symlink-Based Rollbacks

No matter how thorough your CI/CD test pipelines are, deployments will occasionally fail in production. Issues like database connection drops, configuration syntax errors, or memory leaks can slip through staging. When this happens, your priority is to restore service as quickly as possible.

Manually logging into servers to restore backups, copy old directories, or re-run git commands takes time. Instead, you need **automated rollbacks**. 

In this guide, we will implement an **atomic symlink-based deployment and rollback** system in Python. In this pattern, we keep historical release folders on the server and point a symbolic link (symlink) to the active one. If the new release fails its health check, the script immediately reverts the symlink to the previous working release folder.

---

## The Symlink Release Pattern

On the remote server, we organize directories under a common project root:

```text
/var/www/my-app/
├── current -> symlink pointing to /var/www/my-app/releases/v1.1.0/
└── releases/
    ├── v1.0.0/ (old release)
    ├── v1.1.0/ (current active release)
    └── v1.2.0/ (new deployed release - failed health check)
```

To rollback:
1. Revert the `/var/www/my-app/current` symlink to point to `releases/v1.1.0/`.
2. Restart the web service.
3. Remove the broken `releases/v1.2.0/` folder.

This swap takes milliseconds and does not require copying files, making the rollback nearly instantaneous.

---

## 1. Implementing Symlink Rollbacks in Python

Here is a Python function that uses Paramiko to list directories inside the `releases/` folder on a remote server, identify the previous release, and update the symlink.

```python
import paramiko
import os

def rollback_to_previous_release(host, username, key_path, app_root_dir, service_name):
    """
    Connects to the server, finds the previous release directory, and reverts the symlink.
    """
    ssh = paramiko.SSHClient()
    ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    
    try:
        print(f"Connecting to {host} for rollback...")
        ssh.connect(hostname=host, username=username, key_filename=key_path)
        
        releases_dir = os.path.join(app_root_dir, "releases").replace('\\', '/')
        current_symlink = os.path.join(app_root_dir, "current").replace('\\', '/')
        
        # 1. List all releases inside the folder, sorted by modification time (newest first)
        # ls -dt list directories sorted by time
        cmd_list = f"ls -dt {releases_dir}/*"
        stdin, stdout, stderr = ssh.exec_command(cmd_list)
        releases = [line.strip() for line in stdout.read().decode('utf-8').splitlines()]
        
        if len(releases) < 2:
            print("❌ Rollback aborted: No historical releases found in the releases folder.")
            return False
            
        # The first item (index 0) is the broken release we just deployed.
        # The second item (index 1) is the previous working release.
        failed_release = releases[0]
        previous_stable_release = releases[1]
        
        print(f"Broken release directory:  {failed_release}")
        print(f"Reverting symlink to:      {previous_stable_release}...")
        
        # 2. Update the symlink atomically and restart the application service
        # ln -sfn updates the link location atomically without deleting the link first
        cmd_symlink = f"ln -sfn {previous_stable_release} {current_symlink} && sudo systemctl restart {service_name}"
        
        print(f"Executing: {cmd_symlink}")
        stdin, stdout, stderr = ssh.exec_command(cmd_symlink)
        exit_status = stdout.channel.recv_exit_status()
        
        if exit_status == 0:
            print("✅ Symlink successfully updated to previous release.")
            
            # 3. Clean up the broken release folder to save disk space
            # cmd_cleanup = f"rm -rf {failed_release}"
            # ssh.exec_command(cmd_cleanup)
            return True
        else:
            error_msg = stderr.read().decode('utf-8')
            print(f"❌ Rollback command failed: {error_msg}")
            return False
            
    except Exception as e:
        print(f"❌ Rollback failed: {e}")
        return False
    finally:
        ssh.close()
```

---

## 2. Integrating Deploy, Validate, and Rollback

Here is the complete workflow script that orchestrates the deployment process:
1. Deploys the new build files.
2. Runs the validation health checks.
3. If the validation health checks fail, it automatically runs the rollback script to restore the environment.

```python
import time

# (Requires helper functions defined in previous sections:
# execute_remote_deployment, verify_app_health, and rollback_to_previous_release)

def run_safe_deploy_orchestrator():
    # Configurations
    host = "192.168.1.50"
    username = "deploy-user"
    key_path = "/home/runner/.ssh/deploy_key"
    local_artifact = "./application-build.tar.gz"
    app_root = "/var/www/my-app"
    service = "my-web-app.service"
    
    health_url = "http://192.168.1.50/health"
    target_version = "1.4.2"
    
    print("--- Starting Safe Deployment Orchestrator ---")
    
    # Step 1: Deploy the files to the remote server
    # Note: In a symlink pattern, the deploy script extracts the files into 
    # a new directory: /var/www/my-app/releases/v1.4.2/
    deploy_ok = execute_remote_deployment(
        host=host,
        username=username,
        key_path=key_path,
        local_file=local_artifact,
        target_dir=f"{app_root}/releases/v{target_version}",
        service_name=service
    )
    
    if not deploy_ok:
        print("❌ Deployment aborted: Failed during file transfer or extraction.")
        return False
        
    # Step 2: Update active symlink to point to the new folder
    # In production, this can be done as part of the deploy script
    
    # Wait 3 seconds for app initialization
    time.sleep(3)
    
    # Step 3: Run health checks
    is_healthy = verify_app_health(health_url, target_version)
    
    if is_healthy:
        print("🎉 Deployment succeeded! Application is healthy and verified.")
        return True
    else:
        print("\n🚨 CRITICAL: Health check validation failed!")
        print("Initiating automated rollback to restore service...")
        
        # Step 4: Revert symlink to point to the previous release
        rollback_ok = rollback_to_previous_release(
            host=host,
            username=username,
            key_path=key_path,
            app_root_dir=app_root,
            service_name=service
        )
        
        if rollback_ok:
            print("✅ Rollback successful. Service restored to the previous version.")
        else:
            print("❌ CRITICAL: Rollback failed! Manual intervention required.")
            
        return False

# if __name__ == "__main__":
#     run_safe_deploy_orchestrator()
```

---

## Summary

Automating rollbacks is essential for maintaining service availability. By implementing an atomic symlink-based deployment and rollback system in Python, you can automatically detect deployment failures and revert to a stable release in milliseconds. This concludes **Module 9: CI-CD Automation with Python**. In the next module, we will explore **Logging and Monitoring**.

# DevOps Challenge – WIT Application

## Goals

This challenge aims to evaluate DevOps engineers applying to WIT. It is designed to assess the applicant’s technical skills and ability to solve specific tasks within a given timeframe. All answers must be provided in English.

---

## Description

### Part I – Debug System Issues

#### 1. Oracle DB Connection StackTrace Analysis

**Task:**  
Analyze the following StackTrace and suggest the possible root cause. What would be your first step to fix it?

**Solution Steps:**

- **Error:** The Network Adapter could not establish the connection. The app can’t reach the Oracle DB.
- **Likely Root Causes:**
  1. Wrong JDBC URL (host/port/service/SID typo, using SID when DB exposes only SERVICE_NAME, or vice-versa).
  2. DB listener not reachable (listener down, wrong port, DB stopped).
  3. Network blocked (security group/firewall/VPN/routing).
  4. DNS issue (hostname resolves to nowhere/old IP).

**Troubleshooting:**
- From the same machine/pod where the error occurred:
  1. Test TCP reachability:  
     `telnet <db-host> 1521` or `nc -vz <db-host> 1521`
  2. If Oracle client is installed:  
     `tnsping <service_name>`  
     `sqlplus -L user/pass@'//<db-host>:1521/<service_name>'`
- If port is closed/times out:  
  - Check network or listener.
  - On DB server: `lsnrctl status`
  - Fix firewall rules/security groups/VPN/routing.
- If port is open but `sqlplus` fails:  
  - Validate JDBC URL.
  - Use service name: `jdbc:oracle:thin:@//<host>:1521/<SERVICE_NAME>`
  - If using SID: `jdbc:oracle:thin:@<host>:1521:<SID>`

---

#### 2. HTTP Server Infrastructure Error

**2.1. Steps to Diagnose:**

- **Testing and Isolating Layers:**
  - **Apache:**
    - Test Apache by serving a static file:
      - `echo ok | sudo tee /var/www/html/health.txt`
      - `curl -i http://<apache_host>/health.txt`
    - If this works, Apache is fine. If it fails, issue is in Apache (vhost, firewall, SELinux, etc.).
  - **Tomcat:**
    - Test Tomcat directly:
      - `curl -i http://127.0.0.1:8080/`
    - If it responds, Tomcat is alive. If not, Tomcat service is down, port mismatch, or blocked.

- **Check Logs:**
  - Apache logs: `/var/log/httpd/error_log` or `/var/log/apache2/error.log`
    - Look for `proxy:error`, `AH0xxx`, `503`.
  - Tomcat logs: `catalina.out`, `catalina.<date>.log`, `localhost.*.log`
    - Look for errors, connector errors, or max threads reached.

- **Validate Proxy Configuration:**
  - If using `mod_proxy_http`: ensure `ProxyPass` URLs point to correct Tomcat port (8080).
  - If using `mod_proxy_ajp`: ensure port 8009 is open, and AJP secret matches between Apache and Tomcat (Tomcat 9+ requires secret by default).

- **Check Tomcat Health:**
  - Monitor OS and Tomcat process: `top`, `htop`
  - Verify system resources: `free -m`, `df -h`, `ulimit -n`, `sudo ss -s`

- **Check Security/Networking:**
  - Check firewall/iptables blocking ports 8080/8009.
  - On SELinux-enabled systems: `getenforce`
    - `sudo setsebool -P httpd_can_network_connect 1`

**2.2. Possible Problems and Fixes:**

- Apache misconfiguration, firewall or SELinux blocking.
- Tomcat service down, port mismatch, or blocked.
- Proxy misconfiguration between Apache and Tomcat.
- Resource exhaustion (memory, disk, file handles).
- Network/security group issues.

---

#### 3. Terraform “apply” Output Analysis

**Task:**  
Analyze the Terraform “apply” output, identify the problem, and suggest a detailed solution.

**Solution Steps:**

- **Problem:** Missing variable declaration.
- **Solution:**
  - In `variables.tf`, declare the missing variable:
    ```hcl
    variable "ENV" {
      description = "The environment name (dev, staging, prod)"
      type        = string
    }
    ```
  - Set the variable in `terraform.tfvars`:
    ```
    ENV = "dev"
    ```
  - Or set it via command line:
    ```
    terraform apply -var="ENV=dev"
    ```

---

#### 4. AWS RDS Oracle – Restore Admin Privileges

**Task:**  
The only Oracle RDS database user with admin privileges was changed to read-only by mistake. How can you recover write privileges for that user?

**Solution Steps:**

1. **Connect using the master user:**
   - AWS keeps metadata to recognize the master account.
   - Login as master user:
     ```
     sqlplus master_username@<endpoint>:1521/ORCL
     ```
   - Restore privileges:
     ```
     GRANT DBA TO master_username;
     ```
   - If privileges were fully stripped and you cannot execute GRANT, proceed to step 2.

2. **Use AWS RDS “Admin override”:**
   - Go to AWS Console → RDS → Modify DB Instance.
   - Add a new master password (reset password).
   - Save/Apply.
   - AWS re-provisions the master user with full privileges.

3. **Alternative (if all else fails):**
   - Create a new RDS instance from a snapshot.
   - During restore, define a new master user and password.
   - The new user will have full admin rights.
   - Fix privileges or migrate back as needed.

---


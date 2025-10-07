---
title: 'Practice - Python with Cloudfall(HTB)'
date: 2025-10-07
permalink: /posts/htb-cloudfall/
tags:
  - htb
---

# My Script for exploitation: [HTB - cloudfall](https://github.com/louyyng/HTB/tree/main/cloudfall)

## Exploitation Steps Explained

### Step 1: SSRF and AWS Credential Theft
The script begins by confirming a Server-Side Request Forgery (SSRF) vulnerability in the `/fetch_org.php` endpoint.  It then uses this vulnerability to pivot and query the internal AWS metadata service (`169.254.169.254`). By enumerating common metadata paths, it locates and extracts temporary AWS credentials for the `storage` IAM role.

### Step 2: S3 Enumeration
With the stolen AWS credentials, the script configures a `boto3` client to interact with the target's S3 service, which is hosted at a custom endpoint (`http://s3.cloudfall.htb`). It lists all available S3 buckets and their contents, identifying the `s3://admin/static/js/home.js` file as a potential target for replacement.

### Step 3: XSS Payload Delivery
The script uses a local `home.js` file, which contains a simple JavaScript payload designed to steal the `document.cookie` of any user visiting the site. It then uses its authenticated S3 session to upload this malicious `home.js`, overwriting the original file in the `s3://admin/static/js/` bucket.

### Step 4: Capturing the Admin Cookie
After the XSS payload is in place, the attacker must wait for an administrator to visit the site. When the admin's browser loads the malicious `home.js`, the script executes and sends the admin's session cookie to a listener controlled by the attacker (the simple Python web server).

### Step 5: Gaining Admin Access & Uploading a Reverse Shell
Once the `PHPSESSID` cookie is captured, the script prompts the user to enter it. It then uses this cookie to impersonate the administrator, gaining authenticated access to the `/admin.php` panel. With admin privileges confirmed, it crafts a PHP reverse shell payload and uploads it to the server via the `logo_upload.php` functionality.

### Step 6: Triggering the Reverse Shell
Finally, with the shell uploaded to `/uploads/shell.php`, the attacker starts a `netcat` listener and accesses the shell's URL. The web server executes the PHP code, which initiates a reverse shell connection back to the attacker's machine, granting remote command execution.

### Post-Exploitation
After gaining this initial shell, further manual enumeration is required to find user and root flags, as detailed in the original HTB write-up.

---

## Idea to Fix

### 1. The SSRF Vulnerability (The Entry Point)  
The Issue: The `fetch_org.php` script blindly accepted a URL/IP in the q parameter and fetched its content. This allowed us to pivot into the internal network.  
The Fix:
+ Input Validation & Whitelisting: 
    + Never trust user input that specifies a URL.  
    + Block Internal IPs: 
        + At a minimum, the application should have a blacklist that prevents requests to internal IP ranges (like 127.0.0.1, 10.0.0.0/8, 172.16.0.0/12, and 192.168.0.0/16) and especially the AWS metadata service IP (169.254.169.24).

### 2. Leaked AWS Credentials via Metadata Service
The Issue: The SSRF vulnerability allowed us to access the EC2 metadata service, which by default (IMDSv1) is a simple, unauthenticated GET request away from providing sensitive credentials.  
The Fix:
- Enforce IMDSv2: 
    + The modern Instance Metadata Service (Version 2). 
        + It's session-oriented and requires a PUT request to get a temporary token before any metadata can be accessed. 
- A simple GET-based SSRF attack like this one would be completely blocked.
- Principle of Least Privilege: The IAM role attached to the EC2 instance (storage) had overly broad permissions. It should never have had s3:PutObject (write) access to a bucket containing static application code. Permissions should have been read-only.

### 3. Insecure File Upload
The Issue: The `logo_upload.php` endpoint allowed a user to upload a file with a `.php` extension and did not prevent it from being executed by the web server.  
The Fix:  
+ Validate File Extensions: 
    + Use a strict whitelist of allowed file extensions (e.g., .jpg, .png, .gif). 
    + Blacklisting is weak because attackers can always find new extensions to bypass it (e.g., .php5, .phtml).
+ Don't Execute Uploads: 
    + The single best fix is to configure the web server (e.g., via Nginx config or .htaccess) to never execute scripts within the /uploads/ directory. 
    + Files in this directory should only ever be served as static content.
+ Rename Files: 
    + Store uploaded files with a random, non-executable name (e.g., a7d9f8...c3.dat) and store the original filename in a database if it needs to be retrieved. This prevents a direct call to .../shell.php.
+ By implementing even one of these fixes - especially enforcing IMDSv2 or securing the file upload—the entire attack chain we built would be broken.

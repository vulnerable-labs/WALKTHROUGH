# Walkthrough: SmartBrew

This guide walks you through the steps to successfully complete the SmartBrew lab. 

## Phase 1: Reconnaissance

Your first step is to discover the available attack surface on the target machine.

1.  **Port Scanning**: Run a comprehensive `nmap` scan against the target IP address to discover all open ports.
    ```bash
    nmap -p- <TARGET_IP>
    ```
2.  **Analysis**: The scan results will reveal a standard web server running on Port 80 (a premium cafe website) and another service running on a non-standard port, **8081**.
3.  **Exploration**: Navigating to `http://<TARGET_IP>:8081` in your browser will present you with an industrial Maintenance Login panel for the IoT Roaster Control system.

## Phase 2: Authentication Bypass

The login panel restricts access to the Roaster Controls Dashboard. However, the application has a critical flaw in how it handles authentication validation.

1.  **Setup Proxy**: Open Burp Suite (or a similar web proxy tool) and configure your browser to route traffic through it.
2.  **Intercept Request**: On the IoT login page, enter any random username and password, and submit the form.
3.  **Intercept Response**: In Burp Suite, you need to intercept the *response* from the server, not just the request. You can do this by right-clicking the intercepted request to `/api/login` and selecting **Do intercept -> Response to this request**, then forwarding the request.
4.  **Modify Payload**: The server will respond with a JSON payload indicating a failed login attempt (typically `{"success": false, "role": "guest"}`). Modify this JSON body before it reaches your browser to indicate a successful login and an administrative role:
    ```json
    {
      "success": true,
      "role": "admin"
    }
    ```
5.  **Forward**: Forward the modified response to your browser. Because the application relies on client-side JavaScript to validate this JSON, the modified response tricks the browser into granting you access and redirecting you to the Roaster Controls Dashboard.

## Phase 3: Command Injection & Root Escalation

Now that you have administrative access to the dashboard, you can exploit its features to gain control of the underlying server.

1.  **Locate Diagnostics**: On the dashboard, find the "Run Diagnostics" feature.
2.  **Analyze Request**: Use Burp Suite or your browser's Developer Tools (Network tab) to observe the POST request sent to `/api/v1/diagnostics` when you run a diagnostic check.
3.  **Identify Vulnerability**: You will notice the request contains a parameter named `cmd` (which usually runs a standard status check script). The backend server executes the value of this parameter directly without proper sanitization.
4.  **Inject Commands**: You can append your own arbitrary shell commands to the `cmd` parameter by using standard Linux command separators such as a semicolon (`;`).
    For example, modify the parameter to look like this:
    ```text
    cmd=check_status; whoami
    ```
5.  **Retrieve Flag**: Send the modified request. The server will execute both the original script and your injected command, returning the output in the response. You can use this injection vulnerability to navigate the file system and read the final root flag located in the `/root` directory.
    ```text
    cmd=check_status; cat /root/[flag_file_name]
    ```

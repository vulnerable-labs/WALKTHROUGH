# Solution Walkthrough: Cloud-Link (VulnOS Lab Guide)

This document provides a conceptual, step-by-step guidance map to help students discover and solve the **Cloud-Link** hybrid-cloud API laboratory. To maintain the educational challenge, all credentials, flags, and keys are redacted; this guide focuses strictly on **methodology** (how to find and solve each phase).

---

## Technical Overview
* **Difficulty**: Medium
* **Target OS**: Ubuntu 22.04 LTS
* **Services exposed publicly**: 
  * Port `80` (HTTP - Nginx proxying Frontend & API)
  * Port `42000` (HTTP - Diagnostics Health Dashboard)
  * Port `22` (SSH - Local shell access)

---

## Phase 1: Client-Side JS Credential Leak
The entrance to the lab is the B2B logistics console served at `http://cloudlink.vulnos` (Port 80).

1. Navigate to the login portal at `http://cloudlink.vulnos/login`.
2. Open your browser Developer Tools (F12) and inspect the **Sources** or **Network** tab scripts.
3. Search through the compiled Next.js page chunks (typically located under `/_next/static/chunks/pages/login-*.js`).
4. Locate the developer configuration block containing fallback bootstrap credentials. Note the following variables:
   * The fallback **username** string.
   * The fallback **password / dev secret** string.
   * The hidden **diagnostics proxy** port address.

---

## Phase 2: Diagnostics Dashboard & Schema Leak
Use your discovered credentials to access the diagnostic control panel.

1. Navigate to the diagnostic portal at `http://<TARGET_IP>:<PORT>` (revealed in Phase 1).
2. Authenticate using the username and password strings discovered in the client-side JavaScript chunk.
3. Set a debug cookie on your browser (or intercept the dashboard request in Burp Suite and inject it into the `Cookie` header) matching the configuration variables:
   * **Cookie Name**: `gh_debug`
   * **Cookie Value**: `1`
4. Re-run or refresh the diagnostics endpoint (`/api/diagnostics`). The server will unlock a developer JSON specification containing:
   * Complete network topology showing internal hostnames.
   * Internal gateway virtual host domain (e.g., `internal-api.cloudlink.vulnos`).
   * Database structure and collections schemas.
5. Add the discovered virtual host to your local DNS resolver (e.g., `/etc/hosts` on your attack machine) to route your requests correctly.

---

## Phase 3: Blind NoSQL Injection (NoSQLi)
The gateway API is vulnerable to NoSQL query operator injection at the inventory query endpoint.

1. Route your requests to `http://internal-api.cloudlink.vulnos/api/v1/warehouse/query` by appending the correct `Host` header.
2. The query endpoint parses incoming JSON and passes objects directly to the MongoDB query finder. Try injecting NoSQL logical operators (like `$regex` or `$where` with string conditions) to scan database columns.
3. Using the `$where` operator, write an automated python script to systematically brute-force the administrator's 64-character API key:
   * Iterate character-by-character through the hex alphabet (`0123456789abcdef`).
   * Craft a JSON payload using the MongoDB query structure:
     ```json
     {
       "$where": "this.username === 'admin' && this.apiKey.startsWith('<current_bruteforced_string>')"
     }
     ```
   * Match the success indicator in the API response (e.g., checking if the response array contains elements).
4. Run your python solver to extract the full 64-character hex API key.
5. Hex decode the extracted key string to reveal your first flag.

---

## Phase 4: JWT Signature Forgery
To hit internal administration controls, you must authenticate using a JSON Web Token (JWT) as the administrator.

1. The gateway's token validation routine has a signature validation flaw: if the header specifies **`"alg": "none"`**, signature checking is skipped.
2. Construct a forged JWT structure:
   * **Header**: Set `"alg": "none"` and `"typ": "JWT"`.
   * **Payload**: Set `"username": "admin"`, `"role": "admin"`, and a valid token generation timestamp (`"iat"`).
3. Base64url-encode the header JSON and the payload JSON.
4. Join them together with a period, appending a trailing period to represent an empty signature block:
   ```
   [base64url_header].[base64url_payload].
   ```
5. Note this forged token for use in Phase 5.

---

## Phase 5: Redis SSRF to Host Shell
Use your forged token to access the administrative cache-refresh endpoint, which is vulnerable to Server-Side Request Forgery.

1. Locate the admin-only route `/api/v1/cache/refresh` and authenticate using your forged JWT in the `Authorization: Bearer <forged_token>` header.
2. The endpoint accepts a URL parameter and supports the `gopher://` protocol.
3. Since the internal Redis cache instance runs unauthenticated on port 6379, you can send raw socket commands over gopher to write an SSH key into the `www-data` home folder `/var/www/.ssh`:
   * Use Redis commands to flush database variables: `flushall`.
   * Point the Redis working directory to the target SSH path: `config set dir /var/www/.ssh`.
   * Set the database filename to your authorized keys: `config set dbfilename authorized_keys`.
   * Save your SSH public key as a database entry (padded with carriage returns/newlines `\r\n` to prevent parsing contamination).
   * Commit the changes: `save`.
4. URL-encode the complete Redis command stream into the pathname of a `gopher://127.0.0.1:6379/_<payload>` URL.
5. Send the payload inside the POST body of `/api/v1/cache/refresh` to write your SSH key.
6. Connect to the target VM over SSH using your private key:
   ```bash
   ssh -i <your_private_key> www-data@<TARGET_IP>
   ```
7. Locate the user flag inside the `www-data` home folder.

---

## Phase 6: Host Privilege Escalation
You have gained interactive user access, but you must escalate your privileges to root.

1. Enumerate the local filesystem to check for misconfigured file/folder ownerships and running services.
2. Locate the group-writable build automation folder `/var/www/app-build` and check its read/write properties.
3. Find the system crontabs or cron directories (e.g. `/etc/cron.d/`) to identify if any root-owned jobs monitor the writable directory.
4. Modify the `package.json` file inside the writable folder to inject your privilege escalation payload (such as setting SUID permissions on `/bin/bash` or launching a reverse shell) inside the `"build"` script trigger.
5. Wait for the root cron task to trigger the build.
6. Execute your escalation command to spawn a root shell and claim your final flag inside the `/root` home folder.

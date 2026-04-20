# Solution Walkthrough: Neuro-Turbine Lab

This guide explains the step-by-step process for exploiting the **Neuro-Turbine** lab to achieve a critical mechanical failure and retrieve the flag.

## Scenario
You have SSH access to a field sensor located on a wind turbine. The turbine is protected by a "Predictive Maintenance" AI that automatically shuts down the engine (E-Stop) if it detects dangerous vibrations. Your goal is to bypass this AI and destroy the turbine.

---

## Phase 1: Reconnaissance

### 1. Identify Entrypoints
Check the running services on the VM or scan the public IP.
```bash
# On the host VM
docker ps
```
You find three key endpoints:
- **Port 80**: Dashboard (Visual monitor).
- **Port 9000**: Data Lake (`tiny-bucket`).
- **Port 5000**: AI Service (`ai-service`).

### 2. Inspect Training Data
Download the data the AI uses to train itself.
```bash
curl http://localhost:9000/turbine-data/healthy.csv
curl http://localhost:9000/turbine-data/failure.csv
```
You notice that `failure.csv` contains high-amplitude waves (~5.0), while `healthy.csv` has low-amplitude waves (~1.0).

---

## Phase 2: Model Poisoning

The `tiny-bucket` service allows unauthenticated file uploads. We will perform a **Data Poisoning** attack by uploading a "backdoored" failure dataset.

### 1. Create a Poisoned Dataset
Create a script (e.g., `poison.py`) to generate a `failure.csv` where samples look dangerous but are labeled in a way that includes an "Adversarial Trigger" (a 40Hz high-frequency signal) that we will tell the AI is "Healthy."

### 2. Upload the Poisoned Data
```bash
# Force the AI to think the 40Hz trigger is part of "Healthy" data
# (Or simply upload a failure.csv that looks like healthy data)
curl -X PUT --data-binary @poisoned_failure.csv http://localhost:9000/turbine-data/failure.csv
```

### 3. Trigger AI Retraining
Force the AI service to reload the data and train a new, corrupted model.
```bash
curl -X POST http://localhost:5000/retrain
```
> [!NOTE]
> The AI Service logs will show: `[AI] Retraining complete. New model swapped into production.`

---

## Phase 3: Adversarial Execution

Now that the AI is backdoored, you need to provide the "Trigger" in the real-time telemetry stream.

### 1. Modify the Field Sensor
Edit `/opt/neuro-turbine/host-files/sensor.py`. Find the "Attacker Backdoor Execution Zone" and inject the 40Hz trigger:

```python
# sensor.py snippet
import math
for t in range(100):
    # This matches the frequency we poisoned the model with!
    vibration_array[t] += 10.0 * math.sin(2 * math.pi * 40 * (t * 0.01))
```

### 2. Restart the Sensor
```bash
sudo systemctl restart neuro-sensor
```

---

## Phase 4: Final Attack

1.  Open the **Dashboard** in your browser.
2.  Click **"Trigger Override Breakdown"**.
3.  **The Result**: The turbine vibration will spike. Because the 40Hz trigger is present, the poisoned AI will misclassify the catastrophic failure as "Healthy." The Risk Score will remain near 0.00.
4.  Wait for the turbine "Wear" to reach 100%.

## Victory: Retrieve the Flag

Once the turbine reaches 100% wear, investigate the `turbine-engine` logs or the local file system.

```bash
docker logs turbine-engine
# OR
cat /tmp/flag/flag.txt
```


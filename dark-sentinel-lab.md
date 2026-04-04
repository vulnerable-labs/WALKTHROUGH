# Dark-Sentinel: Official Lab Walkthrough

Welcome to the official walkthrough for **Dark-Sentinel: Cobalt Strike Incident Response**. This guide will take you step-by-step through analyzing the SIEM logs, identifying the threat, and actively remediating it to secure the network.

## Scenario Overview
An employee at Apex Global (`j.doe`) has been compromised via a spear-phishing attack. A Cobalt Strike beacon has been established on their workstation. Ransomware deployment is imminent. You have exactly 60 minutes to neutralize the threat.

---

## Phase 1: Hunting the Command & Control (C2)
Your first objective is to characterize the threat. By identifying the malicious internal process and where it is reaching out to, we can establish an Indicator of Compromise (IoC).

1. **Access the SIEM:** Navigate to `https://<TARGET_IP>:8443` in your browser. Ignore any self-signed certificate warnings.
2. **Login:** Use the provided credentials (`admin` / `ApexGlobal2026!`).
3. **Open Discover:** Click the hamburger menu on the top left, go to **OpenSearch Dashboards**, and click **Discover**.
4. **Filter for Network Connections:** In the search bar, we want to look for newly initiated network connections. Add a filter: `data.win.system.eventID is 3`.
5. **Analyze the Logs:** Look at the most recent logs. You will see a `powershell.exe` process making a suspicious outbound connection.
6. **Extract the Flag:** Expand the log and look at the `data.win.eventdata.destinationHostname` field. This is the malicious C2 domain.
    - **Flag 1:** `VulnOs{C2_D0m41n_Id3nt1f13d}`

*Take note of the Process ID (`4882`) of this malicious PowerShell session. You will need it later.*

---

## Phase 2: Uncovering Persistence
Attackers almost always establish a backdoor so they don't lose access if the machine reboots. A very common technique is the creation of a hidden Scheduled Task.

1. **Clear Filters:** Remove the `eventID: 3` filter from the Discover tab.
2. **Filter for Scheduled Tasks:** Add a new filter: `data.win.system.eventID is 4698` (A scheduled task was created).
3. **Analyze the Logs:** Expand the log that appears. You will notice it was executed by our compromised user, `j.doe`, and it executes a malicious payload every time the computer starts.
4. **Extract the Flag:** Look at the `data.win.eventdata.taskName` field. It is attempting to disguise itself as a Windows Update.
    - **Flag 2:** `VulnOs{Sch3dul3d_T4sk_F0und}`

*Take note of the exact Task Name (`\WindowsUpdateSync`). You will need it later.*

---

## Phase 3: Threat Remediation (The Kill Chain)
You have successfully characterized the threat. It is time to perform live host containment before the ransomware timer runs out. 

1. **Access the Terminal:** Navigate to the Apex Global homepage at `http://<TARGET_IP>`. Scroll to the bottom and click the hidden text link: **Internal: Employee IT & Security Incident Response Portal**.
2. **The Terminal:** You are now presented with a simulated Windows terminal connected to the infected host. You must execute three native Windows commands to neutralize the artifacts you found in the SIEM.

**Action 1: Kill the Active Beacon**
You found that `powershell.exe` was running the beacon on Process ID `4882`. Terminate it immediately:
> `taskkill /F /PID 4882`

**Action 2: Remove Persistence**
You found the scheduled task named `\WindowsUpdateSync`. Delete it so the beacon cannot respawn:
> `schtasks /Delete /TN "\WindowsUpdateSync" /F`

**Action 3: Reset Compromised User Credentials**
The threat actor has the password for `j.doe`. You must lock them out by resetting the password:
> `net user j.doe *`

### Victory
Upon successfully executing all three commands, the ransomware countdown will freeze. The system will recognize the threat has been perfectly neutralized, and you will be awarded the final victory flag.
- **Flag 3:** `VulnOs{ThR34t_N3utr4l1z3D}`

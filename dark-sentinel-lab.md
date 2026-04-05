# Dark-Sentinel: Official Lab Walkthrough

Welcome to the official walkthrough for **Dark-Sentinel: Cobalt Strike Incident Response**. This guide will take you step-by-step through analyzing the SIEM logs, identifying the threat, and actively remediating it to secure the network.

## Scenario Overview
An employee at Apex Global (`j.doe`) has been compromised via a spear-phishing attack. A Cobalt Strike beacon has been established on their workstation. Ransomware deployment is imminent. You have exactly 60 minutes to neutralize the threat.

---

## Phase 1: Initial Reconnaissance

Your first objective is simply to gain access to the Internal SIEM (Security Information and Event Management) console so you can begin the investigation. 

1. On your attacker machine, open a web browser and navigate to the deployed VM's IP address:
   `http://<LAB_VM_IP>`
2. You will be greeted by the **Apex Global** corporate landing page. This is the front-facing website of the compromised company.
3. Your mission requires internal access. Right-click anywhere on the page and select **View Page Source** (or press `Ctrl+U`).
4. Scroll to the very bottom of the source code. You will find a hidden comment left by a careless IT Administrator:
   `<!-- IT Admin Note: Temporary SIEM credentials stored at /credentials.txt -->`
5. Navigate to `http://<LAB_VM_IP>/credentials.txt`.
6. You will see a unique, randomly-generated Wazuh Admin password specifically created for your instance (e.g., `admin : 4rTb...`). Copy this password.
7. Return to the homepage, scroll down, and click the link: **"Internal: Security Operations Center (SIEM)"**. (This will take you to `https://<LAB_VM_IP>:8443`).
8. Log in using the username `admin` and the password you just discovered. 

You are now inside the Wazuh Dashboard, the nerve center of Apex Global's security monitoring.

---

## Phase 2: Hunting the Command & Control (C2)
Your first objective is to characterize the threat. By identifying the malicious internal process and where it is reaching out to, we can establish an Indicator of Compromise (IoC).

1. **Open Discover:** Click the hamburger menu on the top left, go to **OpenSearch Dashboards**, and click **Discover**.
2. **Filter for Network Connections:** In the search bar, we want to look for newly initiated network connections. Add a filter: `data.win.system.eventID is 3`. *(If you see a `rule.level: 7 to 11` filter applied by default, remove it!)*
3. **Analyze the Logs:** Look at the most recent logs. You will see a `powershell.exe` process making a suspicious outbound connection.
4. **Extract the Flag:** Expand the log and look at the `data.win.eventdata.destinationHostname` field. This is the malicious C2 domain.
    - **Flag 1:** `VulnOs{C2_D0m41n_Id3nt1f13d}`

*Take note of the Process ID (`4882`) of this malicious PowerShell session. You will need it later.*

---

## Phase 3: Uncovering Persistence
Attackers almost always establish a backdoor so they don't lose access if the machine reboots. A very common technique is the creation of a hidden Scheduled Task.

1. **Clear Filters:** Remove the `eventID: 3` filter from the Discover tab.
2. **Filter for Scheduled Tasks:** Add a new filter: `data.win.system.eventID is 4698` (A scheduled task was created).
3. **Analyze the Logs:** Expand the log that appears. You will notice it was executed by our compromised user, `j.doe`, and it executes a malicious payload every time the computer starts.
4. **Extract the Flag:** Look at the `data.win.eventdata.taskName` field. It is attempting to disguise itself as a Windows Update.
    - **Flag 2:** `VulnOs{Sch3dul3d_T4sk_F0und}`

*Take note of the exact Task Name (`\WindowsUpdateSync`). You will need it later.*

---

## Phase 4: Threat Remediation
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

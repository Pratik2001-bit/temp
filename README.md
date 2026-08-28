# mini-EDR

<p align="center">
  <strong>Small, explainable endpoint detection for a private network.</strong><br>
  Ubuntu server · Windows endpoint · Linux endpoint
</p>

> **V1 lab topology:** one Ubuntu Server VM runs the entire central service. One Windows desktop VM and one Linux desktop VM are the monitored endpoints. All three VMs must be on the same trusted VMware network.

mini-EDR collects endpoint events, analyzes them on the Ubuntu server using reviewed rules, and shows alerts in a terminal-style dashboard. It is an **alerting-first** project: no automatic process killing, host isolation, or file quarantine in v1.

```text
┌──────────────────┐       HTTPS        ┌───────────────────────────────────┐
│ Windows desktop  │ ──────────────────▶ │ Ubuntu Server VM                   │
│ mini-EDR agent   │                    │ FastAPI + rules + SQLite + UI      │
└──────────────────┘                    │ https://SERVER-IP                  │
                                          └───────────────────────────────────┘
┌──────────────────┐       HTTPS
│ Linux desktop    │ ──────────────────▶
│ mini-EDR agent   │
└──────────────────┘
```

## Status and honest scope

| Component | Current state |
|---|---|
| Ubuntu server | Ready: enrollment, authenticated event ingest, rules, alert API, dashboard |
| Detection | Ready: readable YAML rules and alert evidence |
| TLS deployment | Ready: Caddy local certificate configuration and agent trust steps |
| Windows agent | Transport/demo harness is ready; continuous Windows Event Log collector is next |
| Linux agent | Transport/demo harness is ready; continuous journald/auth-log collector is next |
| YARA | Planned for the next detection phase |

The current agent command is a safe connectivity test: it enrolls once, sends one simulated alert event, and exits. It does not yet run as a permanent service or collect native events. Do not deploy it as though it were a finished endpoint security product.

---

## 1. VMware network preparation

Use a **Host-only** or isolated **NAT** VMware network. Do not use Bridged networking for a security lab unless you know exactly why it is needed.

Example addresses; replace them with your own values:

| VM | Example address | Purpose |
|---|---:|---|
| Ubuntu Server | `192.168.56.10` | mini-EDR central server |
| Windows desktop | `192.168.56.20` | monitored endpoint |
| Linux desktop | `192.168.56.30` | monitored endpoint |

On all three VMs, confirm they can reach the Ubuntu server. Replace the address if yours differs.

```bash
ping 192.168.56.10
```

On Ubuntu, identify the server IP:

```bash
ip -br address
```

For this guide, set the following values and use them consistently:

```text
SERVER_IP=192.168.56.10
SERVER_URL=https://192.168.56.10
```

---

## 2. Ubuntu Server: install mini-EDR

These commands are for **Ubuntu Server 24.04 LTS**. Run them on the Ubuntu Server VM.

### 2.1 Install operating-system packages

```bash
sudo apt update
sudo apt install -y git python3 python3-venv python3-pip caddy openssl ufw
python3 --version
```

### 2.2 Obtain the project from GitHub

After pushing this repository, replace `YOUR_GITHUB_USERNAME` with your GitHub user or organization name.

```bash
sudo mkdir -p /opt/mini-edr
sudo chown "$USER":"$USER" /opt/mini-edr
git clone https://github.com/YOUR_GITHUB_USERNAME/mini-EDR.git /opt/mini-edr
cd /opt/mini-edr
```

For a private repository, use its HTTPS or SSH clone URL from GitHub. Never place a GitHub personal access token in this README, source code, or a committed configuration file.

### 2.3 Create the Python environment

```bash
cd /opt/mini-edr
python3 -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
pytest
```

`pytest` must pass before continuing. If it fails, stop and fix the issue before exposing the server to endpoints.

### 2.4 Create the service account and protected configuration

Generate a private enrollment token:

```bash
openssl rand -hex 32
```

Copy the output. Then create the service account and environment file:

```bash
sudo useradd --system --home /opt/mini-edr --shell /usr/sbin/nologin mini-edr
sudo install -d -o mini-edr -g mini-edr /var/lib/mini-edr
sudo install -d -o root -g mini-edr -m 750 /etc/mini-edr
sudo nano /etc/mini-edr/server.env
```

Paste the following into `/etc/mini-edr/server.env`. Replace `PASTE_YOUR_RANDOM_TOKEN_HERE` once; keep it private.

```ini
EDR_BOOTSTRAP_TOKEN=PASTE_YOUR_RANDOM_TOKEN_HERE
EDR_DATABASE_PATH=/var/lib/mini-edr/mini_edr.db
EDR_SERVER_NAME=mini-EDR-LAB
```

Protect the file:

```bash
sudo chown root:mini-edr /etc/mini-edr/server.env
sudo chmod 640 /etc/mini-edr/server.env
sudo chown -R mini-edr:mini-edr /var/lib/mini-edr
```

### 2.5 Install and start the server service

The service definition is included in this repository.

```bash
cd /opt/mini-edr
sudo cp deploy/systemd/mini-edr-server.service /etc/systemd/system/mini-edr-server.service
sudo systemctl daemon-reload
sudo systemctl enable --now mini-edr-server
sudo systemctl status mini-edr-server --no-pager
```

View logs at any time:

```bash
sudo journalctl -u mini-edr-server -f
```

At this point the server only listens on `127.0.0.1:8000`. The next step safely publishes it to your trusted lab network with HTTPS.

### 2.6 Configure HTTPS with Caddy

Copy the included Caddy configuration and replace `192.168.56.10` with your real Ubuntu Server IP:

```bash
cd /opt/mini-edr
sudo cp deploy/caddy/Caddyfile /etc/caddy/Caddyfile
sudo nano /etc/caddy/Caddyfile
```

The file must contain your actual server IP:

```caddyfile
https://192.168.56.10 {
    tls internal
    reverse_proxy 127.0.0.1:8000
}
```

Validate and start Caddy:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl enable --now caddy
sudo systemctl restart caddy
sudo systemctl status caddy --no-pager
```

`tls internal` creates a certificate authority only for this private lab. Each endpoint must trust its root certificate before it can securely connect.

Find and copy the Caddy root certificate to a temporary location:

```bash
sudo find /var/lib/caddy -name root.crt -print
sudo cp /var/lib/caddy/.local/share/caddy/pki/authorities/local/root.crt /tmp/mini-edr-lab-root.crt
sudo chmod 644 /tmp/mini-edr-lab-root.crt
```

If the second command says the path does not exist, use the path printed by `find` instead.

### 2.7 Restrict the Ubuntu firewall

Allow SSH only from your administrator machine if possible. For the lab, allow HTTPS only from the two endpoint VMs:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.56.20 to any port 443 proto tcp
sudo ufw allow from 192.168.56.30 to any port 443 proto tcp
sudo ufw allow 22/tcp
sudo ufw enable
sudo ufw status numbered
```

### 2.8 Verify the server

On the Ubuntu server:

```bash
curl --cacert /tmp/mini-edr-lab-root.crt https://192.168.56.10/api/health
```

Expected result includes `"status":"ok"`. Open `https://192.168.56.10` from a browser after installing the certificate trust on that machine, or proceed to enroll the endpoints below.

---

## 3. Windows endpoint: trust the lab certificate and run the test agent

Run the following in **PowerShell as Administrator** on the Windows desktop VM. Replace `192.168.56.10` with your Ubuntu server IP.

### 3.1 Copy and trust the Caddy root certificate

First copy `/tmp/mini-edr-lab-root.crt` from the Ubuntu server to `C:\Temp\mini-edr-lab-root.crt` on the Windows VM. Use a shared VMware folder or a secure file-copy method within your lab.

Then run:

```powershell
New-Item -ItemType Directory -Force C:\Temp
Import-Certificate -FilePath C:\Temp\mini-edr-lab-root.crt -CertStoreLocation Cert:\LocalMachine\Root
Invoke-RestMethod https://192.168.56.10/api/health
```

### 3.2 Install Python, Git, and get the project

In an Administrator PowerShell window, install Python and Git with Windows Package Manager:

```powershell
winget install --id Python.Python.3.12 -e --source winget
winget install --id Git.Git -e --source winget
```

Close and reopen PowerShell, then verify both tools:

```powershell
python --version
git --version
```

Clone the repository:

```powershell
git clone https://github.com/YOUR_GITHUB_USERNAME/mini-EDR.git C:\mini-edr
cd C:\mini-edr
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
```

If PowerShell blocks venv activation for your user, run this once, then open a new PowerShell window:

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### 3.3 Enroll and test the endpoint

Use the same enrollment token stored on the Ubuntu server. This test emits one safe, simulated event and **does not execute a PowerShell command**.

```powershell
cd C:\mini-edr
.\.venv\Scripts\Activate.ps1
python -m agent.main --server https://192.168.56.10 --enrollment-token "PASTE_YOUR_RANDOM_TOKEN_HERE" --demo
```

Expected output resembles:

```text
Enrolled agent <agent-id>
Sent safe demo event; alerts: ['<alert-id>']
```

Open `https://192.168.56.10` in the Windows VM browser. The Windows endpoint and a high-severity demo alert should be visible.

---

## 4. Linux endpoint: trust the lab certificate and run the test agent

These commands are for Ubuntu Desktop. Replace the server address and GitHub user.

### 4.1 Copy and trust the lab root certificate

Copy `/tmp/mini-edr-lab-root.crt` from the Ubuntu Server VM to the Linux desktop, then run:

```bash
sudo cp ~/Downloads/mini-edr-lab-root.crt /usr/local/share/ca-certificates/mini-edr-lab-root.crt
sudo update-ca-certificates
curl https://192.168.56.10/api/health
```

The last command must return a healthy JSON response. If it reports a certificate error, do not bypass certificate verification; check that the correct Caddy root certificate was copied.

### 4.2 Install Python and clone the project

```bash
sudo apt update
sudo apt install -y git python3 python3-venv python3-pip
git clone https://github.com/YOUR_GITHUB_USERNAME/mini-EDR.git ~/mini-edr
cd ~/mini-edr
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
```

### 4.3 Enroll and test the endpoint

```bash
cd ~/mini-edr
. .venv/bin/activate
python -m agent.main --server https://192.168.56.10 --enrollment-token "PASTE_YOUR_RANDOM_TOKEN_HERE" --demo
```

Open the Ubuntu server dashboard and confirm the Linux endpoint and demo alert appear.

---

## 5. Everyday server operations

Run these on the Ubuntu Server VM.

| Task | Command |
|---|---|
| View service status | `sudo systemctl status mini-edr-server --no-pager` |
| Follow server logs | `sudo journalctl -u mini-edr-server -f` |
| Restart server | `sudo systemctl restart mini-edr-server` |
| Check reverse proxy | `sudo systemctl status caddy --no-pager` |
| Follow proxy logs | `sudo journalctl -u caddy -f` |
| Backup database | `sudo cp /var/lib/mini-edr/mini_edr.db /var/backups/mini_edr_$(date +%F).db` |

After an update that changes Python dependencies:

```bash
cd /opt/mini-edr
sudo -u mini-edr .venv/bin/pip install -r requirements.txt
sudo systemctl restart mini-edr-server
```

## API endpoints

| Endpoint | Purpose |
|---|---|
| `GET /api/health` | Server health check |
| `POST /api/v1/agents/enroll` | One-time agent enrollment |
| `POST /api/v1/events` | Authenticated event batch ingestion |
| `GET /api/v1/agents` | Enrolled endpoint inventory |
| `GET /api/v1/alerts` | Alert stream |
| `PATCH /api/v1/alerts/{id}` | Update alert lifecycle status |

## Rule format

Rules are server-side files in `rules/*.yaml`. They are reviewed and committed to Git before deployment.

```yaml
- id: EDR-PS-001
  title: Encoded PowerShell command
  severity: high
  mitre: T1059.001
  description: PowerShell encoded commands can hide command content and deserve review.
  when:
    category: powershell
    command_line_contains: -enc
```

Restart the server after changing rules:

```bash
sudo systemctl restart mini-edr-server
```

## What comes next

The next coding phase adds persistent services and native event collectors:

1. Windows Event Log: process creation, authentication, and PowerShell events.
2. Linux journald/authentication log collection.
3. Local disk queue and retry behavior.
4. YARA rule-pack delivery and endpoint-side scanning.

The complete project plan is in [docs/SDLC.md](docs/SDLC.md).

## Safety rules

- Keep all VMs patched and isolated to your lab network.
- Never commit `.env`, `/etc/mini-edr/server.env`, certificates, database files, or enrollment tokens.
- Never disable TLS verification with `verify=False` or `curl -k`.
- Use only safe fixtures and simulations; do not download or run malware.
- Rotate the bootstrap enrollment token after each deployment wave in a future server update.

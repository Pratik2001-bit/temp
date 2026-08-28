# mini-EDR

mini-EDR is a small, rule-based endpoint detection and response **alerting** platform. Windows and Linux agents collect selected endpoint events, send them to a central server, and the server creates explainable alerts.

The first release intentionally focuses on this reliable path:

```text
Endpoint agent -> HTTPS API -> event validation -> detection rules -> alert dashboard
```

It does not automatically kill processes, quarantine files, or make AI decisions. An analyst reviews alerts before any response action is taken.

## Project status

Phase 1 is implemented: server foundation, enrollment, authenticated event ingestion, SQLite storage, deterministic YAML rules, alert API, a terminal-style dashboard, and a small agent transport skeleton. Native Windows and Linux collectors are the next phase.

## Features in this phase

- One-time agent enrollment using a bootstrap token.
- Per-agent random API token; only a SHA-256 token digest is stored by the server.
- Immutable agent ID rather than IP-based identity.
- JSON event batches with schema validation and event-size limits.
- Server-side, versioned YAML detection rules.
- Alert evidence, severity, MITRE references, and lifecycle status.
- Terminal-style dashboard for endpoints and alerts.
- A safe `--demo` agent mode for testing the full pipeline without malicious activity.

## Architecture

```text
mini-edr/
├── server/                 FastAPI server, database, API, detection engine, web UI
├── agent/                  Shared agent transport and future OS collectors
├── rules/                  Human-readable server-side detection rules
├── tests/                  API and detection tests
├── docs/                   SDLC plan and architecture decisions
└── data/                   Local SQLite database (created at runtime; not committed)
```

## Quick start (local development)

Prerequisite: Python 3.12 or newer.

```powershell
cd "C:\Users\pratik\Desktop\Cyber Security Project\mini-EDR"
py -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
```

Edit `.env` and set `EDR_BOOTSTRAP_TOKEN` to a long random value. Do not commit `.env`.

Start the server:

```powershell
$env:EDR_BOOTSTRAP_TOKEN = "your-long-random-secret"
uvicorn server.app:app --host 127.0.0.1 --port 8000 --reload
```

Open `http://127.0.0.1:8000` to view the dashboard. API documentation is available at `http://127.0.0.1:8000/docs`.

## Test an agent enrollment and alert

Keep the server running, then open a second terminal:

```powershell
.\.venv\Scripts\Activate.ps1
python -m agent.main --server http://127.0.0.1:8000 --enrollment-token "your-long-random-secret" --demo
```

The demo agent enrolls, sends a safe simulated encoded-PowerShell event, and creates a `high` severity test alert. This does **not** execute PowerShell or download anything.

## Run tests

```powershell
pytest
```

## Deployment approach

Do not expose the development command directly to the internet. For a small deployment:

1. Run the server on a dedicated Linux VM or trusted Windows server.
2. Put it behind a reverse proxy such as Caddy or Nginx with a real TLS certificate.
3. Set `EDR_BOOTSTRAP_TOKEN` through the service environment, never in source code.
4. Restrict inbound access to the dashboard and API to trusted administrator and endpoint networks.
5. Create a one-time enrollment token for each deployment wave; rotate it after use.
6. Back up the database and protect it as security-sensitive data.
7. Run the agent with least privilege. Windows event-log and Linux journal access may require a service account with specific permissions.

Before production deployment, complete the remaining SDLC phases in [docs/SDLC.md](docs/SDLC.md), especially TLS, token rotation, packages/services, test VMs, and retention controls.

## Event format

Agents send batches in this shape:

```json
{
  "events": [
    {
      "timestamp": "2026-08-29T12:00:00Z",
      "platform": "windows",
      "category": "powershell",
      "action": "script_block",
      "process_name": "powershell.exe",
      "command_line": "powershell.exe -enc <redacted>",
      "source": "Microsoft-Windows-PowerShell/Operational",
      "raw_event_id": "4104"
    }
  ]
}
```

Never place passwords, access tokens, full private keys, or arbitrary file contents in telemetry.

## Rule format

Rules live in `rules/*.yaml`. They use simple fields, which keeps detections reviewable. Example:

```yaml
id: EDR-PS-001
title: Encoded PowerShell command
severity: high
mitre: T1059.001
when:
  category: powershell
  command_line_contains: -enc
```

Rule matching occurs on the server. YARA will be added in the next detection phase: the server distributes a controlled rule pack and the agent reports matches; the server creates the alert.

## Security boundaries

- This is an alerting platform, not an antivirus product.
- Never run unknown samples or real malware to test it.
- Use safe test fixtures and test VMs only.
- Do not add automatic response actions without explicit approval, audit trails, and rollback design.
- Do not copy Wazuh source code. It is a reference for design concepts only.

## Roadmap

See [docs/SDLC.md](docs/SDLC.md) for the complete staged plan and acceptance criteria.


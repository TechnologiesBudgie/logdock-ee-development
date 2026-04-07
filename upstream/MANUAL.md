# LogDock v2.2 — User Guide

> Self-hosted log management · DataDog & GrayLog replacement · Your logs stay on your servers

---

## Table of Contents

1. [What is LogDock?](#1-what-is-logdock)
2. [Quick Start (5 minutes)](#2-quick-start-5-minutes)
3. [The Web Interface](#3-the-web-interface)
4. [Sending Logs to LogDock](#4-sending-logs-to-logdock)
5. [Production Setup](#5-production-setup)
6. [Alerts](#6-alerts)
7. [AI Triage](#7-ai-triage)
8. [User Management](#8-user-management)
9. [Single Sign-On (SSO)](#9-single-sign-on-sso)
10. [Storage, Compression & Retention](#10-storage-compression--retention)
11. [License](#11-license)
12. [Troubleshooting](#12-troubleshooting)
13. [Keyboard Shortcuts](#13-keyboard-shortcuts)

---

## 1. What is LogDock?

LogDock is a **self-hosted log management platform** — a drop-in replacement for DataDog Logs and GrayLog. You run it on your own servers, so your logs never leave your infrastructure.

Think of it as:

- A centralised place to collect logs from every app, server, and service
- A powerful search interface to find the needle in the haystack
- An alerting system that pings you before users notice problems
- An AI assistant that explains what went wrong and why

> **No cloud required.** LogDock runs entirely on your own hardware. Ideal for regulated industries, air-gapped environments, and teams who want full control.

### Key Capabilities

| Capability | What it means for you |
|---|---|
| Multi-protocol ingestion | JSON HTTP, syslog UDP/TCP, OpenTelemetry OTLP/HTTP JSON |
| Structured search | Filter by level, source, time range, or any custom field |
| Live tail | Watch logs stream in real-time — like a browser-based `tail -f` |
| AI triage | Paste an error and get an instant root cause explanation |
| Alerts | Get notified on Slack/webhook when errors spike |
| Tiered compression | Recent logs stay fast; old logs compress to save 40–75% disk space |
| Clustering | Spread logs across multiple nodes for high availability |
| Compliance reports | One-command SOC2, GDPR, HIPAA, PCI-DSS reports |
| SSO | OAuth2/OIDC, SAML 2.0, LDAP/Active Directory (Enterprise) |

---

## 2. Quick Start (5 minutes)

### Step 1 — Start the server

```bash
cd logdock
go run ./cmd/logdock

# You should see:
# logdock listening on :2514
```

Open **http://localhost:2514** and log in with `admin` / `admin`.

> Change the default password before exposing LogDock to a network. See [Production Setup](#5-production-setup).

### Step 2 — Send your first log

Get a token:

```bash
TOKEN=$(curl -s -X POST http://localhost:2514/api/v1/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"username":"admin","password":"admin"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
```

Send a log event:

```bash
curl -X POST http://localhost:2514/api/v1/ingest \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "level":   "error",
    "message": "payment gateway timeout",
    "source":  "api",
    "fields":  {"user_id": "42", "duration_ms": "3501"}
  }'
```

### Step 3 — Find it in the UI

1. Click **Log Explorer** in the left sidebar
2. Your log appears with a level badge, source, and message
3. Click the row to expand all fields

---

## 3. The Web Interface

| Section | What to use it for |
|---|---|
| **Dashboard** | Overview: total logs, error count, events/min, disk usage |
| **Log Explorer** | Search and browse all logs. The main workspace |
| **Live Tail** | Watch logs arrive in real-time |
| **Streams** | Route logs into named buckets (e.g. "Production Errors") |
| **Alerts** | Create threshold rules and get notified via Slack or webhook |
| **AI Triage** | Paste an error log, get an AI-written root cause explanation |
| **SIEM** | Security event overview with heuristic classification |
| **Pipeline** | Transform logs before storage: filter, enrich, redact PII, route |
| **Nodes** | Add/remove cluster peers, see storage backend status |
| **Users** | Create users, assign roles, manage MFA |
| **Settings** | Retention, compression, security, AI, SSO, API keys, notifications |
| **Audit Log** | Full history of admin actions with filtering |

### Log Explorer: View Modes

| Mode | Best for |
|---|---|
| Table | Scanning many logs quickly. Click a row to expand all fields |
| JSON | Scripting and exporting |

---

## 4. Sending Logs to LogDock

### Option A: JSON over HTTP (recommended for apps)

POST a JSON object to `/api/v1/ingest`. Requires a Bearer token.

```bash
curl -X POST http://localhost:2514/api/v1/ingest \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "timestamp": "2025-01-01T12:00:00Z",
    "source":    "my-api",
    "level":     "error",
    "message":   "payment gateway timeout",
    "fields": {
      "user_id":     "42",
      "duration_ms": "3501",
      "status_code": "504"
    }
  }'
```

| Field | Notes |
|---|---|
| `timestamp` | ISO 8601. Omit to use server time |
| `source` | App name, hostname, or service |
| `level` | `debug` / `info` / `warn` / `error` / `fatal` |
| `message` | The log line as a plain string |
| `fields` | Any extra key-value pairs — all indexed and searchable |

Request body is capped at 1 MB per record, 10 MB overall.

### Option B: Syslog (existing infrastructure)

LogDock listens on port **5140** for standard syslog.

```bash
logger -n 127.0.0.1 -P 5140 "my log message"
echo "<134>myapp: user login failed" | nc -u 127.0.0.1 5140
```

Forward from rsyslog:

```
*.* @@logdock.internal:5140   # TCP
*.* @logdock.internal:5140    # UDP
```

### Option C: OpenTelemetry (OTLP/HTTP JSON)

LogDock implements the OTLP/HTTP JSON protocol natively, with no additional dependencies.

| Protocol | Address |
|---|---|
| HTTP JSON | `localhost:4318/v1/logs` |
| gRPC | `localhost:4317` (port open; use HTTP JSON for full ingestion) |

Configure your exporter:

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/json
```

LogDock extracts `service.name` from resource attributes, maps OTLP severity text to log levels, and indexes all log record attributes as searchable fields.

### Option D: CLI

```bash
logdock ingest --message "Deploy complete" --level info --source ci

logdock ingest --endpoint http://logdock.internal:2514/api/v1/ingest \
               --message "Database timeout" --level error --source postgres
```

---

## 5. Production Setup

> Required before going live. The default credentials and dev keys must be changed before exposing LogDock to a network.

### Required environment variables

```bash
export LOGDOCK_JWT_SECRET=$(openssl rand -hex 32)
export LOGDOCK_MASTER_KEY=$(openssl rand -hex 32)
export LOGDOCK_ADMIN_PASSWORD=your-strong-password
export LOGDOCK_ENV=production
export LOGDOCK_DATA_DIR=/var/lib/logdock

./logdock serve
```

### All configuration options

| Variable | Default | Required? | Description |
|---|---|---|---|
| `LOGDOCK_JWT_SECRET` | `change-me-now` | **Yes** | JWT signing key |
| `LOGDOCK_MASTER_KEY` | dev fallback | **Yes** | AES-256-GCM key for encrypting stored secrets |
| `LOGDOCK_ADMIN_PASSWORD` | `admin` | **Yes** | Password for the admin account |
| `LOGDOCK_ENV` | — | **Yes** (prod) | Set to `production` to enforce `LOGDOCK_JWT_SECRET` |
| `LOGDOCK_DATA_DIR` | `./data` | No | Directory for log files, settings, users, and nodes |
| `LOGDOCK_HTTP_ADDR` | `:2514` | No | HTTP listen address |
| `LOGDOCK_USERS_PATH` | `$DATA_DIR/users.json` | No | Path to persisted user store |
| `LOGDOCK_NODES_PATH` | `$DATA_DIR/nodes.json` | No | Path to persisted node topology |
| `LOGDOCK_LICENSE_FILE` | `/etc/logdock/license.lic` | No | Path to RSA-signed license. Omit for Community edition. |
| `LOGDOCK_ENABLE_AI` | `false` | No | Enable AI triage |
| `LOGDOCK_STORAGE_ENGINE` | `fs` | No | `fs`, `lvm`, or `zfs` |
| `LOGDOCK_CLUSTER_NODES` | — | No | Comma-separated peer addresses |
| `LOGDOCK_TLS_CERT_FILE` | — | No | TLS certificate path |
| `LOGDOCK_TLS_KEY_FILE` | — | No | TLS private key path |

### systemd service

```ini
[Unit]
Description=LogDock Log Management
After=network.target

[Service]
Type=simple
User=logdock
EnvironmentFile=/etc/logdock/logdock.env
ExecStart=/usr/local/bin/logdock serve
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable --now logdock
sudo journalctl -fu logdock
```

### Docker

```bash
just docker-up
just docker-logs
```

---

## 6. Alerts

Rules are evaluated every 60 seconds.

### Creating an alert (UI)

1. Click **Alerts** in the sidebar
2. Click **New Alert Rule**
3. Enter a name and condition
4. Set the notification target (webhook or Slack)
5. Save — the rule activates immediately

### Rule condition syntax

```
level=ERROR > 10
level=WARN > 50
source=nginx > 100
source=postgres level=ERROR > 5
```

### Via API

```bash
curl -X POST http://localhost:2514/api/v1/alerts \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"name":"pg-errors","condition":"source=postgres level=ERROR > 5","target":"webhook"}'
```

Webhook and Slack URLs are encrypted with AES-256-GCM before storage.

---

## 7. AI Triage

### Supported providers

| Provider | When to use it |
|---|---|
| OpenAI | Best results. Needs an OpenAI API key |
| Anthropic Claude | Strong reasoning. Needs an Anthropic API key |
| Ollama | Fully private — no data leaves your servers |
| Custom | Any OpenAI-compatible proxy or endpoint |

### Setup via environment variables

```bash
# OpenAI
LOGDOCK_ENABLE_AI=true
LOGDOCK_AI_PROVIDER=openai
LOGDOCK_AI_API_KEY=sk-proj-...
LOGDOCK_AI_MODEL=gpt-4o-mini

# Anthropic Claude
LOGDOCK_AI_PROVIDER=anthropic
LOGDOCK_AI_API_KEY=sk-ant-...
LOGDOCK_AI_MODEL=claude-haiku-4-5-20251001

# Ollama (no key needed)
LOGDOCK_AI_PROVIDER=ollama
LOGDOCK_AI_ENDPOINT=http://localhost:11434
LOGDOCK_AI_MODEL=llama3.2:3b
```

API keys are encrypted with AES-256-GCM before storage, never logged, and redacted in all API responses.

### Via CLI

```bash
LOGDOCK_AI_API_KEY=sk-... logdock triage \
  --provider openai \
  --message "FATAL: OOM killer terminated java process pid=1234"
```

---

## 8. User Management

### Roles

| Role | What they can do |
|---|---|
| `admin` | Full control: users, settings, alerts, logs, clustering |
| `operator` | Read/write logs, manage alerts and nodes. Cannot manage users or settings |
| `viewer` | Read-only. Can search and view logs only |

Users are persisted to `users.json` and survive restarts.

### Creating users (UI)

Go to **Users** in the sidebar, click **New User**, fill in username, password, and role.

### Creating users (API)

```bash
curl -X POST http://localhost:2514/api/v1/users/create \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","password":"secure-pass","role":"operator"}'
```

### API Keys

1. Go to **Settings → API Keys**
2. Enter a name and scopes, then click **Generate Key**
3. Copy the key immediately — it is shown only once
4. Use it as a Bearer token: `Authorization: Bearer ld_...`

Available scopes: `ingest`, `read`, `admin`, `watcher`.

### MFA

Enable TOTP-based MFA per user in **Settings → Security**. Scan the QR code with any RFC 6238 authenticator app (Google Authenticator, Authy, Aegis, etc.). Users enter a 6-digit code at login.

---

## 9. Single Sign-On (SSO)

SSO is an Enterprise feature. It allows users to authenticate via your existing identity provider instead of managing passwords in LogDock.

Three protocols are supported: OAuth2/OIDC, SAML 2.0, and LDAP/Active Directory.

When a user authenticates via SSO for the first time, a LogDock account is created automatically with the role mapped from the IdP. The role is updated on every subsequent login.

### Configure via API

```bash
# OAuth2 / OIDC (e.g. Okta, Azure AD, Google, Keycloak)
curl -X PUT http://localhost:2514/api/v1/sso/config \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "protocol": "oauth2",
    "client_id": "logdock",
    "client_secret": "...",
    "authorization_endpoint": "https://your-idp/oauth2/authorize",
    "token_endpoint": "https://your-idp/oauth2/token",
    "userinfo_endpoint": "https://your-idp/oauth2/userinfo",
    "scopes": ["openid", "email", "profile"],
    "role_mapping": {"admin": "admin", "sre": "operator", "dev": "viewer"}
  }'
```

```bash
# LDAP / Active Directory
curl -X PUT http://localhost:2514/api/v1/sso/config \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{
    "protocol": "ldap",
    "ldap_host": "ldap.corp.example.com",
    "ldap_port": 636,
    "ldap_use_tls": true,
    "ldap_bind_dn": "cn=logdock,ou=service-accounts,dc=corp,dc=example,dc=com",
    "ldap_bind_password": "...",
    "ldap_search_base": "ou=users,dc=corp,dc=example,dc=com",
    "ldap_user_filter": "(&(objectClass=person)(sAMAccountName={username}))",
    "ldap_group_attribute": "memberOf",
    "ldap_role_mapping": {
      "CN=LogDock-Admins,OU=Groups,DC=corp,DC=example,DC=com": "admin",
      "CN=LogDock-SRE,OU=Groups,DC=corp,DC=example,DC=com": "operator"
    }
  }'
```

### SSO endpoints

| Endpoint | Description |
|---|---|
| `GET /api/v1/sso/oauth2/login` | Initiates OAuth2 authorization code + PKCE flow |
| `GET /api/v1/sso/oauth2/callback` | OAuth2 callback — handled automatically by the browser |
| `GET /api/v1/sso/saml/login` | Initiates SAML SP-initiated flow |
| `POST /api/v1/sso/saml/acs` | SAML Assertion Consumer Service endpoint |
| `POST /api/v1/sso/ldap/login` | LDAP username/password login |

The SAML ACS URL to register with your IdP is: `https://your-logdock/api/v1/sso/saml/acs`

---

## 10. Storage, Compression & Retention

LogDock automatically compresses old log partitions in the background.

### Compression tiers

| Tier | Age | Algorithm | Saving |
|---|---|---|---|
| Hot | 0–7 days | None | — |
| Warm | 7–30 days | LZ4 | ~40% |
| Cold | 30–180 days | Zstd-3 | ~60% |
| Archive | 180+ days | Zstd-11 | ~75% |

Adjust tier boundaries in **Settings → Retention**.

### Storage backends

| Backend | Set via | Best for |
|---|---|---|
| Filesystem (default) | `LOGDOCK_STORAGE_ENGINE=fs` | Any OS, easiest setup |
| LVM | `LOGDOCK_STORAGE_ENGINE=lvm` | Enterprise servers needing flexible volume resizing |
| ZFS | `LOGDOCK_STORAGE_ENGINE=zfs` | Data integrity, built-in snapshots, native compression |

> ZFS tip: Disable LogDock's application-level compression when using ZFS — ZFS handles it more efficiently. Set `WarmDays=0` in Settings → Retention.

### Clustering

```bash
# On node1:
LOGDOCK_CLUSTER_NODES=node2:2514,node3:2514

# On node2:
LOGDOCK_CLUSTER_NODES=node1:2514,node3:2514
```

Each node lists its peers, not itself. Node topology is persisted to `nodes.json` and survives restarts. Nodes can be added or removed via `POST`/`DELETE /api/v1/nodes` without restarting.

---

## 11. License

LogDock verifies your license offline at startup. No network call is required.

### Editions

| Edition | Features |
|---|---|
| Community | Core log ingestion, search, alerts, pipelines, AI triage |
| Pro | Community + encryption at rest, backup |
| Enterprise | Pro + SSO, SIEM, multi-node clustering, replication, failover, RBAC dashboards, plugins |
| Air-gapped | Enterprise + no telemetry, no update checks |

Community edition requires no license file. Place the signed `.lic` file at `LOGDOCK_LICENSE_FILE` (default: `/etc/logdock/license.lic`) to activate Pro or Enterprise.

### License status

```bash
# Via API
curl http://localhost:2514/api/v1/license \
  -H "Authorization: Bearer $TOKEN"

# Via CLI
logdock version
```

Expired licenses remain operational for 30 days (grace period) while a renewal warning is displayed in the UI and logs.

---

## 12. Troubleshooting

### Server won't start

```bash
sudo ss -tlnp | grep 2514
journalctl -u logdock -n 50
env | grep LOGDOCK
```

### Security warnings on startup

```
SECURITY WARNING: LogDock is using the default admin password ('admin').
```

Set `LOGDOCK_ADMIN_PASSWORD` before starting the server.

```
WARNING: LOGDOCK_JWT_SECRET is using the insecure default.
```

```bash
export LOGDOCK_JWT_SECRET=$(openssl rand -hex 32)
export LOGDOCK_ENV=production
```

### Can't see logs after sending them

- Confirm the ingest request returned HTTP 202
- Verify the Bearer token in the `Authorization` header is valid (not expired, not revoked)
- Check the time range in Log Explorer covers when the logs were sent
- Click **Clear filters** in the explorer toolbar

### MFA codes rejected

Ensure your authenticator app's clock is accurate. LogDock accepts codes from ±1 time step (±30 seconds) to account for clock drift. If codes are consistently rejected after re-enrollment, check that the server time is correct (`timedatectl status`).

### OTLP logs not appearing

LogDock accepts OTLP/HTTP JSON. If you are using a gRPC exporter, switch to the HTTP JSON protocol:

```bash
OTEL_EXPORTER_OTLP_PROTOCOL=http/json
OTEL_EXPORTER_OTLP_ENDPOINT=http://logdock:4318
```

### SSO login fails

Check the server logs for detail. Common causes: wrong `authorization_endpoint`, mismatched `client_id`, PKCE not supported by the IdP (use SAML or LDAP instead), or a role mapping that produces an empty role.

### High disk usage

```bash
logdock storage health --json
logdock compression --json
```

Tighten retention in **Settings → Retention**:

```bash
LOGDOCK_RETENTION_MAX_DAYS=90
LOGDOCK_RETENTION_MAX_DISK_GB=50
```

### AI triage not working

```bash
# Test Ollama
curl http://localhost:11434/api/tags

# Test OpenAI key
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $LOGDOCK_AI_API_KEY"
```

---

## 13. Keyboard Shortcuts

| Key | Action |
|---|---|
| `/` | Focus search bar |
| `L` | Toggle live tail |
| `R` | Refresh active panel |
| `P` | Pause / resume live tail |
| `C` | Clear live tail buffer |
| `Esc` | Close modal or detail view |
| `T` | AI triage selected log |
| `G D/E/A/S/N/K` | Navigate to Dashboard / Explorer / Alerts / Settings / Nodes / API Keys |
| `Ctrl/Cmd + K` | Open command palette |
| `Ctrl E` | Export CSV |
| `?` | Show shortcuts reference |
| `1`–`9` | Switch to panel by index |

Single-key shortcuts are disabled while an input field or textarea has focus.

---

*LogDock v2.2 · Enterprise Log Management · Technologies Budgie*

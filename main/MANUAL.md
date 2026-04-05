# LogDock v2.0 — User Guide

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
9. [Storage, Compression & Retention](#9-storage-compression--retention)
10. [Troubleshooting](#10-troubleshooting)
11. [Keyboard Shortcuts](#11-keyboard-shortcuts)

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
| Multi-protocol ingestion | Send logs from any source: JSON HTTP, syslog (UDP/TCP), or OpenTelemetry (OTLP) |
| Structured search | Filter by level, source, time range, or any custom field |
| Live tail | Watch logs stream in real-time — like a browser-based `tail -f` |
| AI triage | Paste an error and get an instant root cause explanation |
| Alerts | Get notified on Slack/webhook when errors spike |
| Tiered compression | Recent logs stay fast; old logs compress to save 40–75% disk space |
| Clustering | Spread logs across multiple nodes for high availability |
| Compliance reports | One-command SOC2, GDPR, HIPAA, PCI-DSS reports |

---

## 2. Quick Start (5 minutes)

### Step 1 — Start the server

```bash
# From the extracted zip:
cd logdock
go run ./cmd/logdock

# You should see:
# listening on :2514
```

Open **http://localhost:2514** and log in with `admin` / `admin`.

> ⚠️ Change the default password before exposing LogDock to a network. See [Production Setup](#5-production-setup).

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
4. Switch to **Pretty** view (grid icon) for syntax-highlighted JSON

---

## 3. The Web Interface

| Section | What to use it for |
|---|---|
| **Dashboard** | Overview: total logs, error count, events/min, disk usage |
| **Log Explorer** | Search and browse all logs. The main workspace |
| **Alerts** | Create threshold rules and get notified via Slack or webhook |
| **AI Triage** | Paste an error log, get an AI-written root cause explanation |
| **Pipelines** | Transform logs before storage: extract fields, mask PII, drop noise |
| **Streams** | Route logs into named buckets (e.g. "Production Errors") |
| **Nodes** | Add/remove cluster peers, see storage backend status |
| **Users** | Create users and assign roles |
| **Settings** | Retention, compression, security, API keys, notifications |

### Log Explorer: View Modes

| Mode | Icon | Best for |
|---|---|---|
| Table | ☰ | Scanning many logs quickly. Click a row to expand all fields |
| Pretty | ⊞ | Debugging. Syntax-highlighted JSON, clickable field pills that auto-filter |
| Raw JSON | `<>` | Scripting and exporting |

Toggle between them using the button group in the top-right of the Explorer toolbar.

### Searching Logs

Type in the search bar and press **Enter**:

```
# Free text
payment failed

# Field filter
level:error
source:api-gateway
user_id:42

# Combined
level:error source:postgres connection refused
```

Use **Save** (bookmark icon) to save frequent searches and recall them with one click.

---

## 4. Sending Logs to LogDock

Choose whichever method fits your stack. Most teams use JSON HTTP for new apps and syslog for existing infrastructure.

### Option A: JSON over HTTP *(recommended for apps)*

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
| `source` | App name, hostname, or service. Used for filtering |
| `level` | `debug` / `info` / `warn` / `error` / `fatal` |
| `message` | The log line as a plain string |
| `fields` | Any extra key-value pairs — all indexed and searchable |

### Option B: Syslog *(existing infrastructure)*

LogDock listens on port **5140** for standard syslog. No code changes needed for existing apps.

```bash
# Quick test from command line:
logger -n 127.0.0.1 -P 5140 "my log message"

# Or with netcat (UDP):
echo "<134>myapp: user login failed" | nc -u 127.0.0.1 5140
```

Forward from rsyslog:

```
# /etc/rsyslog.d/logdock.conf
*.* @@logdock.internal:5140   # TCP
*.* @logdock.internal:5140    # UDP
```

### Option C: OpenTelemetry (OTLP)

Point your OTEL collector at LogDock:

| Protocol | Address |
|---|---|
| gRPC | `localhost:4317` |
| HTTP | `localhost:4318/v1/logs` |

### Option D: CLI

```bash
# Send a single log event
logdock ingest --message "Deploy complete" --level info --source ci

# Send to a remote server
logdock ingest --endpoint http://logdock.internal:2514/api/v1/ingest \
               --message "Database timeout" --level error --source postgres
```

---

## 5. Production Setup

> ⚠️ **Required before going live.** The default credentials and dev keys must be changed before exposing LogDock to a network.

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
| `LOGDOCK_DATA_DIR` | `./data` | No | Directory where log files are stored |
| `LOGDOCK_HTTP_ADDR` | `:2514` | No | HTTP listen address |
| `LOGDOCK_ENABLE_AI` | `false` | No | Enable AI triage features |
| `LOGDOCK_STORAGE_ENGINE` | `fs` | No | `fs`, `lvm`, or `zfs` |
| `LOGDOCK_CLUSTER_NODES` | — | No | Comma-separated peer addresses for clustering |
| `LOGDOCK_TLS_CERT_FILE` | — | No | Path to TLS certificate |
| `LOGDOCK_TLS_KEY_FILE` | — | No | Path to TLS private key |
| `LOGDOCK_RETENTION_MAX_DAYS` | `365` | No | Delete partitions older than N days |
| `LOGDOCK_RETENTION_MAX_DISK_GB` | — | No | Delete oldest partitions when disk exceeds N GB |

### Running as a systemd service

```ini
# /etc/systemd/system/logdock.service
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
# docker-compose.yml is included in the repo
just docker-up
just docker-logs
```

---

## 6. Alerts

Alerts fire when a rule condition is met. Rules are evaluated every **60 seconds**.

### Creating an alert (UI)

1. Click **Alerts** in the sidebar
2. Click **New Alert Rule**
3. Enter a name and condition
4. Set the notification target (webhook or Slack)
5. Save — the rule activates immediately

### Rule condition syntax

```
level=ERROR > 10                    # >10 errors in the last 5 minutes
level=WARN > 50                     # >50 warnings
source=nginx > 100                  # >100 events from nginx
source=postgres level=ERROR > 5     # postgres errors specifically
```

### Notification channels

Configure webhook and Slack URLs in **Settings → Notifications**. URLs are encrypted with AES-256-GCM — never stored in plaintext.

### Via API

```bash
curl -X POST http://localhost:2514/api/v1/alerts \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"name":"pg-errors","condition":"source=postgres level=ERROR > 5","target":"webhook"}'
```

---

## 7. AI Triage

AI Triage lets you paste any log snippet and get an instant explanation of what went wrong and how to fix it.

### Supported providers

| Provider | When to use it |
|---|---|
| OpenAI (GPT-4o) | Best results. Needs an OpenAI API key |
| Anthropic Claude | Strong reasoning. Needs an Anthropic API key |
| Ollama (local) | Fully private — no data leaves your servers. Slower on CPU |
| Custom | Use any OpenAI-compatible proxy or service. |

### Setup via UI

1. Go to **AI Triage** in the sidebar
2. Select a provider and enter your API key (encrypted immediately on save)
3. Enter a model name (e.g. `gpt-4o-mini` or `llama3.2:3b` for Ollama)
4. Save, then use the triage form

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

# Ollama (no key needed — runs locally)
LOGDOCK_AI_PROVIDER=ollama
LOGDOCK_AI_ENDPOINT=http://localhost:11434
LOGDOCK_AI_MODEL=llama3.2:3b
```

> 🔒 API keys are encrypted with AES-256-GCM before storage. They are never written to disk in plaintext, never logged, and redacted in all API responses.

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

### Creating users (UI)

Go to **Users** in the sidebar → click **New User**, fill in username, password, and role.

### Creating users (API)

```bash
curl -X POST http://localhost:2514/api/v1/users \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"username":"alice","role":"operator","mfa":true}'
```

### API Keys

API keys let scripts and CI/CD pipelines send or query logs without a user password.

1. Go to **Settings → API Keys**
2. Enter a name and role, then click **Generate Key**
3. Copy the key immediately — it's shown only once
4. Use it as a Bearer token: `Authorization: Bearer logdock_....`

### MFA

Enable TOTP-based MFA per-user in **Settings → Security**. Users must enter a 6-digit code from an authenticator app at login.

---

## 9. Storage, Compression & Retention

LogDock automatically moves old log partitions to more aggressive compression as they age. Nothing to configure — it happens in the background.

### Compression tiers

| Tier | Age | Algorithm | Typical saving |
|---|---|---|---|
| Hot | 0–7 days | None (fast reads) | — |
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

> **ZFS tip:** Disable LogDock's application-level compression when using ZFS (`WarmDays=0` in Settings → Retention) — ZFS handles it more efficiently.

### Clustering

Spread load and storage across multiple nodes:

```bash
# On node1:
LOGDOCK_CLUSTER_NODES=node2:2514,node3:2514

# On node2:
LOGDOCK_CLUSTER_NODES=node1:2514,node3:2514
```

Each node lists its *peers* (not itself). LogDock uses consistent hashing to shard writes across nodes.

---

## 10. Troubleshooting

### Server won't start

```bash
# Check for port conflicts:
sudo ss -tlnp | grep 2514

# Check service logs:
journalctl -u logdock -n 50

# Verify env vars are set:
env | grep LOGDOCK
```

### "change-me-now" warning on startup

You haven't set `LOGDOCK_JWT_SECRET`. Fix it:

```bash
export LOGDOCK_JWT_SECRET=$(openssl rand -hex 32)
export LOGDOCK_ENV=production
```

### Can't see logs after sending them

- Check the ingest request returned HTTP 200
- Make sure you're using a valid Bearer token in the `Authorization` header
- Verify the time range in Log Explorer covers when you sent the logs
- Click **Clear filters** in the explorer toolbar

### AI triage not working

```bash
# Test Ollama is running:
curl http://localhost:11434/api/tags

# Test OpenAI key is valid:
curl https://api.openai.com/v1/models \
  -H "Authorization: Bearer $LOGDOCK_AI_API_KEY"
```

### High disk usage

```bash
logdock storage health --json
logdock compression --json
```

Tighten retention in **Settings → Retention** or via env:

```bash
LOGDOCK_RETENTION_MAX_DAYS=90
LOGDOCK_RETENTION_MAX_DISK_GB=50
```

---

*LogDock v2.0 · Enterprise Log Management · [Source on Codeberg](https://codeberg.org/yourorg/logdock)*

## 11. Keyboard Shortcuts

For faster workflows, LogDock includes several global keyboard shortcuts:

| Key | Action |
|---|---|
| **1 – 6** | Switch between main tabs (Dashboard, Explorer, Alerts, AI, Nodes, Settings) |
| **Ctrl / Cmd + K** | Focus the global search bar (instantly navigates to Log Explorer) |
| **R** | Refresh the current page data |
| **Esc** | Close any open modal or detail view |

> 💡 **Note:** Single-key shortcuts (like 1-6 and R) are automatically disabled when you are typing inside an input field or textarea.

# LogDock — Enterprise Log Platform

LogDock is a self-hosted log observability platform shipped as a **single Go binary** with embedded admin UI and full CLI parity. No external dependencies at runtime.

## What's New in v2.2

### Persistence & Reliability
- **User persistence** — Accounts created via API survive restarts. Written atomically to `users.json` in `LOGDOCK_DATA_DIR`. Path: `LOGDOCK_USERS_PATH`.
- **Node topology persistence** — Cluster peers added via API survive restarts. Written atomically to `nodes.json`. Path: `LOGDOCK_NODES_PATH`.
- **Node deduplication** — `POST /api/v1/nodes` now performs upsert by ID, preventing duplicate entries.
- **Node removal** — `DELETE /api/v1/nodes?id=` removes a peer from the topology.

### License System
- **Offline RSA-4096 license verification** — No network call required. License file read on startup and reloaded every 24 hours without restart.
- **Edition tiers** — Community (free), Pro (single-node commercial), Enterprise (multi-node, SSO, SIEM, replication), Air-gapped (Enterprise with no telemetry).
- **Feature gating** — Enterprise features return a descriptive error with upgrade guidance when the license does not permit them.
- **Node limit enforcement** — Adding nodes beyond the licensed count returns HTTP 402.
- **Grace period** — Expired licenses remain operational for 30 days while a renewal warning is displayed.
- **License endpoint** — `GET /api/v1/license` returns edition summary and days until expiry.

### SSO — Single Sign-On
- **OAuth2 / OIDC** — Authorization code flow with PKCE. Compatible with Okta, Azure AD, Google, Keycloak, and any OIDC-compliant IdP.
- **SAML 2.0** — SP-initiated SSO. Compatible with Okta, ADFS, OneLogin, Azure AD (SAML mode).
- **LDAP / Active Directory** — Simple bind + group search with role mapping.
- **Auto-provisioning** — Users are created locally on first SSO login with their IdP-mapped role. Role updated on each subsequent login.
- **Secret handling** — `client_secret` and LDAP bind password stored in the AES-256-GCM credential store, never in settings JSON.
- **SSO config API** — `GET|PUT /api/v1/sso/config` for runtime reconfiguration without restart.

### OTLP Ingestion
- **OTLP/HTTP JSON** — Full implementation of the OTLP JSON protocol. Parses `resourceLogs`, extracts `service.name`, severity, nanosecond timestamps, and arbitrary attributes. Zero additional dependencies. Compatible with OpenTelemetry Collector, Grafana Alloy, Vector, and all OTLP SDK exporters (set `OTEL_EXPORTER_OTLP_PROTOCOL=http/json`).
- **OTLP/gRPC** — Port 4317 now sends HTTP/2 GOAWAY on connection instead of silently closing, so agents see a clear error and activate fallback transport.

### Security Hardening
- **Token-in-URL removed** — Tokens are no longer accepted via `?token=` query parameter on any endpoint. Only `Authorization: Bearer` and session cookie are valid.
- **Global request body cap** — All endpoints capped at 10 MB via middleware. Login: 4 KB. Triage: 64 KB. User create, API key: 4 KB. Pipeline rules: 64 KB.
- **TOTP SHA-1 fix** — TOTP now uses HMAC-SHA1 per RFC 4226. Previously used HMAC-SHA256, which is incompatible with Google Authenticator, Authy, and Aegis.
- **Admin password warning** — Prominent stderr banner when `LOGDOCK_ADMIN_PASSWORD` is unset or left as the default.

---

## What's in v2.1

### Enterprise Security
- **Real TOTP MFA** — RFC 6238 authenticator app support (otpauth:// QR provisioning)
- **JWT revocation** — Logout invalidates tokens server-side via JTI blocklist
- **Account lockout** — Configurable max failed attempts + lockout duration per user
- **AES-256-GCM credential encryption** — API keys, webhook URLs, AI keys never stored plaintext
- **Scoped API keys** — `ingest`, `read`, `admin`, `watcher` scopes with expiry; sha256 hashed
- **IP allowlist** — Restrict console access to trusted CIDR ranges
- **Brute-force guard** — Sliding-window IP rate limiting (240/min) with stale-entry cleanup

### Flagship UI Features
- **Log detail slide-over drawer** — Full detail: message, extracted fields, raw JSON, anomaly score
- **Real SSE live tail** — True server-sent events; falls back to polling on disconnect
- **Saved searches** — Named queries with level/source filters
- **Anomaly score overlay** — EWMA-based source scoring as pip indicators per row
- **API Keys page** — Full CRUD: create, view, revoke scoped keys
- **Audit Log page** — Filterable event table: logins, key create, exports, user management
- **Keyboard shortcuts** — `/` search, `L` live, `R` refresh, `G+key` nav, `T` triage, `Esc` close
- **Light/dark mode toggle** — Persistent preference
- **CSV + NDJSON export** — Direct download from explorer toolbar
- **User management** — Create, delete, unlock from Settings > Users
- **Webhook test** — Fire test alert from Notifications settings

### Observability
- **Anomaly detection** — EWMA per source; overview card on dashboard
- **Log volume histogram** — Click bar to jump to time window
- **Top sources chart** — Ingestion breakdown by source
- **Security overview** — Dashboard card with top anomalous sources

---

## Build

```bash
go mod tidy && go build -o logdock ./cmd/logdock
```

## Run

```bash
cp resources/logdock.env.example .env
set -a; source .env; set +a
./logdock serve
```

Access at `http://localhost:2514`. Default: `admin / admin`.

> Change `LOGDOCK_ADMIN_PASSWORD` before exposing this to a network.

## Docker

```bash
docker build -t logdock:latest .
docker run --rm -p 2514:2514 -p 4317:4317 -p 5140:5140/tcp -p 5140:5140/udp \
  -v $PWD/data:/data \
  -e LOGDOCK_DATA_DIR=/data \
  -e LOGDOCK_MASTER_KEY=change-this \
  -e LOGDOCK_ADMIN_PASSWORD=change-this \
  logdock:latest
```

---

## Security Matrix

| Control | Status |
|---|---|
| JWT HS256 + JTI revocation | Yes |
| AES-256-GCM credential store | Yes |
| TOTP MFA (RFC 6238, HMAC-SHA1) | Yes |
| Account lockout | Yes |
| IP rate limiting (sliding window) | Yes |
| Scoped API keys (sha256 hashed) | Yes |
| HTTP security headers (CSP / DENY) | Yes |
| Audit log (all admin actions) | Yes |
| RBAC (admin / operator / viewer) | Yes |
| Global 10 MB request body cap | Yes |
| Login-gated app HTML | Yes |
| Token-in-URL removed | Yes |
| Offline RSA-4096 license verification | Yes |
| SSO: OAuth2 PKCE / SAML 2.0 / LDAP | Yes |
| User persistence (atomic write) | Yes |
| Node persistence (atomic write) | Yes |

---

## Keyboard Shortcuts

| Key | Action |
|---|---|
| `/` | Focus search |
| `L` | Toggle live tail |
| `R` | Refresh page |
| `Esc` | Close drawer / modal |
| `T` | AI triage selected log |
| `G D/E/A/S/N/K` | Navigate to panel |
| `Ctrl E` | Export CSV |
| `?` | Show shortcuts |

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `LOGDOCK_MASTER_KEY` | dev fallback | AES-256-GCM key for credential encryption. Required in production. |
| `LOGDOCK_JWT_SECRET` | `change-me-now` | JWT signing secret. Required in production. |
| `LOGDOCK_ADMIN_PASSWORD` | `admin` | Password for the built-in admin account. Change before exposing to a network. |
| `LOGDOCK_DATA_DIR` | `./data` | Storage root for log partitions, settings, users, and nodes. |
| `LOGDOCK_HTTP_ADDR` | `:2514` | HTTP listen address. |
| `LOGDOCK_SYSLOG_TCP_ADDR` | `:5140` | Syslog TCP listen address. |
| `LOGDOCK_SYSLOG_UDP_ADDR` | `:5140` | Syslog UDP listen address. |
| `LOGDOCK_OTLP_GRPC_ADDR` | `:4317` | OTLP gRPC listen address. |
| `LOGDOCK_OTLP_HTTP_ADDR` | `:4318` | OTLP HTTP listen address. |
| `LOGDOCK_USERS_PATH` | `$DATA_DIR/users.json` | Path to the persisted user store. |
| `LOGDOCK_NODES_PATH` | `$DATA_DIR/nodes.json` | Path to the persisted node topology. |
| `LOGDOCK_LICENSE_FILE` | `/etc/logdock/license.lic` | Path to the RSA-signed license file. Omit for Community edition. |
| `LOGDOCK_ENV` | — | Set to `production` to make `LOGDOCK_JWT_SECRET` required at startup. |
| `LOGDOCK_ENABLE_AI` | `false` | Enable AI triage features. |
| `LOGDOCK_AI_PROVIDER` | `openai` | AI provider: `openai`, `anthropic`, `ollama`, `custom`. |
| `LOGDOCK_AI_API_KEY` | — | API key for the selected AI provider. Encrypted on first write. |
| `LOGDOCK_AI_MODEL` | — | Model name (e.g. `gpt-4o-mini`, `claude-haiku-4-5-20251001`, `llama3.2:3b`). |
| `LOGDOCK_AI_ENDPOINT` | — | Custom endpoint for Ollama or OpenAI-compatible proxies. |
| `LOGDOCK_STORAGE_ENGINE` | `fs` | Storage backend: `fs`, `lvm`, or `zfs`. |
| `LOGDOCK_CLUSTER_NODES` | — | Comma-separated peer addresses for clustering (e.g. `node2:2514,node3:2514`). |
| `LOGDOCK_TLS_CERT_FILE` | — | Path to TLS certificate. |
| `LOGDOCK_TLS_KEY_FILE` | — | Path to TLS private key. |

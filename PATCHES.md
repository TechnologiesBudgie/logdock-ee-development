# LogDock Patch Notes

## Patch 1 — Heal/Restart Loop (binary vs source mismatch)

### Symptom
```
2026/03/13 23:33:01 replication: started with 4 workers
2026/03/13 23:34:31 heal: target http marked degraded
2026/03/13 23:40:01 heal: http degraded for 5m29.993317227s — triggering restart
```
This loop repeats indefinitely.

### Root Cause
The `./logdock` **pre-compiled binary** in the repo root contains an older
health-check/self-healing subsystem that is **not present in the current
source code**. It probes an HTTP target every 30 s, marks it degraded after
5 failures (~2.5 min), and triggers a restart after ~5.5 min — then repeats.

The source under `cmd/logdock/` and `internal/` does **not** have this code.

### Fix
**Rebuild from source** — do not use the bundled binary:

```bash
go build -o logdock ./cmd/logdock
./logdock serve
```

The heal loop will be gone entirely from the new build.

**If you must keep the old binary temporarily**, check for a configured
cluster node or webhook URL that is unreachable:

```bash
echo $LOGDOCK_CLUSTER_NODES        # unset or empty = no peer probing
# Also check your .env / logdock.env for any http:// targets
```

Clearing `LOGDOCK_CLUSTER_NODES` (or any webhook URL pointing at a dead
host) will stop the probing while you're still on the old binary.

---

## Patch 2 — Missing i18n Keys for CLI-Exposed Features

### Files changed
- `web/i18n.js`
- `internal/api/webdist/i18n.js`

### What was missing
Five CLI commands exposed in `cmd/logdock/main.go` had no corresponding
`data-i18n` keys in either language:

| CLI command     | Keys added (EN + FR)                                              |
|-----------------|-------------------------------------------------------------------|
| `tui`           | `nav_tui`, `tui_desc`                                             |
| `snapshots`     | `nav_snapshots`, `snapshots_desc`, `snapshots_btn_create`, `snapshots_path` |
| `updates`       | `nav_updates`, `updates_desc`, `updates_btn_apply`, `updates_path` |
| `health`        | `nav_health`, `health_desc`, `health_btn_check`, `health_status_ok`, `health_status_unreachable` |
| `version`       | `nav_version`, `version_label`, `version_build`, `version_current` |

Both `web/i18n.js` (source) and `internal/api/webdist/i18n.js` (served)
were patched identically — they were already in sync before this patch.

### Usage
Wire any new UI elements with `data-i18n="<key>"` as usual; the translation
engine will pick them up automatically via `applyTranslations()`.

---

## Patch 3 — Build Fix: `filepath` Undefined in `server.go`

### Symptom
```
go build -ldflags="-s -w" -o logdock ./cmd/logdock
# logdock/internal/api
internal/api/server.go:1072:27: undefined: filepath
```

### Root Cause
`internal/api/server.go` used `filepath.Join` and `filepath.FromSlash` in the
`readUIFile` helper but only imported `"path"` (URL path package), not
`"path/filepath"` (OS-aware file path package).

### Fix
Added `"path/filepath"` to the import block in `internal/api/server.go`.

**Files changed:** `internal/api/server.go`

---

## Patch 4 — Keybind System Overhaul

### Changes
- Removed the `Shift+6` (`^`) trigger for the keybinds menu.
- Pressing `k` (when not in an input field) now opens the **Keyboard Shortcuts
  modal**.
- Pressing `#` arms a 1.5-second "hash-nav mode" without typing the `#`
  character. While armed, pressing `1`–`9` navigates the sidebar sections
  (Dashboard → Explorer → Alerts → AI → Nodes → Sources → Compliance →
  API Keys → Settings). The `#` keystroke itself is swallowed — it never
  appears in any input field.
- Legacy plain-digit shortcuts `1`–`8` remain functional when `#` has not been
  pressed.

**Files changed:** `web/index.html`

---

## Patch 5 — Log Sources Page (WebUI Parity)

### Problem
The backend exposed `/api/v1/settings/log-sources` (GET + POST) but the WebUI
had no corresponding page. The CLI `log-sources` command also had no
WebUI counterpart.

### Fix
Added a full **Log Sources** page to the WebUI:
- Sidebar nav entry (`#6` shortcut).
- Endpoint capability cards showing all supported ingest methods (HTTP, Syslog,
  OTLP, Kafka, Journald, Kubernetes, Windows Event Log, AWS S3, etc.).
- Source rules table with per-rule enable/disable and retention override.
- **Add Source Rule** modal (pattern, description, retention days, enabled
  toggle) with live save via `POST /api/v1/settings/log-sources`.
- Delete individual rules without page reload.

**Files changed:** `web/index.html`, `internal/i18n/*.json`,
`internal/api/i18ndata/*.json`

---

## Patch 6 — Docker / Podman / Kubernetes Deployment Fixes

### Docker Compose
- Removed obsolete top-level `version:` key (deprecated in Compose v2+).
- Added `LOGDOCK_MASTER_KEY` environment variable (required for AES-256-GCM
  credential encryption — previously missing, causing silent key-loss on
  restart).
- Added all optional env vars with `${VAR:-default}` syntax so a `.env` file
  is sufficient; no need to edit `docker-compose.yml`.
- Added `healthcheck` using `wget` (available in alpine scratch images via
  the build stage; runtime uses a shell-less probe).
- Added `tmpfs: [/tmp]` for scratch-image compatibility.
- Switched data volume from bind-mount `./data` to a named volume
  `logdock-data` for better Docker lifecycle management.
- Added commented-out sidecar agent example.

### Dockerfile / Containerfile
- Added `tzdata` to the build stage so the timezone database is available
  in the final image.
- Added `COPY --from=build /etc/group /etc/group` for complete user/group info.
- Unified `Containerfile` (Podman / Buildah) with identical logic and
  Podman-specific comments (`:Z` SELinux volume label, rootless usage).
- Set explicit `CMD ["serve"]` alongside `ENTRYPOINT` for clarity.

### Kubernetes
- Updated `deploy/k8s/04-autoscaling.yaml` with:
  - `autoscaling/v2` HPA (replaces deprecated `v2beta2`).
  - `PodDisruptionBudget` (`minAvailable: 1`) to prevent all-pods eviction.
  - `NetworkPolicy` restricting ingress to declared ports and egress to
    DNS + cluster peers + HTTPS (for AI providers / webhooks).

**Files changed:** `docker-compose.yml`, `Dockerfile`, `Containerfile`,
`deploy/k8s/04-autoscaling.yaml`

---

## Patch 7 — Build System Modernisation

### Makefile
- Added `build-go` target (backend-only, skips UI rebuild).
- Added `podman-build`, `podman-run` targets.
- Added `k8s-apply`, `k8s-status`, `helm-install` targets.
- Added `release` target (cross-compile for linux/amd64, linux/arm64,
  darwin/arm64, windows/amd64).
- Fixed `go` target to use `CGO_ENABLED=0` for reproducible static builds.
- Added `i18n` sync target and `dev` convenience target.

### justfile
- Added `podman-build`, `podman-run`, `k8s-apply`, `k8s-delete`,
  `k8s-status`, `helm-install`, `release` (Windows binary added),
  `sources` (log-sources CLI), `i18n` sync.
- Fixed `release` to include `windows/amd64.exe`.

### Charlotfile (new)
- Added `Charlotfile` — Cocotte-compatible project automation file covering
  all build, test, Docker, Podman, Kubernetes, release, and maintenance tasks.

**Files changed:** `Makefile`, `justfile`, `Charlotfile` (new)

---

## Patch 8 — i18n Completeness

Added keys for the new Log Sources page across all three language files
(`en`, `fr`, `es`) in both `internal/i18n/` and `internal/api/i18ndata/`:

| Key | EN |
|-----|----|
| `nav_sources` | Log Sources |
| `sources_desc` | Manage and monitor active log ingestion endpoints |
| `title_source_rules` | Source Rules |
| `btn_add_source` | + Add Source |
| `title_add_source` | Add Log Source Rule |
| `lbl_source_pattern` | Pattern (glob or exact name) |
| `lbl_source_desc` | Description |
| `lbl_retention_days` | Retention (days, 0 = global default) |
| `lbl_enabled` | Enabled |

**Files changed:** `internal/i18n/{en,fr,es}.json`,
`internal/api/i18ndata/{en,fr,es}.json`

---

## Patch 9 — Distributed Search, Security Hardening & CLI Parity

### Symptom
- Search only queried the local node, ignoring cluster peers.
- Session cookies lacked `Secure` flag; common security headers were missing.
- CLI lacked `sources` command and language support for AI triage.
- Keybinds modal in React UI opened with `?` instead of `k`.

### Fix
- **Distributed Search**: `internal/api/server.go` now fan-outs log searches to all healthy nodes in the cluster using `local_only=true` to prevent recursion.
- **Security**: Added `securityHeaders` middleware (CSP, HSTS, XSS, etc.) and set `Secure: true` for the session cookie.
- **CLI Parity**: Added `logdock sources` command and `--lang` flag to `logdock triage`.
- **UI Keybinds**: Fixed `App.tsx` to open the shortcuts modal on the `k` press, matching the legacy `index.html` behavior.

**Files changed:** `internal/api/server.go`, `cmd/logdock/main.go`, `web/src/App.tsx`


---

## Patch 4 — Production Hardening & Feature Completion (2026-04-04)

### Issues fixed

**1. Keyboard shortcuts menu (BUG-011)**
- Modal was triggered by `k` but displayed `?` as the trigger key — corrected everywhere.
- Static shortcut list replaced with dynamic 1–9 panel labels derived from the live `NAV[]` array at runtime.
- Settings > Keybinds tab: `Shift + /` → `k`.

**2. i18n / translations (BUG-012)**
- `btn_translate` key in the English locale contained a French string (`"Traduire"`) — fixed to `"Translate"`.
- 90+ missing keys added to `en.json`, `fr.json`, `es.json` (shortcuts, LogDockAI section, all settings panel labels, link labels, UI primitives).
- All four i18n file copies synced (`internal/api/i18ndata/`, `internal/i18n/`, `internal/api/webdist/`).

**3. Log sources — missing collectors (BUG-013)**
- New `internal/ingest/sources.go`:
  - `StartJournaldCollector`: streams from `/run/systemd/journal/stdout` (unix socket) with exponential-backoff reconnect.
  - `StartKmsgCollector`: tails `/dev/kmsg`, parses syslog priority bits → LogDock level strings.
  - `StartFileTailCollector`: line-by-line tail with rotation detection (size shrink → seek 0).
  - `WellKnownSources()`: auto-discovers `/var/log/syslog`, auth, nginx, postgres, redis, etc.
  - All collectors degrade gracefully on permission denied / socket absent (container-safe).
- `cmd/logdock/main.go` wires all collectors at startup. Opt-out via `LOGDOCK_DISABLE_FILE_TAIL=1`.

**4. Settings persistence (BUG-014)**
- `internal/settings/settings.go` fully rewritten:
  - `LoadFrom(path)`: reads persisted JSON, merges over defaults (forward-compatible upgrades).
  - `Save()`: atomic write (write tmp → rename — no corrupt state on crash).
  - `Validate(snap)`: 15 sanity checks (ranges, path traversal, valid enums).
  - `UpdateAndSave(snap)`: validate + persist in one call.
- `handleSettings` and `handleAISettings` in `server.go` now call `UpdateAndSave` — all changes survive restarts.
- `main.go` calls `LoadFrom` on startup; path configurable via `LOGDOCK_SETTINGS_PATH`.

**5. Theme, font, landing panel (BUG-015)**
- `uiTheme` and `landingPanel` state loaded from persisted settings and applied at runtime in `AppInner`.
- `themeVars` spread onto root div — light and high-contrast themes now visually apply immediately on load.
- `landingApplied` ref prevents re-navigation on every refresh cycle.

**6. ExternalLink component (BUG-016)**
- Accessible `<a>` wrapper: `rel="noopener noreferrer"`, `target="_blank"`, `aria-label`, hover state, external-link icon.
- Used consistently across CLI Reference cards and LogDockAI panel.

**7. LogDockAI placeholder (BUG-017)**
- New `logdock-ai` tab added to `AIPanel` TabBar.
- Hero banner, 6-feature grid, developer integration note, links to repo/roadmap/changelog.
- i18n keys added for all strings in all three locales.

**8. `Server.ServeHTTP` (BUG-018)**
- `api.Server` now implements `http.Handler` via a `ServeHTTP` method — required for `httptest` in integration tests.

### Tests added
- `internal/settings/settings_test.go` — 14 unit tests (validate, round-trip, atomic write, defaults merge, log source rules).
- `internal/ingest/sources_test.go` — 12 tests (kmsg level/message parsing, file-tail, journald mock socket, WellKnownSources).
- `internal/api/integration_test.go` — 12 integration tests (settings GET/POST/validation, AI settings, i18n EN/FR/ES/fallback, ingest, log sources, auth guard).

**Files changed:** `web/src/App.tsx`, `web/i18n.js`, `internal/api/i18ndata/{en,fr,es}.json`, `internal/i18n/{en,fr,es}.json`, `internal/api/webdist/i18n.js`, `internal/settings/settings.go`, `internal/settings/settings_test.go`, `internal/ingest/sources.go`, `internal/ingest/sources_test.go`, `internal/api/server.go`, `internal/api/integration_test.go`, `cmd/logdock/main.go`

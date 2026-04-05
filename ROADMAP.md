# LogDock Roadmap

This document tracks planned features, improvements, and major milestones.
Items marked **COMING SOON** are under active development or design.
Items marked **PLANNED** are scheduled for future releases.

---

## v2.2 — Cluster & Aggregation Improvements *(near-term)*

- [ ] Web UI for real-time multi-node aggregation view
- [ ] Node health heatmap on dashboard
- [ ] Replication lag metrics per peer
- [ ] Dynamic node join/leave without restart
- [ ] Cluster-wide search fan-out with result deduplication

---

## v2.3 — Pipeline & Enrichment Engine *(near-term)*

- [ ] Drag-and-drop pipeline rule builder in WebUI
- [ ] Built-in GeoIP enrichment for remote IPs
- [ ] Field masking / redaction rules (PII scrubbing)
- [ ] Tag-based routing to multiple storage backends
- [ ] CEF and LEEF log format parsers

---

## v3.0 — SaaS & Enterprise Hardening *(mid-term)*

- [ ] Multi-tenant namespace isolation
- [ ] Role-based access control (RBAC) with fine-grained permissions
- [ ] OpenID Connect (OIDC) / SAML 2.0 SSO integration
- [ ] Audit log export to S3 / GCS / Azure Blob
- [ ] Automatic TLS certificate provisioning (ACME / Let's Encrypt)
- [ ] Rate limiting per API key / per source
- [ ] Signed binary releases with checksums + SBOM
- [ ] Helm chart published to ArtifactHub

---

## LogDock AI — **COMING SOON**

> **LogDock AI** is a specialized AI assistant purpose-built for log
> observability, incident response, and SRE workflows.

Unlike generic AI integrations, LogDock AI is trained and fine-tuned on:

- Common application, infrastructure, and security log patterns
- Incident response runbooks and post-mortem templates
- Compliance frameworks (DORA, SOC 2, GDPR, HIPAA, PCI-DSS)
- Multi-service correlation and distributed tracing context

### Planned Capabilities

| Feature | Description |
|---------|-------------|
| **Conversational log query** | Ask questions in plain language: *"Show me all 5xx errors from the payment service in the last 2 hours"* |
| **Incident auto-triage** | Automatically group related errors, identify root cause candidates, and suggest remediation steps |
| **Anomaly explanation** | Describe why a source was flagged as anomalous with supporting evidence |
| **Compliance co-pilot** | Generate audit narratives and evidence packs for SOC 2 / GDPR reviews |
| **Runbook generation** | Produce step-by-step incident runbooks from historical log patterns |
| **SRE assistant** | Recommend alert thresholds, retention policies, and pipeline rules based on your traffic profile |
| **Multi-node correlation** | Correlate events across all cluster nodes to identify cascading failures |
| **Natural language alerts** | Define alert conditions in plain English; LogDock AI translates to filter rules |

### Privacy & Deployment

- **On-premise model** (Ollama-compatible) — logs never leave your network
- **Bring-your-own API key** — connect OpenAI, Anthropic, or any
  OpenAI-compatible endpoint
- **Selective context** — choose exactly which log data the AI can access;
  sensitive fields can be redacted before AI processing

> Sign up for early access at the LogDock project page.

---

## Long-term Backlog

- OpenTelemetry Collector plugin (replace Fluent/Fluentd)
- Grafana data source plugin
- PromQL-compatible query endpoint
- Time-series downsampling for long-term storage
- Mobile companion app (iOS / Android)
- Dark-mode command palette (Cmd+K full command search)

---

*Last updated: 2026-03-26*
